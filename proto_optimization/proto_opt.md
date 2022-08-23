Did you know that there are several ways to make [Protobuf](https://developers.google.com/protocol-buffers/docs/overview) de/serilization faster in Python ? 

In this blog post I will walk you through a couple of Protobuf de/serilization optimizations that we implemented in
[Jina](https://github.com/jina-ai/jina). I will mainly cover two concepts:

* 1 - Lazy deserialization
* 2 - Partial deserialization

Just so we are on the same page, even though I will present optimization that are implemented in 
[Jina](https://github.com/jina-ai/jina) but my goal will be to guide you through creating a minimal working example of
each (optimizations) in Python. I will link the related Pull Requests when necessary to showcase these optimizations 
being used in practice in Jina.


# 0 - Some context

## What is Protobuf ?

Well if you are reading this blog post you probably want to optimize your Protobuf de/serilization and therefore know a 
already about Protobuf but just so we are on the same page let me briefly explain what is Protobuf according to me. 
You can skip if you don't feel the need.

`Protobuf is a **binary format** that is used to **serialize** structured data` 

**Serialize** mean that you can translate your data (e.g a Python object) into a language agnostic format
(e.g. a binary format or a text file) in order to send it to a remote system and reconstruct later on.

**Binary** means that the data is stored in a binary format. (Well thank you we already know that). It should be opposed 
to the text format. It is just a different way of encoding the data. (Remember when doing `with open(file, 'b') as f:` 
in python ? the `'b'` stand for binary).

For instance the JSON format is another serialization format but not it is not binary based but a text based format. 

Of course Protobuf is more than that it can leverage a lot of optimization (e.g. compression) and it is commonly used with 
gRPC to create blazingly fast and efficient network communication (Jina is using gRPC as a main protocol that's why I am
talking about Protobuf today).


## 1 - Lazy deserialization

The first Protobuf de/serilization optimization that we are using in Jina is 
`Do not to deserilized at all`... :sweat_smile: 

Okay this sound stupid let me rephrase:

`Do not to deserilized when you don't need it`

Let me give you a bit more of context. [Jina](https://github.com/jina-ai/jina) is a Python framework that empower Ml people
by letting them easily create cloud native microservices based cross and multi modal application (e.g. [Neural Search](https://medium.com/jina-ai/what-is-jina-and-neural-search-7a9e166608ab)
, [Image generation pipeline](https://github.com/jina-ai/dalle-flow) ...). Without going into too much details, the data
that flows into the different microservices are [Document](https://docarray.jina.ai/fundamentals/document/) which are 
send over gRPC once they are serialized into Protobuf bytes. the
different microservices are what we called [Executor](https://docs.jina.ai/fundamentals/executor/index.html) they are just abstraction of
Python code that process the data (i.e. Document ). These Executors are tight together in a 
[Flow](https://docs.jina.ai/fundamentals/flow/index.html) which is just a DAG (Directed Acyclic Graph) that represents the sequence of execution of the Executors.
The entry point of our microservices based application is the [Gateway](https://docs.jina.ai/fundamentals/gateway/index.html)
which is another microservices which role is to route requests to the right Executors by following the DAG (Flow).

`The Gateway does not always need to deserialize the data since most of the time it will only forward the data to the right Executor`

And it is the Gateway that we will leverage our first optimization: the `lazy deserialization`.

We are going to **build a small Gateway** that will forward data requests to the right Services.

Okay stop with the talking and let's start coding :smiling_imp:

let's create a workspace folder
```bash
mkdir proto_opt && cd proto_opt
```

and a `proto_opt.proto` file.

First we need a proto that will represent the request that we will send over gRPC.

A **Request** will be composed of a **Header** and some **Data**.



We will use the `proto3` syntax so lets precise it in our `proto_opt.proto` file 
```protobuf
syntax = "proto3";
```

for the ease of simplicity our Data will only be a list of string

```protobuf
message Data {
    repeated string strings=1;
}
```

Our Header will be composed of a `request_id` a `status` and an optional `target`.

```protobuf
message Header {
    string request_id = 1; 
    string status = 2; 
    string target = 3; 
}
```
We are ready to create the proto for the Requests:

```protobuf
message Request {
  Header header = 1;
  Data data = 2;
}
```

Now we need to create the [gRPC service](https://grpc.io/docs/what-is-grpc/core-concepts/#service-definition) for Gateway.
Let's start by doing a naive implementation. It will **always** deserialize the data and forward it to all the Executors (i.e. the data processing microservices).

```protobuf
service Gateway {
  rpc Process(Request) returns (Request) {}
}
```

Note: in practice we would want this service to work in a streaming mode but for the sake of simplicity we will only have one request at a time.
You can take look at [Jina proto](https://github.com/jina-ai/jina/blob/master/jina/proto/jina.proto#L165-L169) to see how we can leverage stream in practice 

Now we just need to generate the python code from the proto file. But first install the grpcio-tools package
```bash
pip install grpcio-tools
```
then
```bash
python -m grpc_tools.protoc -I. --python_out=. --grpc_python_out=. proto_opt.proto
```

a new file `proto_opt.py` will appear !

```
proto_opt
├── proto_opt.proto
├── proto_opt_pb2.py
└── proto_opt_pb2_grpc.py
```

We are done on the gRPC/Protobuf definition lets actually code the Gateway in Python.

The Gateway is very basic. It has a list of Executor address, each time it receives a request it will forward it to all them

We just need to extend the GatewayServicer python class generate by the grpcio-tools package.

Let's createa a `gateway.py` file and work within it for now.

```python
from typing import List
import proto_opt_pb2_grpc

class Gateway(proto_opt_pb2_grpc.GatewayServicer):
    def __init__(self, executors_addr: List[str]):
        self.executors_addr = executors_addr
```

We need then to implement the `Process` method from the service. 
Note: Yes the method name start is an upper case P even if it is against python nomenclature but that's because it is a RPC method.

```python  
    def Process(self, request, context):
        for addr in self.executors_addr:
            self.send_request(addr, request)
        return request

    def send_request(self, addr: str, request):
        print(f"Sending request to {addr}")
```

Okay we are almost there to have our first naive Gateway working ! We just need to add the gRPC server and a client to interact with it.

In the `gateway.py` let's add the gRPC server

```python
```python
import grpc
from concurrent import futures

def serve():
    server = grpc.server(futures.ThreadPoolExecutor(max_workers=10))
    proto_opt_pb2_grpc.add_GatewayServicer_to_server(
        GatewayServicer(executors_addr=['addr1', 'addr2']), server
    )
    server.add_insecure_port('[::]:50051')
    server.start()
    server.wait_for_termination()

if __name__ == '__main__':
    serve()
```

In a new `client.py` let's add the client

First we need to create the Requests
```python
import proto_opt_pb2

def get_requests(id):
    header = proto_opt_pb2.Header(
        request_id=id, status='healthy', target_executor='exec1'
    )
    data = proto_opt_pb2.Data(data='data1')
    return proto_opt_pb2.Request(header=header, data=data)
```

Then we need to add the code to call the gRPC server. We create a stub to connect to the server, and then we call the `Process` method.

```python
if __name__ == '__main__':
    channel = grpc.insecure_channel('localhost:50051')
    stub = proto_opt_pb2_grpc.GatewayStub(channel)
    stub.Process(get_requests('1'))
```

FINALLY we have the first Gateway working ! Everything will be easier now since we will just focus on optimizing it.

Let's run everything:

```bash
python gateway.py
python client.py

>>> Sending request to addr1
>>> Sending request to addr2
```

The Gateway received the requests and did its job of broadcasting it to all the Executors :smile:

Even if it is only supposed to forward the requests there actually still is a de/serialization step that we need to optimize.
Indeed, each time the raw protobuf bytes arrive to the Gateway they are deserialized into a python object, that's slow and useless since,
we don't need it to actually be a python object, we just want to forward the raw bytes.

### Implement the lazy deserialization

To avoid this extra step we will deserialize the request only when it is needed, i.e. only when we need to access the content of the requests.
And we will do that in a lazy manner. Meaning that it will be invisible in the code of the GatewayServicer that this extra step happened.




















