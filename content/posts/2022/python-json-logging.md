---
title: "Python JSON Logging"
date: 2022-09-14T15:09:46+02:00
tags: [python, logging, picologging]
author: "Amin"
hidemeta: false
comments: true
description: "How to do simple JSON logging in Python?"
canonicalURL: ""
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
ShowRssButtonInSectionTermList: true
cover:
    image: "https://www.loggly.com/wp-content/uploads/2017/03/0000_why-json-is-the-best-application-log-format-and-how-to-switch.png"
    alt: "JSON Logging"
    caption: "Taken from loggly.com"
    relative: false
    hidden: false
---

## Use-case

It's very common for projects to do `JSON` logging if you are working with
third-party tools or open-source projects like `Logstash` to process your logs.
These tools usually need more complex filtering on the structured data, so using JSON is preferred there.

We also wanted to integrate with a third-party tool at work and we needed to add the JSON
formatting logs in our projects.

I will not go into the details of why or why not you should decide JSON logging,
as each approach will have it's pros and cons. Instead I will explain how you can do it in Python.

The final goal would be to go from:

```
2022-09-14 23:47:11,506248 - myapp - DEBUG - debug message
```

To:

```json
{
    "threadName": "MainThread",
    "name": "root",
    "thread": 140735202359648,
    "created": 1336281068.506248,
    "process": 41937,
    "processName": "MainProcess",
    "relativeCreated": 9.100914001464844,
    "module": "app",
    "funcName": "do_logging",
    "levelno": 20,
    "pathname": "app.py",
    "lineno": 20,
    "asctime": ["2022-09-14 23:47:11,506248"],
    "message": "debug message",
    "filename": "main.py",
    "levelname": "DEBUG",
}
```

## Existing projects

If you do a quick search, like I did, you will find two (more or less) active projects which do this:

- [python-json-logger](https://github.com/madzak/python-json-logger)
- [json-log-formatter](https://github.com/marselester/json-log-formatter)
- And more

And it's interesting that if you check the downloads of these project at [PePY](https://pepy.tech/)
or some other tool you see many people probably actually use them. As of writing this, I checked that `python-json-logger` has a daily download rate of ~200K per day!

## Why I think you probably don't need that

The Python `logging` module provides a `Formatter` which can be used
to do logging in any formatting you want.

A very simple and minimal example of a `JSON` formatter can be written as:

```python
import json
import logging


class JSONFormatter(logging.Formatter):
    def __init__(self) -> None:
        super().__init__()

        self._ignore_keys = {"msg", "args"}

    def format(self, record: logging.LogRecord) -> str:
        message = record.__dict__.copy()
        message["message"] = record.getMessage()

        for key in self._ignore_keys:
            message.pop(key, None)

        if record.exc_info and record.exc_text is None:
            record.exc_text = self.formatException(record.exc_info)

        if record.exc_text:
            message["exc_info"] = record.exc_text

        if record.stack_info:
            message["stack_info"] = self.formatStack(record.stack_info)

        return json.dumps(message)
```

The code is really simple, for each record you will get the `dict` from the record,
and turn in to JSON with `json.dumps()`.

There are only some conditions to add `stack_info` and `exc_info` if they are available,
which should format exception info according to the record. You can easily modify that to fit your needs.

And to use this formatter with the loggers:

```python
import logging

# import JSONFormatter

logger = logging.getLogger(__name__)
logger.setLevel(logging.DEBUG)

handler = logging.StreamHandler()
handler.setFormatter(JSONFormatter())

logger.addHandler(handler)

logger.debug("debug message")
```

which will output:

```json
{
    "name": "__main__",
    "levelname": "DEBUG",
    "levelno": 10,
    "pathname": "main.py",
    "filename": "main.py",
    "module": "main",
    "exc_info": null,
    "exc_text": null,
    "stack_info": null,
    "lineno": 38,
    "funcName": "<module>",
    "created": 1663168021.864416,
    "msecs": 864.4158840179443,
    "relativeCreated": 1.2068748474121094,
    "thread": 8673392128,
    "threadName": "MainThread",
    "processName": "MainProcess",
    "process": 14747,
    "message": "debug message",
}
```

For list of all `LogRecord` attributes you can check the Python's [documentation](https://docs.python.org/3/library/logging.html#logrecord-attributes).

That's why I think for a code so simple, you are probably better off with implementing in your own code,
rather than relying on a third-party package.
