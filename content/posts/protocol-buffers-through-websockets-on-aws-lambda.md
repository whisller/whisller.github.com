---
title: "Protocol Buffers RPC calls through WebSockets on AWS Lambda with API Gateway"
date: 2024-01-04
description: "Protocol Buffers RPC calls through WebSockets on AWS Lambda with API Gateway"
tags: ["protocol buffers", "websockets", "aws", "lambda", "python"]
---

We might argue about what is the important aspect of good communication.

But what are the few basic principles of communication where, at least, two parties will understand each other? Use of the same __*Glossary*__, happens in __*Real Time*__ and uses __*Same language*__.

Ok, but how that's connected to Protocol Buffers, WebSockets, AWS?

Systems built nowadays use different technologies and languages. You might see Python/PHP/Java on the backend, JavaScript/Swift/Kotlin on frontend (including mobile) apps. It's expected that the backend and frontend communicate with each other in real time and are able to share data without things "getting lost in translation".

## How?
To show how it could be done programmatically, we will build a simple backend service that allows you to fetch the geolocation of a few places in the world. I will use __Python__, __AWS Lambda__ + __AWS API Gateway__, __WebSockets__, and __[serverless.com](https://www.serverless.com/)__ to achieve that.

If you want to go straight to code, you can clone/fork [github.com/whisller/article-protocol-buffers-aws-lambda](https://github.com/whisller/article-protocol-buffers-aws-lambda).

## Protocol Buffers
Our communication wouldn't be successful if we would not agree on a common vocabulary. It could be even worse if we would have slight details lost in translation, that could lead to unpredictable problems that could be hard to track down.

Idea behind Protocol Buffers is to give you a tool that will let you serialize data in e.g. Java, and send it through the network, or store it in a database (file, database etc.) for later use by different language e.g. PHP without losing data structure/format.

Protocol Buffers is not the only tool that does that. Have also a look at [Apache Avro](https://avro.apache.org/), [Cap'N Proto](https://capnproto.org/), [FlatBuffers](https://flatbuffers.dev/).
Even though the main idea behind them is very similar, they differ quite a lot. So it's always good to do an assessment before choosing your tool.

### Schema
Schema will allow us to define RPC methods and data structure that we will send through the network.

#### RPC (Remote Procedure Call)
When you hear "Protocol Buffers" and "RPC" the first thing that comes to your mind is [gRPC](https://grpc.io/), isn't it? :)
But I have zero experience with setting it up in AWS Lambda context, with WebSockets on top of it, it might be even more challenging.

So we will try a simpler approach, use of Protocol Buffers message to encapsulate information about invoked function and passed parameters.


```proto
// proto/rpc.proto
syntax = "proto3";

message RPCCall {
  /**
   * Envelope for RPC call
   */

  // RPC method that will be called
  string method = 1;

  // Serialised params that will be passed to RPC method
  RPCCallParams params = 2;
}

message RPCCallParams {
  int32 limit = 1;
}
```

and definition of our services
```proto
// proto/service.proto

syntax = "proto3";

import "rpc.proto";
import "google/type/latlng.proto";

message InterestingPlace {
  string name = 1;
  google.type.LatLng location = 2;
}

message Response {
  repeated InterestingPlace results = 1;
}

service InterestingPlaces {
  rpc GetRandom(RPCCallParams) returns (Response);
}
```

I've split the definition of RPC from the service for easier maintenance and readability.

You can also see the custom type `google.type.LatLng` being used. This comes from [googleapis/googleapis](https://github.com/googleapis/googleapis), which contains quite a lot of useful types.

If your `protoc` is showing an error like:
```Bash
google/type/latlng.proto: File not found.
service.proto:6:1: Import "google/type/latlng.proto" was not found or had errors.
service.proto:28:3: "google.type.LatLng" is not defined.
```

Make sure that you:
1. Checkout [googleapis/googleapis](https://github.com/googleapis/googleapis)
2. Include it as import with `-I` e.g. `protoc -I=./googleapis`

If everything is in order, this command should generate Python files inside the `protobuf_classes` directory:
```Bash
protoc -I=proto -I=./googleapis --python_out=protobuf_classes proto/*.proto
```

{{< alert >}}
**Keep in mind**
{{< /alert >}}
1. "Why not to use `google.protobuf.service` instead?", as of `2.3.0` of Protocol Buffers [google.protobuf.service](https://googleapis.dev/python/protobuf/latest/google/protobuf/service.html#module-google.protobuf.service) is deprecated. Recommended way of dealing with it is to write your plugin to generate code.

{{< alert "circle-info" >}}
**Worth reading**
{{< /alert >}}
1. [googleapis/googleapis](https://github.com/googleapis/googleapis)

## AWS Lambda + WebSockets
As we have a Protocol Buffers schema that serves as the contract between the backend and clients, we can now build the backend side of it.

For handling CloudFormation on AWS, I will use the [serverless.com](https://www.serverless.com/) framework.

### Routing Messages from Client to Server Handler
API Gateway for WebSockets introduces the concept of routing messages. There are three predefined routes that we can use:
* `$connect`: triggered when a new connection from the client is initiated
* `$disconnect`: triggered when the connection is closed by either the client or the server
* `$default`: triggered when there is no other matching route


#### serverless.yml
```yaml
functions:
  Connection:
    handler: interesting_places.handler_connection.handle
    events:
      - websocket:
          route: $connect
      - websocket:
          route: $disconnect

  DefaultMessage:
    handler: interesting_places.handler_message.handle
    events:
      - websocket:
          route: $default
```

This way, we route `$connect` and `$disconnect` to one lambda, which can be used to manage sessions. All other messages, in our case Protocol Buffers messages, will be routed to `$default`.

### Granting Permission to Send Messages Back from Server to Connected Clients
To send messages back from the server to the client, we need to grant permission to the role that executes the lambda.


#### serverless.yml

```yaml
provider:
  iam:
    role:
      statements:
        - Sid: Policy739283796a5497fadc13d1404b932eb
          Effect: Allow
          Action:
            - execute-api:ManageConnections
          Resource:
            - - arn:aws:execute-api:${aws:region}:${aws:accountId}:*/${sls:stage}/POST/@connections/*
```

{{< alert >}}
**Keep in mind**
{{< /alert >}}
1. Binary frames are not supported by API Gateway. You can read about it in [Working with binary media types for WebSocket APIs](https://docs.aws.amazon.com/apigateway/latest/developerguide/websocket-api-develop-binary-media-types.html)

{{< alert "circle-info" >}}
**Worth reading**
{{< /alert >}}
1. [About WebSocket APIs in API Gateway](https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-websocket-api-overview.html), AWS documentation
2. [WebSockets](https://www.serverless.com/framework/docs/providers/aws/events/websocket), serverless.com documentation
3. [Sending data from backend services to connected clients](https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-websocket-api-data-from-backend.html)

### Lambda handler for WebSocket communication
With Protocol Buffers schema and CloudFormation definition being ready, we can implement some python code.

```Python
import base64

from requests_iam_session import AWSSession

from . import services
from .protobuf_classes import rpc_pb2


def handle(event, context):
    # Unique identifier of open WebSocket connection e.g "Q-Fj2fXjoAACJuw="
    connection_id = event["requestContext"]["connectionId"]
    # API Gateway domain e.g. xyz.execute-api.us-east-1.amazonaws.com
    domain_name = event["requestContext"]["domainName"]
    # Your API stage, e.g. staging
    stage = event["requestContext"]["stage"]

    body = base64.b64decode(event["body"])

    rpc_call = rpc_pb2.RPCCall()
    rpc_call.ParseFromString(body)

    # Dynamically select service that contains business logic
    response = getattr(services, rf"service_{rpc_call.method}")(rpc_call.params)
    encoded_response = base64.b64encode(response.SerializeToString()).decode("utf-8")

    # Send a message back to connected client
    session = AWSSession(f"https://{domain_name}")
    session.post(f"/{stage}/@connections/{connection_id}", data=encoded_response)
```

I believe the code is self-explanatory. Here's a breakdown of what we do:
1. Decode the message.
2. Map it with the Protocol Buffers schema.
3. Call our Python function based on the message's payload and obtain the Protocol Buffers response.
4. Sign the response message using [Signature Version 4 (SigV4)](https://docs.aws.amazon.com/IAM/latest/UserGuide/create-signed-request.html) and send it back to the client.

Our service could be as simple as:

```Python
import random

from google.type.latlng_pb2 import LatLng

from .protobuf_classes import rpc_pb2, service_pb2

places = [
    dict(name="Natural History Museum, London, UK", location=(51.496727, -0.176484)),
    dict(
        name="Copernicus Science Centre",
        location=(52.24189590931264, 21.028735065872638),
    ),
    dict(name="Ustka Lighthouse", location=(54.58805789697384, 16.8546758958559)),
    dict(name="The British Museum", location=(51.51954671919121, -0.12699952985545082)),
]


def service_GetRandom(params: rpc_pb2.RPCCallParams):
    limit = params.limit

    random_places = random.sample(places, limit)

    results = []
    for random_place in random_places:
        latlng = LatLng()
        latlng.latitude = random_place["location"][0]
        latlng.longitude = random_place["location"][1]

        interesting_place = service_pb2.InterestingPlace()
        interesting_place.name = random_place["name"]
        interesting_place.location.CopyFrom(latlng)

        results.append(interesting_place)

    response = service_pb2.Response()
    response.results.extend(results)

    return response
```

{{< alert "circle-info" >}}
**Worth reading**
{{< /alert >}}
1. [Invoking a WebSocket API](https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-how-to-call-websocket-api.html)

## Final Thoughts
If you want to see an example project that you can deploy, please visit [whisller/article-protocol-buffers-aws-lambda](https://github.com/whisller/article-protocol-buffers-aws-lambda).

Obviously, it's not an application that I would recommend putting into production. Still, it should give you a good understanding of starting to build your own WebSocket communication with Protocol Buffers in the AWS Lambda ecosystem.

## The End
That's it! I hope you found something useful in this post! If your company needs some help with AWS, [get in touch]({{< ref "/contact" >}} "Get in touch").
