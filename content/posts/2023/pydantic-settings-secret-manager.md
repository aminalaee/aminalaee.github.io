---
title: "Pydantic Settings with AWS Secrets Manager"
date: 2023-01-26T16:53:06+01:00
tags: ["python", "pydantic", "aws", "secrets manager"]
author: "Amin"
hidemeta: false
comments: true
description: ""
canonicalURL: ""
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
ShowRssButtonInSectionTermList: true
showtoc: true
tocopen: true
cover:
    image: ""
    alt: ""
    caption: ""
    relative: false
    hidden: true
---

## Intro

If you are already familiar with `Pydantic`, one of the useful components of `Pydantic` is the [Settings Management](https://docs.pydantic.dev/usage/settings/).
This will allow you to read settings variables from different sources and parse and validate them into class(es) using `Pydantic`. 

Let's see a minimal example. First we need to set up the variable:

```shell
$ export API_KEY=xxx
```

And then we can read it into the `Settings` class with:

```py
from pydantic import BaseSettings


class Settings(BaseSettings):
    api_key: str


print(Settings().dict())
"""
{'api_key': 'xxx'}
"""
```

Different data sources are available to read variables from, currently the options include:

- Environment variables
- Dotenv file
- Docker secrets

But what if you want to add new data sources? For example if you want to read it from a JSON file or AWS Secrets Manager?

Luckily `Pydantic` Settings allows you to customize the settings resources easily. Checking [here](https://docs.pydantic.dev/usage/settings/#customise-settings-sources) you can add or remove resources or change their priorities.

So let's see how we can add a custom resource to read secrets from AWS Secrets Manager.

## AWS Secrets Manager

**Note**: If you are familiar with AWS Secrets Manager or already have access to an existing one with some secrets defined you can skip to the next section.

Secrets Manager is a service which allows you to store any sensitive information like database passwords, API Keys, etc and retrieve them in your code without worrying about environment variables or dotenv files being passed around in your source code. Obviously your secrets are encrypted in AWS and decrypted in two steps when you want to retrieve them.

On top of this, you also get other benefits like secret rotation, logging and managing permissions based on AWS IAM policies.

If you already have an AWS account and credentials you can move to the next step and use the example code, otherwise I will use [localstack](https://github.com/localstack/localstack) to create a local version of all AWS services and test this without actually working with an AWS account.

You can install `localstack` easily with:

```shell
$ pip install localstack
```

After it's installed, you can start `localstack` to run using `Docker`:

```shell
$ localstack start -d
```

AWS services should now be available on your local at `http://localhost:4566`.

If you alrady have `aws` CLI installed, you can try using the `localstack` with:

```shell
$ export AWS_ACCESS_KEY_ID="test"
$ export AWS_SECRET_ACCESS_KEY="test"
$ export AWS_DEFAULT_REGION="us-east-1"

$ aws --endpoint-url=http://localhost:4566 secretsmanager list-secrets
```

Or you can install `awslocal`, which is a wrapper around `aws` CLI and you don't need to use the `--endpoint-url` anymore.

```shell
$ pip install awscli-local

$ awslocal secretsmanager list-secrets
```

This will probably get you some empty response because we haven't defined any secrets yet.

Now let's try to define two secrets:

```shell
$ awslocal secretsmanager create-secret \
    --name example_secret \
    --secret-string "SECRET_VALUE"

$ awslocal secretsmanager create-secret \
    --name example_complex \
    --secret-string "{\"user\":\"diegor\",\"password\":\"EXAMPLE-PASSWORD\"}"
```

Next let's try to list the secrets again with `awslocal secretsmanager list-secrets` which will give a similiar response to this:

```shell
{
    "SecretList": [
        {
            "ARN": "arn:aws:secretsmanager:us-east-1:000000000000:secret:example_secret-xxxx",
            "Name": "example_secret",
            "LastChangedDate": xxxxxxxxxxxxxxxxx,
            "SecretVersionsToStages": {
                "e49ae6a9-201b-45a8-93ea-a474cf3427cb": [
                    "AWSCURRENT"
                ]
            },
            "CreatedDate": xxxxxxxxxxxxxxxxx
        },
        {
            "ARN": "arn:aws:secretsmanager:us-east-1:000000000000:secret:example_complex-xxxx",
            "Name": "example_complex",
            "LastChangedDate": xxxxxxxxxxxxxxxxx,
            "SecretVersionsToStages": {
                "87b93858-f03a-431e-8560-2c1ec836945f": [
                    "AWSCURRENT"
                ]
            },
            "CreatedDate": xxxxxxxxxxxxxxxxx
        }
    ]
}
```

Now we can move to our implementation code.

## Pydantic Settings with Secrets Manager

The first step is to connect to the AWS secrets manager service using `boto3` and create a client. In case of using `localstack` you can use the following credentials to connect to it:

```python
from boto3 import Session


session = Session(
    aws_access_key_id="test",
    aws_secret_access_key="test",
    region_name="us-east-1",
)

client = session.client(
    endpoint_url="http://localhost:4566",
    service_name="secretsmanager",
    region_name="us-east-1",
)
```

Pydantic `BaseSettings` class can use a nested `Config` class which does extra configurations for the settings management. Next we define a `SecretManagerConfig` class which will modify `Config`'s behaviour:

```python
import json
from typing import Any

from pydantic.env_settings import SettingsSourceCallable


class SecretManagerConfig:
    @classmethod
    def _get_secret(cls, secret_name: str) -> str | dict[str, Any]:
        # Get secret string value
        secret_string = client.get_secret_value(SecretId=secret_name)["SecretString"]
        try:
            # try to decode it into a dict if it was JSON encoded
            return json.loads(secret_string)
        except json.decoder.JSONDecodeError:
            return secret_string
        

    @classmethod
    def get_secrets(cls, settings: BaseSettings) -> dict[str, Any]:
        # We go through the fields and try to fetch a secret with that field name
        return {
            name: cls._get_secret(name) for name, _ in settings.__fields__.items()
        }

    @classmethod
    def customise_sources(
        cls,
        init_settings: SettingsSourceCallable,
        env_settings: SettingsSourceCallable,
        file_secret_settings: SettingsSourceCallable,
    ):
        # Here we add the `cls.get_secrets` method as an extra data source
        return (
            init_settings,
            cls.get_secrets,
            env_settings,
            file_secret_settings,
        )
```

The method `customise_sources`  is the method that `Pydantic` Settings allows you to add or delete new data sources. So it's overriden to add our custom data source `get_secrets` there.

In `get_secrets` we go through the `Settings.__fields__` and get the secret details by using this name. For each secret string returned, we can try to load it into a dict using `json.loads` in case our secret was JSON encoded.

And we define our settings classes next:

```python
from pydantic import BaseSettings, BaseModel


class DatabaseSettings(BaseModel):
    user: str
    password: str


class Settings(BaseSettings):
    example_secret: str
    example_complex: DatabaseSettings

    class Config(SecretManagerConfig):
        ...
```

And finally we can load our settings with:

```python
print(Settings().dict())
"""
{'example_secret': 'SECRET_VALUE', 'example_complex': {'user': 'diegor', 'password': 'EXAMPLE-PASSWORD'}}
"""
```

This allows us to load both simple key, value secrets and JSON encoded ones into sub-settings which `Pydantic` handles very nicely.

Obviously this can be improved to handle more cases or even be written in better ways, but this will give you the idea how you can do continue from here.

The complete code:

```python
import json
from typing import Any

from boto3 import Session
from pydantic import BaseSettings, BaseModel
from pydantic.env_settings import SettingsSourceCallable


session = Session(
    aws_access_key_id="test",
    aws_secret_access_key="test",
    region_name="us-east-1",
)

client = session.client(
    endpoint_url="http://localhost:4566",
    service_name="secretsmanager",
    region_name="us-east-1",
)


class SecretManagerConfig:
    @classmethod
    def _get_secret(cls, secret_name: str) -> str | dict[str, Any]:
        secret_string = client.get_secret_value(SecretId=secret_name)["SecretString"]
        try:
            return json.loads(secret_string)
        except json.decoder.JSONDecodeError:
            return secret_string
        

    @classmethod
    def get_secrets(cls, settings: BaseSettings) -> dict[str, Any]:
        return {
            name: cls._get_secret(name) for name, _ in settings.__fields__.items()
        }

    @classmethod
    def customise_sources(
        cls,
        init_settings: SettingsSourceCallable,
        env_settings: SettingsSourceCallable,
        file_secret_settings: SettingsSourceCallable,
    ):
        return (
            init_settings,
            cls.get_secrets,
            env_settings,
            file_secret_settings,
        )


class DatabaseSettings(BaseModel):
    user: str
    password: str


class Settings(BaseSettings):
    example_secret: str
    example_complex: DatabaseSettings

    class Config(SecretManagerConfig):
        ...


print(Settings().dict())
"""
{'example_secret': 'SECRET_VALUE', 'example_complex': {'user': 'diegor', 'password': 'EXAMPLE-PASSWORD'}}
"""
```

