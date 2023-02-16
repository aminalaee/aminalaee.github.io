---
title: "Python requests Connection Pool"
date: 2023-02-16T12:00:47+01:00
tags: ["python", "requests", "httpx", "http"]
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
    image: "images/http-persistent-connection.png"
    alt: "HTTP Persistent Connection"
    caption: "Taken from haproxy.com"
    relative: false
    hidden: false
---

## Summary

Recently I was working on a project that integrated with
some internal and external APIs using HTTP requests.
You probably have heard about or worked with the [`requests`](https://requests.readthedocs.io/en/latest/) library
in Python which is probably the de-facto HTTP client package which is
much easier to work with compared to the built-in HTTP module of Python.

The previous implementation in our project was using the `requests` library
and for each new request it did something like `requests.get(...)` to do the integration with other services.


If you have worked a bit with `requests` you probably know about the `Session` object or it's equivalent `Client` object in [`httpx`](https://www.python-httpx.org/).
This reminded me of this section in Python `httpx` documentation:

> If you do anything more than experimentation, one-off scripts, or prototypes, then you should use a `Client` instance.

The benefit of using `Session` or `Client` objects is that
you will be able to do HTTP persistent connections, and if that's supported by the server too,
the underlying TCP connection between your client and server will be kept open,
so you will re-use the same resources for future requests.

## Usage

Let's write a simple script to see how connection pooling works in action.
This script will just enable DEBUG logging and send some requests to `httpbin.org`.

Let's send some requests in a naive way:

```python
import logging

import requests


logging.basicConfig(level=logging.DEBUG)

requests.get("http://httpbin.org")
requests.get("http://httpbin.org")
```

And this is the log generated:

```
DEBUG:urllib3.connectionpool:Starting new HTTP connection (1): httpbin.org:80
DEBUG:urllib3.connectionpool:http://httpbin.org:80 "GET / HTTP/1.1" 200 9593
DEBUG:urllib3.connectionpool:Starting new HTTP connection (1): httpbin.org:80
DEBUG:urllib3.connectionpool:http://httpbin.org:80 "GET / HTTP/1.1" 200 9593
```

As expected a new HTTP connection is started per request.
Now let's try to use a `Session` object to see the difference.

```python
import logging

import requests


logging.basicConfig(level=logging.DEBUG)

session = requests.Session()
session.get("http://httpbin.org")
session.get("http://httpbin.org")
```

And now running this you will see something like:

```
DEBUG:urllib3.connectionpool:Starting new HTTP connection (1): httpbin.org:80
DEBUG:urllib3.connectionpool:http://httpbin.org:80 "GET / HTTP/1.1" 200 9593
DEBUG:urllib3.connectionpool:http://httpbin.org:80 "GET / HTTP/1.1" 200 9593
```

Great, for the first HTTP request a new HTTP connection is opened,
but for the second request no new connection is opened, thanks to the `urllib3`'s connection pool.

## Benchmarking

For the setup I'm running a dummy `Django` application with `gunicorn`
and using a `Caddy` as reverse proxy to handle the persistent connections.

So here I start the Django application:

```shell
$ gunicorn example.wsgi -w 4
```

And in another terminal, run reverse proxy:

```shell
$ caddy reverse-proxy --from :9000 --to :8000
```

And now let's run a simple script to benchmark the difference:

```python
import requests


url = "http://localhost:9000/"


def time_requests():
    elapsed = 0

    for _ in range(1_000):
        response = requests.get(url)
        elapsed += response.elapsed.total_seconds()

    print(f"time_requests --> elapsed: {elapsed}")


def time_requests_session():
    session = requests.Session()
    elapsed = 0

    for _ in range(1_000):
        response = session.get(url)
        elapsed += response.elapsed.total_seconds()

    print(f"time_requests_session --> elapsed: {elapsed}")


time_requests()
time_requests_session()
```

This is the result I get on my laptop,
but you should probably get the same results:

```
time_requests --> elapsed: 2.1866910000000015
time_requests_session --> elapsed: 1.7474469999999986
```

So we could easily reach 25% of performance improvement
with literally adding one line of code.
Of course this will vary based on your use-case and environment,
but generally it should be an improvement compared to the previous approach.

Now let's do the same thing with `httpx`:

```python
import httpx


def time_httpx_client():
    client = httpx.Client()
    elapsed = 0

    for _ in range(1_000):
        response = client.get(url)
        elapsed += response.elapsed.total_seconds()

    print(f"time_httpx_client --> elapsed: {elapsed}")


time_httpx_client()
```

Which gives me the same result as `requests.Session` approach. The nice thing about `httpx` is that even though it supports both sync and async usage,
most of the interface is similar to the `requests`.

### Final notes

You might be wondering why I'm using `response.elapsed.total_seconds()`
in both scripts. We could just get the timestamps before and after the loop and have the total time calculated.

I think the `requests` docs explains `elapsed` attribute very well:

> The amount of time elapsed between sending the request and the arrival of the response (as a timedelta). This property specifically measures the time taken between sending the first byte of the request and finishing parsing the headers. It is therefore unaffected by consuming the response content or the value of the stream keyword argument.

So this will (hopefully) give a more realistic benchmark that we are
not including the response body consumption into our benchmark,
but remember benchmarks can be misleading and you need to test it for your
own use-case to see how it works!
