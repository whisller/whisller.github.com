---
title: "Protocol Buffers RPC calls through WebSockets on AWS Lambda with API Gateway"
date: 2024-01-04
description: "Protocol Buffers RPC calls through WebSockets on AWS Lambda with API Gateway"
tags: ["protocol buffers", "websockets", "aws", "lambda", "python", "devops"]
---

We might argue about what is the important aspect of good communication.

But what are the few basic principles of communication where, at least, two parties will understand each other? Use of the same __*Glossary*__, happens in __*Real Time*__ and uses __*Same language*__.

Ok, but how that's connected to Protocol Buffers, WebSockets, AWS?

Systems built nowadays use different technologies and languages. You might see Python/PHP/Java/etc. on the backend, JavaScript/Swift/Kotlin/etc. on frontend (including mobile) apps. It's expected that the backend and frontend communicate with each other in real time and are able to share data without loosing semantic e.g. double data type.

## How?
To show how it could be done programmatically, we will build a simple backend service that returns the geolocation of a few places in the world. I will use __Python__, __AWS Lambda__ + __AWS API Gateway__, __WebSockets__, and __[serverless.com](https://www.serverless.com/)__ to achieve that.

If you want to go straight to code, you can clone/fork [github.com/whisller/article-protocol-buffers-aws-lambda](https://github.com/whisller/article-protocol-buffers-aws-lambda).

## Protocol Buffers
Our communication wouldn't be successful if we would not agree on a common vocabulary, same comes if different technologies/languages sharing data would interpret it differently. That bugs lead to unpredictable problems that could be hard to track down.

Protocol Buffers is a tool that lets you serialize data and send it through the network, or you can save this data in a database (file, database etc.) for later use, without losing its structure/format.

Protocol Buffers is not the only tool that does that. There is at least few that tries to achieve similar goal, but they differ in dealing with data.

To list some of them:
* [Apache Avro](https://avro.apache.org/)
* [Cap'N Proto](https://capnproto.org/)
* [FlatBuffers](https://flatbuffers.dev/).

So before choosing tool check their props & cons.

### Schema
Protocol Buffers schema allows us to define data structure for messages that we will send from client to backend and back, through WebSocket connection.

#### RPC (Remote Procedure Call)
When you hear "Protocol Buffers" and "RPC" the first thing that comes to your mind is [gRPC](https://grpc.io/), isn't it? :)
But today I will not talk about gRPC, main reason for it is that I haven't tried to set up it in AWS ecosystem, so my experience on this field is close to zero.

I will take different approach, use of Protocol Buffers message to encapsulate information about invoked function and passed parameters.


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

You can also see the custom type `google.type.LatLng` being used. That's one of the types defined in [googleapis/googleapis](https://github.com/googleapis/googleapis).

At this point you should have:
* [protoc](https://grpc.io/docs/protoc-installation/) installed
* [googleapis/googleapis](https://github.com/googleapis/googleapis) checked out locally (to be used by `protoc`)

We can now test if our Protocol Buffers schema is correct:
```Bash
protoc -I=proto -I=./googleapis --python_out=protobuf_classes proto/*.proto
```

If everything is correct, this command will generate Python files inside the `protobuf_classes` directory.

## AWS Lambda + WebSockets
For handling CloudFormation on AWS, I will use the [serverless.com](https://www.serverless.com/) framework.

### Routing Messages from Client to Server Handler
API Gateway for WebSockets has a concept of routing messages. There are three predefined routes that we can use:
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

{{< alert "circle-info" >}}
**Worth reading**
{{< /alert >}}
1. [About WebSocket APIs in API Gateway](https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-websocket-api-overview.html), AWS documentation
2. [WebSockets](https://www.serverless.com/framework/docs/providers/aws/events/websocket), serverless.com documentation

This way, we route `$connect` and `$disconnect` to one lambda, which can be used to manage sessions. All other events, will be routed to `$default`.

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

{{< alert "circle-info" >}}
**Worth reading**
{{< /alert >}}
1. [Using IAM authorization](https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-websocket-control-access-iam.html)

### Lambda handler for WebSocket communication
Lambda handler is a place that will listen to API Gateway events and will let us communicate back to client through open WebSocket connection.

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

Here's a breakdown of what this lambda function does:
1. Decode the message.
2. Map it to the Protocol Buffers `RPCCall` message
3. Based on `RPCCall.method` calls corresponding service function that executes business logic and returns back prepared `Response` Protocol Buffers message
4. Sign the message using [Signature Version 4 (SigV4)](https://docs.aws.amazon.com/IAM/latest/UserGuide/create-signed-request.html) and sends it back to the client through open WebSockets connection

{{< alert >}}
**Keep in mind**
{{< /alert >}}
1. Binary frames are not supported by API Gateway. You can read about it in [Working with binary media types for WebSocket APIs](https://docs.aws.amazon.com/apigateway/latest/developerguide/websocket-api-develop-binary-media-types.html)

{{< alert "circle-info" >}}
**Worth reading**
{{< /alert >}}
1. [Sending data from backend services to connected clients](https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-websocket-api-data-from-backend.html)


Our service function could be as simple as:

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

There is also example of [python client](https://github.com/whisller/article-protocol-buffers-aws-lambda/blob/main/client.py) that you can use to communicate with this test project.

Obviously, it's not production ready. Still I hope it gave you some starting point to build your own project.

## The End
That's it! I hope you found something useful in this post! If your company needs some help with AWS, [get in touch]({{< ref "/contact" >}} "Get in touch").
