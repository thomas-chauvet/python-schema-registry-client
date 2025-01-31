# How to use it with Faust

This section describe how integrate this library with [Faust](https://faust.readthedocs.io/en/latest/)

## Schemas, Custom Codecs and Serializers

Because we want to be sure that the message that we encode are valid, we can use [Avro](https://docs.oracle.com/database/nosql-12.1.3.1/GettingStartedGuide/avroschemas.html) or [JSON](https://json-schema.org/) schemas. Also, [Introduction to Schemas in Apache Kafka with the Confluent Schema Registry](https://medium.com/@stephane.maarek/introduction-to-schemas-in-apache-kafka-with-the-confluent-schema-registry-3bf55e401321) is a good post to start with `schemas`.
Avro and JSON can be used to define the data schema for a record's value. This schema describes the fields allowed in the value, along with their data types.

In order to use `avro schemas` or `json schemas` with `Faust`, we need to define a custom codec and a custom serializer able to talk with the `schema-registry`, and to do that, we will use the `MessageSerializer`.

For serializing `avro schemas` we should use the `FaustSerializer`. For serializing `json schemas` we should use the `FaustJsonSerializer`.

For our demonstration, let's imagine that we have the following `avro schema`:

```json
{
    "type": "record",
    "namespace": "com.example",
    "name": "AvroUsers",
    "fields": [
        {"name": "first_name", "type": "string"},
        {"name": "last_name", "type": "string"}
    ]
}
```

Let's register the custom `codec`

```python title="Trivial Usage"
# codecs.codec.py
from schema_registry.client import SchemaRegistryClient, schema
from schema_registry.serializers.faust import FaustSerializer

# create an instance of the `SchemaRegistryClient`
client = SchemaRegistryClient(url=settings.SCHEMA_REGISTRY_URL)

# schema that we want to use. For this example we
# are using a dict, but this schema could be located in a file called avro_user_schema.avsc
avro_user_schema = schema.AvroSchema({
    "type": "record",
    "namespace": "com.example",
    "name": "AvroUsers",
    "fields": [
        {"name": "first_name", "type": "string"},
        {"name": "last_name", "type": "string"}
    ]
})

avro_user_serializer = FaustSerializer(client, "users", avro_user_schema)

# function used to register the codec
def avro_user_codec():
    return avro_user_serializer
```

and add in `setup.py` the following code in order to tell faust where to find the custom codecs.

```python title="setup.py"
setup(
    ...
    entry_points={
        'console_scripts': [
            'example = example.app:main',
        ],
        'faust.codecs': [
            'avro_users = example.codecs.avro:avro_user_codec',
        ],
    },
)
```

Now the final step is to integrate the faust model with the AvroSerializer.

```python title="user.models.py"
import faust


class UserModel(faust.Record, serializer='avro_users'):
    first_name: str
    last_name: str
```

Now our application is able to send and receive message using arvo schemas!!!! :-)

```python title="application"
import logging

from your_project.app import app
from .codecs.codec import avro_user_serializer
from .models import UserModel

users_topic = app.topic('avro_users', partitions=1, value_type=UserModel)

logger = logging.getLogger(__name__)


@app.agent(users_topic)
async def users(users):
    async for user in users:
        logger.info("Event received in topic avro_users")
        logger.info(f"First Name: {user.first_name}, last name {user.last_name}")


@app.timer(5.0, on_leader=True)
async def publish_users():
    logger.info('PUBLISHING ON LEADER FOR USERS APP!')
    user = {"first_name": "foo", "last_name": "bar"}
    await users.send(value=user, value_serializer=avro_user_serializer)
```

The full example is [here](https://github.com/marcosschroh/faust-docker-compose-example/blob/master/faust-project/example/codecs/avro.py)

### Usage with dataclasses-avroschema for avro schemas

You can also use this funcionality with [dataclasses-avroschema](https://github.com/marcosschroh/dataclasses-avroschema) and you won't have to provide the avro schema.
The only thing that you need to do is add the `AvroModel` class and use its methods:

```python title="user.models.py"
import faust

from dataclasses_avroschema import AvroModel


class UserModel(faust.Record, AvroModel, serializer='avro_users'):
    first_name: str
    last_name: str


# codecs.codec.py
from schema_registry.client import SchemaRegistryClient, schema
from schema_registry.serializers.faust import FaustSerializer

from users.models import UserModel

# create an instance of the `SchemaRegistryClient`
client = SchemaRegistryClient(url=settings.SCHEMA_REGISTRY_URL)

avro_user_serializer = FaustSerializer(client, "users", UserModel.avro_schema())  # usign the method avro_schema to get the avro schema representation

# function used to register the codec
def avro_user_codec():
    return avro_user_serializer
```

### Usage with pydantic for json schemas
You can also use this funcionality with [dataclasses-pydantic](https://github.com/samuelcolvin/pydantic) and you won't have to provide the json schema.
The only thing that you need to do is add the `BaseModel` class and use its methods:

```python title="users.models.py"
import faust

from pydantic import BaseModel


class UserModel(faust.Record, BaseModel, serializer='json_users'):
    first_name: str
    last_name: str


# codecs.codec.py
from schema_registry.client import SchemaRegistryClient, schema
from schema_registry.serializers.faust import FaustJsonSerializer

from users.models import UserModel

# create an instance of the `SchemaRegistryClient`
client = SchemaRegistryClient(url=settings.SCHEMA_REGISTRY_URL)

json_user_serializer = FaustJsonSerializer(client, "users", UserModel.schema_json())  # usign the method schema_json to get the json schema representation

# function used to register the codec
def json_user_codec():
    return json_user_serializer
```
