+++
title = "Storing Go Structs in Redis using ReJSON"
description = "Enhancing redis experience for JSON data"
tags = [
    "golang",
    "redis",
    "json"
]
categories = [
    "library"
]
date = 2018-05-16
author = "Nitish Malhotra"
+++

![golang-redis](https://redislabs.com/wp-content/uploads/2018/03/golang-redis.jpg)

*Image courtesy : https://redislabs.com/blog/redis-go-designed-improve-performance/*

A lot of you might be familiar with [Redis](https://redis.io/). For the *uninitiated*, redis is one of, if not the most, popular and widely adopted database/cache.

The official documentation describes redis as,

> Redis is an open source (BSD licensed), in-memory data structure store, used as a database, cache and message broker. It supports data structures such as strings, hashes, lists, sets, sorted sets with range queries, bitmaps, hyperloglogs and geospatial indexes with radius queries. Redis has built-in replication, Lua scripting, LRU eviction, transactions and different levels of on-disk persistence, and provides high availability via Redis Sentinel and automatic partitioning with Redis Cluster.

What sets redis apart, from other (traditional) databases, is that it is a [key-value](https://en.wikipedia.org/wiki/Key-value_database) store (add to that, it is in-memory). What this means is that in such a database, all values are stored against a single key (*think python dictionaries*).

However, I digress, this post is not about redis, so let’s move along...

---

## Using Go to interact with Redis

As a Go developer using redis, there comes a time, when we need to cache our objects in redis. Let’s see how we can do this using the `HMSET` in Redis.

A simple go structure would look like,

```go
type SimpleObject struct {
    FieldA string
    FieldB int
}

simpleObject := SimpleObject{“John Doe”,24}
```

It is evident that, to store the object in redis, we would need to transform it into a key value pair. We this by using the Go Struct field name as the key and the struct value to be stored against it.

A **hash** would be a perfect candidate, to tie all the fields that belong to an object, back to the object itself. To this from the `redis-cli` we would do the following,

```console
127.0.0.1:6379> HMSET simple_object fieldA “John Doe” fieldB 24
OK
```

The result, fetched using the `HGETALL` command, would be,

```console
127.0.0.1:6379> HGETALL simple_object
fieldA
John Doe
fieldB
24
```

Alright, so now we know how the object gets marshaled into the database. Let’s proceed to doing this programmatically!

Although there a quite a few go clients for redis out there I consider working with [Redigo](https://github.com/gomodule/redigo). It has a great community on [github](https://github.com/gomodule/redigo), and is one of the most popular go-clients for redis, with over 4K stars.

### Redigo helpers — AddFlat and ScanStruct

Redigo comes equipped with a great set of helper functions, one of which we will use `AddFlat`, to flatten our structure, before adding it into redis.

```go
// Get the connection object
conn, err := redis.Dial(“tcp”, “localhost:6379”)
if err != nil {
    return
}
// Invoke the command using the Do command
_, err = conn.Do(“HMSET”, redis.Args{“simple_object”}.AddFlat(simpleObject)…)
if err != nil {
    return
}
```

Now if you wish to read this object back into your object, we can do this with the `HGETALL` command,

```go
value, err := redis.Values(conn.Do(“HGETALL”, key))
if err != nil {
    return
}
object := SimpleStruct{}
err = redis.ScanStruct(value, &object)
if err != nil {
    return
}
```

> Easy enough right ? Let’s see something more involved …

---

### Embedded Objects in Go Structs

Now let’s take a more **complex** structure,

```go
type Student struct {
    Info *StudentDetails `json:”info,omitempty”`
    Rank int             `json:”rank,omitempty”`
}
type StudentDetails struct {
    FirstName string
    LastName  string
    Major     string
}
studentJD := Student{
    Info: &StudentDetails{
        FirstName: “John”,
        LastName:  “Doe”,
        Major:     “CSE”,
    },
    Rank: 1,
}
```

What we have in our hands now is an **embedded** struct, with `StudentDetails`, as a member of the `Student` object.

Let’s try using `HMSET` again,

```console
// Invoke the command using the Do command
_, err = conn.Do(“HMSET”, redis.Args{“JohnDoe”}.AddFlat(studentJD)…)
if err != nil {
    return
}
```

If we look at redis at this point, we will see the info object to be stored as –

```console
127.0.0.1:6379> HGETALL JohnDoe
Info
&{John Doe CSE}
Rank
1
```

Now this is the problem bit. When we try to retrieve the information back into the object, `**ScanStruct**` fails with the error,

```console
redigo.ScanStruct: cannot assign field Info: cannot convert from Redis bulk string to *main.StudentDetails
```

> EPIC FAIL !

This happened because in redis everything is stored as a **string** [`*bulk string*` for larger objects].

### What Now

A quick search would take you to a couple of solutions. One of those solutions suggest using a Marshaler (`JSON` `marshal`) and others suggest `MessagePack`.

I am going to present the `JSON` based solution below.

```go
b, err := json.Marshal(&studentJD)
if err != nil {
    return
}

_, err = conn.Do(“SET”, “JohnDoe”, string(b))
if err != nil {
    return
}
```

To retrieve it just read the `JSON` string back using the `GET` command.

```go
objStr, err = redis.String(conn.Do(“GET”, “JohnDoe”))
if err != nil {
    return
}

b := []byte(objStr)student := &Student{}err = json.Unmarshal(b, student)
if err != nil {
    return
}
```

Well this works great, if all we wish to do is *cache the object in its entirety*.

What if we wanted to just add, modify or read one of the fields, for example, if *John Doe* changed his major to *EE* from *CSE* ??

The only way we can do this is by reading the `JSON` string, `Un-marshaling` it into the object, modifying the object and re-writing it into redis. That seems like a lot of work!

> If you are wondering, doing this with the Hash is trivial by using the HGET/HSET commands. If only, that worked — bummer!

![Just cause, no web-post can exist without a meme …](https://miro.medium.com/max/800/1*ed0x40y1Y30I4NB_ahm6WQ.jpeg)
*Just cause, no web-post can exist without a meme …*

### ReJSON

The excellent team at [RedisLabs](https://redislabs.com/) sweat it out and brought us a solution, that lets us treat our objects as traditional `JSON` objects.

Lets jump right into it. I pick this example straight from the [`rejson`](http://rejson.io/) documentation,

```console
127.0.0.1:6379> JSON.SET amoreinterestingexample . ‘[ true, { “answer”: 42 }, null ]’
OK
127.0.0.1:6379> JSON.GET amoreinterestingexample
“[true,{\”answer\”:42},null]”
127.0.0.1:6379> JSON.GET amoreinterestingexample [1].answer
“42”
127.0.0.1:6379> JSON.DEL amoreinterestingexample [-1]
1
127.0.0.1:6379> JSON.GET amoreinterestingexample
“[true,{\”answer\”:42}]”
```

To do this programmatically, we could definitely use `Redigo` in its raw form. [it’s meant to support any commands supported by Redis, using the `conn.Do(…)` method].

However, I took some time to turn all the ReJSON commands, into a Go convenience package, called [`go-rejson`](https://github.com/nitishm/go-rejson).

Going back to our `Student` object, we could programmatically add it into Redis using the following step.

```console
import "github.com/nitishm/go-rejson"

_, err = rejson.JSONSet(conn, “JohnDoeJSON, “.”, studentJD, false, false)
if err != nil {
    return
}
```

A quick inspection in `redis-cli` gives us,

```console
127.0.0.1:6379> JSON.GET JohnDoeJSON
{“info”:{“FirstName”:”John”,”LastName”:”Doe”,”Major”:”CSE”},”rank”:1}
```

If I wish to just read the info field from the redis entry I would perform a `JSON.SET` as follows,

```console
127.0.0.1:6379> JSON.GET JohnDoeJSON .info
{“FirstName”:”John”,”LastName”:”Doe”,”Major”:”CSE”}
```

Similarly with the `rank` field, I could reference the `.rank`,

```console
127.0.0.1:6379> JSON.GET JohnDoeJSON .rank
1
```

To programmatically retrieve the student object, we would use the `JSON.GET` command via the `JSONGet()` method,

```go
v, err := rejson.JSONGet(conn, “JohnDoeJSON, “”)
if err != nil {
    return
}

outStudent := &Student{}
err = json.Unmarshal(outJSON.([]byte), outStudent)
if err != nil {
    return
}
```

To set the `rank` field we could use the `JSON.SET` command on the `.rank` field using the `JSONSet()` method,

```go
_, err = rejson.JSONSet(conn, “JohnDoeJSON, “.info.Major”, “EE”, false, false)
if err != nil {
    return
}
```

Inspecting the entry in `redis-cli`, we get,

```console
127.0.0.1:6379> JSON.GET JohnDoeJSON{“info”:{“FirstName”:”John”,”LastName”:”Doe”,”Major”:”EE”},”rank”:1}
```

---

### Running this example

#### Launch redis with rejson module using Docker

```console
docker run -p 6379:6379 --name redis-rejson redislabs/rejson:latest
```

#### Clone the example from github

```console
# git clone https://github.com/nitishm/rejson-struct.git
# cd rejson-struct
# go run main.go
```

---

To learn more about the `Go-ReJSON` package, visit [github](https://github.com/nitishm/go-rejson).
Read more about `ReJSON` at their official documentation page, [rejson](http://rejson.io/).

---

## Update - Jan 28, 2019

This library now also supports redis client, go-redis (https://github.com/go-redis/redis).

The library now supports two of the most popular Redis clients written in Go:

- https://github.com/gomodule/redigo
- https://github.com/go-redis/redis

The new version v2.0.0 adds a layer of abstraction, that allows us to support more clients in the time to come.

I would also like to give a shout out to [Shivam Rathore](https://github.com/Shivam010) for his excellent contributions in making v2.0.0 as possibility!

---

*Originally published on [medium](https://itnext.io/storing-go-structs-in-redis-using-rejson-dab7f8fc0053)*

{{ template "_internal/disqus.html" . }}
