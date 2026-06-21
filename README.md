INSTALL IT HERE https://pypi.org/project/sfprotostash/0.1.0/

# sfprotostash

Pre-generated Python gRPC stubs for the [Salesforce Pub/Sub API](https://developer.salesforce.com/docs/platform/pub-sub-api/guide/intro.html).

This package ships the generated `pubsub_api_pb2` and `pubsub_api_pb2_grpc` modules so you don't have to run `protoc` yourself. Just `pip install` and import.

## Installation

```bash
pip install sfprotostash
```

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

## Usage

The stubs handle the gRPC layer. You bring authentication. The Pub/Sub API expects three metadata headers on every call: `accesstoken`, `instanceurl`, and `tenantid`.

### Authenticate

Any Salesforce auth flow that yields an access token works. Below is the JWT bearer flow:

```python
import os, time, jwt, requests

with open(os.getenv("SF_JWT_KEY_PATH"), "rb") as f:
    private_key = f.read()

claim = {
    "iss": os.getenv("SF_CLIENT_ID"),
    "sub": os.getenv("SF_USERNAME"),
    "aud": "https://login.salesforce.com",   # use test.salesforce.com for sandboxes
    "exp": int(time.time()) + 300,
}
signed_jwt = jwt.encode(claim, private_key, algorithm="RS256")

resp = requests.post(
    "https://login.salesforce.com/services/oauth2/token",
    data={
        "grant_type": "urn:ietf:params:oauth:grant-type:jwt-bearer",
        "assertion": signed_jwt,
    },
)
resp.raise_for_status()
auth = resp.json()

access_token = auth["access_token"]
instance_url = auth["instance_url"]
tenant_id    = auth["id"].split("/")[-2]   # org ID
```

> JWT signing with RS256 requires `pyjwt[crypto]`:
> `pip install "pyjwt[crypto]"`

### Connect and call

```python
import grpc
from sfprotostash import pubsub_api_pb2 as pb2
from sfprotostash import pubsub_api_pb2_grpc as pb2_grpc

metadata = (
    ("accesstoken", access_token),
    ("instanceurl", instance_url),
    ("tenantid", tenant_id),
)

with grpc.secure_channel("api.pubsub.salesforce.com:7443",
                         grpc.ssl_channel_credentials()) as channel:
    stub = pb2_grpc.PubSubStub(channel)

    # Simplest round-trip: fetch a topic's info and schema_id
    topic = stub.GetTopic(
        pb2.TopicRequest(topic_name="/data/AccountChangeEvent"),
        metadata=metadata,
    )
    print(topic)
```

### Subscribe to events

`Subscribe` is a bidirectional stream. Event payloads come back as binary Avro and need the topic's schema to decode:

```python
import io, avro.schema, avro.io

schema_cache = {}

def get_schema(stub, schema_id):
    if schema_id not in schema_cache:
        schema_json = stub.GetSchema(
            pb2.SchemaRequest(schema_id=schema_id), metadata=metadata
        ).schema_json
        schema_cache[schema_id] = avro.schema.parse(schema_json)
    return schema_cache[schema_id]

def decode(schema, payload):
    decoder = avro.io.BinaryDecoder(io.BytesIO(payload))
    return avro.io.DatumReader(schema).read(decoder)

def fetch_requests(topic):
    while True:
        yield pb2.FetchRequest(
            topic_name=topic,
            replay_preset=pb2.ReplayPreset.LATEST,
            num_requested=1,
        )

with grpc.secure_channel("api.pubsub.salesforce.com:7443",
                         grpc.ssl_channel_credentials()) as channel:
    stub = pb2_grpc.PubSubStub(channel)
    stream = stub.Subscribe(fetch_requests("/data/AccountChangeEvent"),
                            metadata=metadata)
    for response in stream:
        for event in response.events:
            schema = get_schema(stub, event.event.schema_id)
            print(decode(schema, event.event.payload))
```

> `num_requested` is credit-based flow control, not a batch size. When the
> credits run out the stream pauses until you send another `FetchRequest`.
> Replenish credits as you process events.

## Topic names

| Type | Format | Example |
| --- | --- | --- |
| Platform event | `/event/<Name>__e` | `/event/MyEvent__e` |
| Change Data Capture | `/data/<Object>ChangeEvent` | `/data/AccountChangeEvent` |
| CDC (all changes) | `/data/ChangeEvents` | `/data/ChangeEvents` |

Note CDC channels use the `ChangeEvent` suffix — `/data/AccountChangeEvent`, not `/data/Account`.

## Updating the stubs

These stubs are generated from Salesforce's [`pubsub_api.proto`](https://github.com/forcedotcom/pub-sub-api). To refresh them when Salesforce updates the proto:

```bash
python -m grpc_tools.protoc --proto_path=protos \
  --python_out=src/sfprotostash \
  --grpc_python_out=src/sfprotostash \
  protos/pubsub_api.proto

# fix the bare import in the generated grpc file
sed -i '' 's/^import pubsub_api_pb2 as/from . import pubsub_api_pb2 as/' \
  src/sfprotostash/pubsub_api_pb2_grpc.py
```

Then bump the `version` in `pyproject.toml`, rebuild, and publish.

## License

MIT
