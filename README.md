
# sfprotostash

Pre-generated Python gRPC stubs for the [Salesforce Pub/Sub API](https://developer.salesforce.com/docs/platform/pub-sub-api/guide/intro.html).

This package ships the generated `pubsub_api_pb2` and `pubsub_api_pb2_grpc` modules so you don't have to run `protoc` yourself. Just `pip install` and import.

## Installation

[Official Pypi Page](https://pypi.org/manage/project/sfprotostash/release/0.1.0/.html).


```bash
pip install sfprotostash
```
https://pypi.org/manage/project/sfprotostash/release/0.1.0/

`grpcio` and `protobuf` are pulled in automatically as dependencies.

## What's inside

| Module | Contents |
| --- | --- |
| `pubsub_api_pb2` | Message types (`TopicRequest`, `FetchRequest`, `SchemaRequest`, etc.) |
| `pubsub_api_pb2_grpc` | The `PubSubStub` client and service definitions |

```python
from sfprotostash import pubsub_api_pb2
from sfprotostash import pubsub_api_pb2_grpc
```
