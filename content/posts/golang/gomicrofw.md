+++
title = "Building a microservice framework in Golang"
description = "An opinionated runtime framework for Go microservices"
tags = [
    "golang",
    "micro-services",
]
categories = [
    "project"
]
date = 2018-05-12
author = "Nitish Malhotra"
+++

![DancingGopher](https://camo.githubusercontent.com/c70f18274a81ee98dca1c116b68d5a35847b2e65/687474703a2f2f7374617469632e76656c76657463616368652e6f72672f70616765732f323031382f30362f31332f70617274792d676f706865722f64616e63696e672d676f706865722e676966)

## Introduction

Exactly one year ago I started my job at [CSP Inc.](https://www.cspi.com/) to work on the [ARIA](https://www.cspi.com/aria-software-defined-security/) product line. At the time we were a team of four engineers, all hailing from an embedded background. Having worked extensively in C/C++ in the past, we had little to no experience in Golang (heck I hadn’t even heard of Golang at that point). We were faced with a daunting task — to build the next biggest Software Defined Security product! The project had multiple components ranging from the web-service front-end to developing an agent software which would run on our Secure Intelligent Adapters (SIA).

I was tasked with designing and developing the Controlplane (which we refer to as the Software Defined Security Orchestrator — SDSo). The SDSo was to be developed as a collection of multiple microservices mostly written in Golang. Over the past year I have worked with multiple open-source libraries and implemented services using Kafka, gRPC, Redis, MySQL, HTTP, Celery, among several others. In this time, I have created over twenty microservices written exclusively in Golang. With the team growing from four to fifteen engineers working on the product, I have seen microservices being developed in a style as unique as the developer who worked on it. However, in principle, each one of these services are the same. Each one composed of one or more running servers, storage options and one or more upstream clients.

Due to the fast paced nature of our work, it was absolutely essential for me and my team of four engineers to focus on the business logic of our services. Rather than rehashing boilerplate code to setup our microservices such as initializing our server, database, clients etc., I came up with a simplified framework which I will be describing in this post.

---

## Microservice interface

Each microservice should satisfy the Microservice interface. This is not imposed but only serves as a guideline.

```go
type Microservice interface {
    // RegisterServer registers a server with a ServerType identifier. The server must implement the ServerInterface
    RegisterServer(serverType server.ServerType, server server.ServerInterface) (err error)
    // Start is used to start the microservice. All servers registered with the microservice are started when this
    // method is invoked.
    Start() (err error)
    // Stop is used to gracefully stop the microservice. All servers registered with the microservice are gracefully stopped
    // when this method is invoked.
    Stop() (err error)
}
```

An example of how we implement this interface would be :

```go
type ServiceMgr struct {
    // A map of all the servers registered with the service
    servers map[server.ServerType]server.ServerInterface
}
func (ss *ServiceMgr) RegisterServer(serverType server.ServerType, server server.ServerInterface) (err error) {
    // Store the server in a map
    ss.servers[serverType] = server
    return
}
func (ss *ServiceMgr) Start() (err error) {
    // Do all your service initialization here.
    err = ss.init()
    if err != nil {
        return
    }
    // Invoke the start method on each of the servers
    for t, s := range ss.servers {
        err = s.Start()
        if err != nil {
            return err
        }
    }
    return
}
func (ss *ServiceMgr) Stop() (err error) {
    // Issue a graceful shutdown call to each of the servers
    for t, s := range ss.servers {
        err = s.Stop()
        if err != nil {
            return err
        }
    }
    return
}
```

We will get to the `init()` seen in *line 12* above in a few moment. (*)

## Server interface

Each server would need to implement the server interface which is composed of the following :

A `Start()` method which would start the would start the server as a go routine.

A `Stop()` method which would gracefully stop the server by sending a message on its quit channel returned at the time of starting

A `RegisterNamespace()` and `RegisterService()` to register a namespace which allows for using the same channel/topic for sending messages destined for different endpoints identified by their service type string.

```go
// ServerType is a typedef for server identifier
type ServerType string

// Service is a typedef for service identifier
type Service string

// ServiceEndpointMap is a typedef for a map of Service endpoints identified by their service type
type ServiceEndpointMap map[Service]endpoint.Endpoint

// Handler is a func signature implemented by the Endpoint Handle() method
type Handler func(in []byte, serviceEndpointMap ServiceEndpointMap) (err error)

// StartStopInterface is composed of the Start() and Stop() methods to be implemented by the server
type StartStopInterface interface {
    // Start is used to start the server
    Start() (err error)
    // Stop is used to gracefully stop the server
    Stop() (err error)
}

// ServerInterface defines the methods that all servers registed with the microservice must implement
type ServerInterface interface {
    StartStopInterface
    // RegisterNamesapce is used to register a namespace (like a kafka channel/topic or grpc namespace)
    // with the server.
    RegisterNamespace(namespace string)
    // RegisterService is used to register a service and its endpoints with the server.
    RegisterService(namespace string, service Service, endpoint endpoint.Endpoint)
}
```

An implementation of the server would look something like follows -

Dont get confused by the nomenclature [`KafkaConsumer`] below. At CSPi we use `kafka` not just as a `pubsub` framework but also as a server for our microservices. Typically used when the service being targeted needs to *asychronous*. (You may note that the response in the `Handle()` is not returned to the caller. Sending a response is an implementation detail when using kafka as a server).

```go
// KafkaConsumer encapsulates all members used for the kafka server.
type KafkaConsumer struct {
    cfg    *configuration.KafkaConfig
    config *sarama.Config
    master sarama.Consumer
    topics map[string]Topic
    quit   chan bool
}

// Topic encapsulates a kafka channel name and a map of endpoints used on that channel
type Topic struct {
    Name               string
    ServiceEndpointMap ServiceEndpointMap
}

// MyTopic returns a new Topic instance
func MyTopic(name string) (topic Topic) {
    return Topic{
        Name:               name,
        ServiceEndpointMap: make(ServiceEndpointMap),
    }
}

// MyKafkaConsumer takes in configuration object and returns a new KafkaConsumer instance.
func MyKafkaConsumer(cfg *configuration.KafkaConfig) (kc *KafkaConsumer, err error) {
    config := sarama.NewConfig()
    config.Consumer.Return.Errors = true

    master, err := sarama.NewConsumer(cfg.Brokers, config)
    if err != nil {
        return
    }

    kc = &KafkaConsumer{
        cfg,
        config,
        master,
        make(map[string]Topic),
        make(chan bool),
    }

    return
}

// Start is an implementation of the Server Start() method.
func (kc *KafkaConsumer) Start() (err error) {
    for _, topic := range kc.topics {
        go func(topic Topic) {
            consumer(kc.master, kc.quit, topic, ByteHandler, kc.cfg)
            select {
            case <-kc.quit:
                return
            }
        }(topic)
    }
    return
}

// Stop is an implementation of the Server Stop() method
func (kc *KafkaConsumer) Stop() (err error) {
    close(kc.quit)
    kc.master.Close()
    return
}

// RegisterNamespace is an implementation of the Server RegisterNamespace method.
// In kafka a namespace is the same as the channel name.
func (kc *KafkaConsumer) RegisterNamespace(name string) {
    kc.topics[name] = MyTopic(name)
    return
}

// RegisterService is an implementation of the server RegisterService method.
// In kafka this is used to demux the messages coming in on a single channel/topic.
func (kc *KafkaConsumer) RegisterService(namespace string, service Service, ep endpoint.Endpoint) {
    kc.topics[namespace].ServiceEndpointMap[service] = ep
    return
}
```

## Endpoint interface

The business logic of our service is encapsulated in the Endpoint interface within the `Handle()` method.

```go
package endpoint

import "context"

// Endpoint provides the main interface to be implemented by each endpoint.
// Endpoints are where the user provides all the business logic of the microservice.
type Endpoint interface {
    Handle(ctx context.Context, request []byte) (response []byte, err error)
}
```

An implementation of which would look something like this :

```go
package endpoint

import (
    "context"

    log "github.com/sirupsen/logrus"
)

// HelloEndpoint is the endpoint specific object
type HelloEndpoint struct {
}

// MyHelloEndpoint returns an instance of the HelloEndpoint object
func MyHelloEndpoint() *HelloEndpoint {
    return &HelloEndpoint{}
}

// Handle implements the Endpoint Handle method.
// This is where the business logic of the endpoint is implemented.
func (ep *HelloEndpoint) Handle(ctx context.Context, request []byte) (response []byte, err error) {
    log.Infof("\n=======\nHello %s\n=======\n", request)
    return
}
```

The users job is to register this endpoint as a handler for the service identified by a string `"hello"` registered under the `"greeter"` namespace. In `kafka` the namespace is synonymous with the `topic` name.

```go
// Do all other initialization work, specific to the microservice here.
func (ss *ServiceMgr) init() (err error) {
    sv := ss.servers[server.ServerType("kafka")]
    // Register a namespace
    sv.RegisterNamespace("greeter")

    // Register your endpoints against the namespace with the server
    sv.RegisterService("greeter", server.Service("hello"), endpoint.MyHelloEndpoint())
    return
}
```

That wraps up my post about microservices.

Keep watching this space for more edits, completed with snippets and gifs to come.

---

## Doing this with an RPC Server instead of Kafka

The Server interface implementation would like below :

```go
package server

import (
    "encoding/json"
    "go-micro-framework/microservice/configuration"
    "go-micro-framework/microservice/endpoint"
    "log"
    "net"

    context "golang.org/x/net/context"

    grpc "google.golang.org/grpc"
)

type RPCNamespace struct {
    Name               string
    ServiceEndpointMap ServiceEndpointMap
}

func MyRPCNamespace(name string) (namespace RPCNamespace) {
    return RPCNamespace{
        Name:               name,
        ServiceEndpointMap: make(ServiceEndpointMap),
    }
}

type RPCServer struct {
    addr       string
    namespaces map[string]RPCNamespace
    server     *grpc.Server
}

func MyRPCServer(cfg *configuration.RPCConfig) (rpc *RPCServer, err error) {
    rpc = &RPCServer{
        addr:       cfg.Address,
        namespaces: make(map[string]RPCNamespace),
    }
    return
}

func (rpc *RPCServer) Start() (err error) {
    lis, err := net.Listen("tcp", rpc.addr)
    if err != nil {
        log.Fatalf("failed to listen: %v", err)
    }

    rpc.server = grpc.NewServer()
    RegisterMessageServiceServer(rpc.server, rpc)
    go rpc.server.Serve(lis)

    return
}

func (rpc *RPCServer) Stop() (err error) {
    rpc.server.GracefulStop()
    return
}

func (rpc *RPCServer) RegisterNamespace(name string) {
    rpc.namespaces[name] = MyRPCNamespace(name)
    return
}

func (rpc *RPCServer) RegisterService(namespace string, service Service, ep endpoint.Endpoint) {
    rpc.namespaces[namespace].ServiceEndpointMap[service] = ep
    return
}

func (rpc *RPCServer) Command(c context.Context, msg *RPCMessage) (resp *RPCResponse, err error) {
    out, err := ByteHandler(msg.Data, rpc.namespaces[msg.Namespace].ServiceEndpointMap)
    if err != nil {
        resp = &RPCResponse{
            Error: err.Error(),
        }
        return
    }
    resp = &RPCResponse{
        Data: out,
    }
    return
}

type RPCClient struct {
    addr   string
    client MessageServiceClient
}

func MyRPCClient(cfg *configuration.RPCConfig) (rpc *RPCClient, err error) {
    conn, err := grpc.Dial(cfg.Address, grpc.WithInsecure())
    if err != nil {
        return
    }
    rpc = &RPCClient{
        addr:   cfg.Address,
        client: NewMessageServiceClient(conn),
    }
    return
}

func (rpc *RPCClient) Send(namespace string, msg interface{}) (resp string, err error) {
    b, err := json.Marshal(msg)
    if err != nil {
        return
    }
    respObj, err := rpc.client.Command(
        context.Background(),
        &RPCMessage{
            Namespace: namespace,
            Data:      b,
        },
    )
    if err != nil {
        return
    }

    resp = string(respObj.Data)
    return
}
```

I generated the RPC server using the `protoc` tool using the following `protobuf` specification file.

```go
syntax = "proto3";

package rpc;

service MessageService {
    rpc Command(RPCMessage) returns (RPCResponse) {};
}

message RPCMessage {
    string Namespace = 1;
    bytes Data = 2;
}

message RPCResponse {
    bytes Data = 1;
    string Error = 2;
}
```

, which can be used to generate the proto server using the protoc tool provided by `gRPC`.

```go
package server

// This is created using the rpc.proto file.

import (
    fmt "fmt"
    "math"

    "github.com/golang/protobuf/proto"
    context "golang.org/x/net/context"
    grpc "google.golang.org/grpc"
)

// Reference imports to suppress errors if they are not otherwise used.
var _ = proto.Marshal
var _ = fmt.Errorf
var _ = math.Inf

// This is a compile-time assertion to ensure that this generated file
// is compatible with the proto package it is being compiled against.
// A compilation error at this line likely means your copy of the
// proto package needs to be updated.
const _ = proto.ProtoPackageIsVersion2 // please upgrade the proto package

type RPCMessage struct {
    Namespace            string   `protobuf:"bytes,1,opt,name=Namespace" json:"Namespace,omitempty"`
    Data                 []byte   `protobuf:"bytes,2,opt,name=Data,proto3" json:"Data,omitempty"`
    XXX_NoUnkeyedLiteral struct{} `json:"-"`
    XXX_unrecognized     []byte   `json:"-"`
    XXX_sizecache        int32    `json:"-"`
}

func (m *RPCMessage) Reset()         { *m = RPCMessage{} }
func (m *RPCMessage) String() string { return proto.CompactTextString(m) }
func (*RPCMessage) ProtoMessage()    {}
func (*RPCMessage) Descriptor() ([]byte, []int) {
    return fileDescriptor_rpc_848df06cb7d89181, []int{0}
}
func (m *RPCMessage) XXX_Unmarshal(b []byte) error {
    return xxx_messageInfo_RPCMessage.Unmarshal(m, b)
}
func (m *RPCMessage) XXX_Marshal(b []byte, deterministic bool) ([]byte, error) {
    return xxx_messageInfo_RPCMessage.Marshal(b, m, deterministic)
}
func (dst *RPCMessage) XXX_Merge(src proto.Message) {
    xxx_messageInfo_RPCMessage.Merge(dst, src)
}
func (m *RPCMessage) XXX_Size() int {
    return xxx_messageInfo_RPCMessage.Size(m)
}
func (m *RPCMessage) XXX_DiscardUnknown() {
    xxx_messageInfo_RPCMessage.DiscardUnknown(m)
}

var xxx_messageInfo_RPCMessage proto.InternalMessageInfo

func (m *RPCMessage) GetNamespace() string {
    if m != nil {
        return m.Namespace
    }
    return ""
}

func (m *RPCMessage) GetData() []byte {
    if m != nil {
        return m.Data
    }
    return nil
}

type RPCResponse struct {
    Data                 []byte   `protobuf:"bytes,1,opt,name=Data,proto3" json:"Data,omitempty"`
    Error                string   `protobuf:"bytes,2,opt,name=Error" json:"Error,omitempty"`
    XXX_NoUnkeyedLiteral struct{} `json:"-"`
    XXX_unrecognized     []byte   `json:"-"`
    XXX_sizecache        int32    `json:"-"`
}

func (m *RPCResponse) Reset()         { *m = RPCResponse{} }
func (m *RPCResponse) String() string { return proto.CompactTextString(m) }
func (*RPCResponse) ProtoMessage()    {}
func (*RPCResponse) Descriptor() ([]byte, []int) {
    return fileDescriptor_rpc_848df06cb7d89181, []int{1}
}
func (m *RPCResponse) XXX_Unmarshal(b []byte) error {
    return xxx_messageInfo_RPCResponse.Unmarshal(m, b)
}
func (m *RPCResponse) XXX_Marshal(b []byte, deterministic bool) ([]byte, error) {
    return xxx_messageInfo_RPCResponse.Marshal(b, m, deterministic)
}
func (dst *RPCResponse) XXX_Merge(src proto.Message) {
    xxx_messageInfo_RPCResponse.Merge(dst, src)
}
func (m *RPCResponse) XXX_Size() int {
    return xxx_messageInfo_RPCResponse.Size(m)
}
func (m *RPCResponse) XXX_DiscardUnknown() {
    xxx_messageInfo_RPCResponse.DiscardUnknown(m)
}

var xxx_messageInfo_RPCResponse proto.InternalMessageInfo

func (m *RPCResponse) GetData() []byte {
    if m != nil {
        return m.Data
    }
    return nil
}

func (m *RPCResponse) GetError() string {
    if m != nil {
        return m.Error
    }
    return ""
}

func init() {
    proto.RegisterType((*RPCMessage)(nil), "rpc.RPCMessage")
    proto.RegisterType((*RPCResponse)(nil), "rpc.RPCResponse")
}

// Reference imports to suppress errors if they are not otherwise used.
var _ context.Context
var _ grpc.ClientConn

// This is a compile-time assertion to ensure that this generated file
// is compatible with the grpc package it is being compiled against.
const _ = grpc.SupportPackageIsVersion4

// Client API for MessageService service

type MessageServiceClient interface {
    Command(ctx context.Context, in *RPCMessage, opts ...grpc.CallOption) (*RPCResponse, error)
}

type messageServiceClient struct {
    cc *grpc.ClientConn
}

func NewMessageServiceClient(cc *grpc.ClientConn) MessageServiceClient {
    return &messageServiceClient{cc}
}

func (c *messageServiceClient) Command(ctx context.Context, in *RPCMessage, opts ...grpc.CallOption) (*RPCResponse, error) {
    out := new(RPCResponse)
    err := grpc.Invoke(ctx, "/rpc.MessageService/Command", in, out, c.cc, opts...)
    if err != nil {
        return nil, err
    }
    return out, nil
}

// Server API for MessageService service

type MessageServiceServer interface {
    Command(context.Context, *RPCMessage) (*RPCResponse, error)
}

func RegisterMessageServiceServer(s *grpc.Server, srv MessageServiceServer) {
    s.RegisterService(&_MessageService_serviceDesc, srv)
}

func _MessageService_Command_Handler(srv interface{}, ctx context.Context, dec func(interface{}) error, interceptor grpc.UnaryServerInterceptor) (interface{}, error) {
    in := new(RPCMessage)
    if err := dec(in); err != nil {
        return nil, err
    }
    if interceptor == nil {
        return srv.(MessageServiceServer).Command(ctx, in)
    }
    info := &grpc.UnaryServerInfo{
        Server:     srv,
        FullMethod: "/rpc.MessageService/Command",
    }
    handler := func(ctx context.Context, req interface{}) (interface{}, error) {
        return srv.(MessageServiceServer).Command(ctx, req.(*RPCMessage))
    }
    return interceptor(ctx, in, info, handler)
}

var _MessageService_serviceDesc = grpc.ServiceDesc{
    ServiceName: "rpc.MessageService",
    HandlerType: (*MessageServiceServer)(nil),
    Methods: []grpc.MethodDesc{
        {
            MethodName: "Command",
            Handler:    _MessageService_Command_Handler,
        },
    },
    Streams:  []grpc.StreamDesc{},
    Metadata: "rpc.proto",
}

func init() { proto.RegisterFile("rpc.proto", fileDescriptor_rpc_848df06cb7d89181) }

var fileDescriptor_rpc_848df06cb7d89181 = []byte{
    // 171 bytes of a gzipped FileDescriptorProto
    0x1f, 0x8b, 0x08, 0x00, 0x00, 0x00, 0x00, 0x00, 0x02, 0xff, 0xe2, 0xe2, 0x2c, 0x2a, 0x48, 0xd6,
    0x2b, 0x28, 0xca, 0x2f, 0xc9, 0x17, 0x62, 0x2e, 0x2a, 0x48, 0x56, 0xb2, 0xe3, 0xe2, 0x0a, 0x0a,
    0x70, 0xf6, 0x4d, 0x2d, 0x2e, 0x4e, 0x4c, 0x4f, 0x15, 0x92, 0xe1, 0xe2, 0xf4, 0x4b, 0xcc, 0x4d,
    0x2d, 0x2e, 0x48, 0x4c, 0x4e, 0x95, 0x60, 0x54, 0x60, 0xd4, 0xe0, 0x0c, 0x42, 0x08, 0x08, 0x09,
    0x71, 0xb1, 0xb8, 0x24, 0x96, 0x24, 0x4a, 0x30, 0x29, 0x30, 0x6a, 0xf0, 0x04, 0x81, 0xd9, 0x4a,
    0xe6, 0x5c, 0xdc, 0x41, 0x01, 0xce, 0x41, 0xa9, 0xc5, 0x05, 0xf9, 0x79, 0xc5, 0x08, 0x25, 0x8c,
    0x08, 0x25, 0x42, 0x22, 0x5c, 0xac, 0xae, 0x45, 0x45, 0xf9, 0x45, 0x60, 0x7d, 0x9c, 0x41, 0x10,
    0x8e, 0x91, 0x03, 0x17, 0x1f, 0xd4, 0xd6, 0xe0, 0xd4, 0xa2, 0xb2, 0xcc, 0xe4, 0x54, 0x21, 0x3d,
    0x2e, 0x76, 0xe7, 0xfc, 0xdc, 0xdc, 0xc4, 0xbc, 0x14, 0x21, 0x7e, 0x3d, 0x90, 0x33, 0x11, 0x0e,
    0x93, 0x12, 0x80, 0x09, 0xc0, 0x6c, 0x52, 0x62, 0x48, 0x62, 0x03, 0x7b, 0xc3, 0x18, 0x10, 0x00,
    0x00, 0xff, 0xff, 0x62, 0x4f, 0xa7, 0xc1, 0xd3, 0x00, 0x00, 0x00,
}
```

You can find all the code hosted on [github](https://github.com/nitishm/go-micro-framework/).

To test this framework out I have provided an example directory completed with a simple microservice running `kafka` as a server which can found in the `/examples/` folder. Simple start the server by running the main.go in the `/examples/kafka-microservice/` directory.

To send a message over the listening kafka topic, use the `producer.go` program provided under the `/examples` directory.

[A `docker-compose` file is in the works for using the example microservice. I will update the page as soon as I have that ready. If you are curious you can always manually startup your kafka server using the `Dockerfile` provided at https://hub.docker.com/r/spotify/kafka/].

---

*If you looking for a more mature and independent framework, take a look at gizmo, go-kit and go-micro. The reason we didn’t use any of these framework at our workplace was because we found these frameworks to be more web-dev-centric and not as flexible for our diverse needs.*

---

*Originally published on [medium](https://medium.com/codezillas/building-a-microservice-framework-in-golang-dd3c9530dff9)*
