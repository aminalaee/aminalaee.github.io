---
title: "Pytest async tests and fixtures"
date: 2023-06-29T12:00:06+01:00
tags: ["python", "pytest", "asyncio", "testing"]
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
---

## Async tests and fixtures

Pytest already has the official `pytest-asyncio` plugin which allows
you to write async tests and fixtures.

According to `pytest-asyncio` docs, you can have an `async` test
with an `async` fixture like this:

```python
from typing import Any, AsyncGenerator

import pytest
import pytest_asyncio
from httpx import AsyncClient


@pytest_asyncio.fixture
async def client() -> AsyncGenerator[AsyncClient, Any]:
    async with AsyncClient() as c:
        yield c


@pytest.mark.asyncio
async def test_api_call(client: AsyncClient) -> None:
    response = await client.get("http://test.com")
    assert response.status_code == 200
```

There are a few points to consider here:

- Obviously the fixture and test are defined as `async def` since that was the intention.
- The fixture should be decorated with `@pytest_asyncio.fixture` so pytest can pick this up properly.
- The tests should be marked with `@pytest.mark.asyncio` decorator.

As an alternative you can define a variable `pytestmark = pytest.mark.asyncio`
in your test file to treat all tests as async and avoid the repetition.

## Discovery mode

This works well but with some tweaks it can be made simpler.
The `pytest-asyncio` offers a `discovery mode` concept which allows you to control how
async tests are discovered.

If your project only uses `asyncio` as the asynchronous programming library,
you can take advantage of the discovery mode to make this simpler:

```python
from typing import Any, AsyncGenerator

import pytest
from httpx import AsyncClient


@pytest.fixture
async def client() -> AsyncGenerator[AsyncClient, Any]:
    async with AsyncClient() as c:
        yield c


async def test_api_call(client: AsyncClient) -> None:
    response = await client.get("http://test.com")
    assert response.status_code == 200
```

This way:

- You can define fixtures with normal `@pytest.fixture` decorator.
- You don't need to mark tests as async, so no decorators needed.

You can now run the tests with:

```shell
$ pytest tests --asyncio-mode=auto 
```

If you are using `pyproject.toml` file you can add this with:

```toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
```

And run the tests with:

```shell
$ pytest tests
```

## Asyncio event loop

When testing a `FastAPI` or `SQLAlchemy` project,
you might get the error `<Task pending> attached to a different loop`.

The reason is that you should define event loop for pytest to use.
You can add the `event_loop` fixture to `conftest.py` with:

```python
import asyncio
from typing import Any, Generator

import pytest


@pytest.fixture(scope="session")
def event_loop() -> Generator[asyncio.AbstractEventLoop, Any, None]:
    loop = asyncio.get_event_loop()
    yield loop
    loop.close()
```
