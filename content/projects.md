+++
title = "OSS Projects"
date = 2020-03-22T02:13:50Z
author = "Nitish Malhotra"
slug = "projects"
tags = ["github","open-source"]
+++
---

### [Go-ReJSON](https://github.com/nitishm/go-rejson)

Go-ReJSON is a Go client for ReJSON Redis Module.

> ReJSON is a Redis module that implements ECMA-404 The JSON Data Interchange Standard as a native data type. It allows storing, updating and fetching JSON values from Redis keys (documents).

Primary features of ReJSON Module:

* Full support of the JSON standard
* JSONPath-like syntax for selecting element inside documents
* Documents are stored as binary data in a tree structure, allowing fast access to sub-elements
* Typed atomic operations for all JSON values types

> Each and every feature of ReJSON Module is fully incorporated in the project.

Enjoy ReJSON with the type-safe Redis client, Go-Redis/Redis or use the print-like Redis-api client GoModule/Redigo. Go-ReJSON supports both the clients. Use any of the above two client you want, Go-ReJSON helps you out with all its features and functionalities in a more generic and standard way.

Support for mediocregopher/radix and other Redis clients is in our RoadMap. Any contributions on the support for other clients is hearty welcome.

---

### [Vegeta-Server](https://github.com/nitishm/vegeta-server)

A RESTful API server for [vegeta](https://github.com/tsenart/vegeta), a load testing tool written in Go.

Vegeta is a versatile HTTP load testing tool built out of a need to drill HTTP services with a constant request rate. The vegeta library is written in Go, which makes it ideal to implement server in Go.

The REST API model enables users to, asynchronously, submit multiple attacks targeting the same (or varied) endpoints, lookup pending or historical reports and update/cancel/re-submit attacks, using a simple RESTful API.

---

### [Engarde](https://github.com/nitishm/engarde)

Parse default [envoy access logs](https://www.envoyproxy.io/docs/envoy/v1.8.0/configuration/access_log#default-format) like a champ with engarde and [jq](https://github.com/stedolan/jq)

---

### [Logger](https://github.com/nitishm/logger)

A wrapper around [logrus](https://github.com/sirupsen/logrus) that allows setting default fields, and supports adding and removing configurable fields.
