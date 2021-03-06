---
title: Flask WebApp Configuration
date: 2021-03-06 18:32:46
categories: programming
tags:
    - Flask
    - Python
    - beginner
---

Whenever I'm doing something [Flask](https://flask.palletsprojects.com/) related for the first time my first stop is [Miguel Grinberg's Blog](https://blog.miguelgrinberg.com/). There, and in his [Flask book](https://flaskbook.com/) I learned about application factory pattern and how to handle different configurations between development, staging, production etc.

### Simple example

Entry point into application is `wsgi.py` which creates Flask application object. We read which configuration should be used from environment variable. Default is production, preventing accidentally serving insecure configurations in production if variable is missing (This will be a common theme today.).

```python
# wsgi.py

import os

from my_app import create_app

config = os.environ.get("MYAPP_CONFIG", "production")
app = create_app(config)
```

In `my_app/__init__.py` we have a standard application factory pattern. If you want to learn why this pattern is a good idea use resources above or Flask documentation. We won't dive into it in this post.

```python
# my_app/__init__.py

from flask import Flask
# ...

from config import config


def create_app(config_name):
    app = Flask(__name__)
    app.config.from_object(config[config_name])
    # ...
    return app
```

And finally we have a `config.py` file which holds different configurations. Most of the variables are read from environment in [12 factor](https://12factor.net/config) style.

```python
# config.py

import os


class Config:
    SECRET_KEY = os.environ.get("SECRET_KEY") or "development secret key"
    SQLALCHEMY_TRACK_MODIFICATIONS = False
    SQLALCHEMY_DATABASE_URI = os.environ.get("DATABASE_URL")


class DevelopmentConfig(Config):
    DEBUG = True
    SQLALCHEMY_DATABASE_URI = os.environ.get(
        "DATABASE_URL"
    ) or "sqlite:///" + os.path.join(basedir, "data-dev.sqlite")


class TestingConfig(DevelopmentConfig):
    TESTING = True
    WTF_CSRF_ENABLED = False
    SQLALCHEMY_DATABASE_URI = (
        os.environ.get("TEST_DATABASE_URL") or "sqlite:///:memory:"
    )


class ProductionConfig(Config):
    # these are duplicated from base config
    # but they ensure we do not get default values by mistake
    SECRET_KEY = os.environ.get("SECRET_KEY")


config = {
    "development": DevelopmentConfig,
    "testing": TestingConfig,
    "production": ProductionConfig,
}
```

Let's think about what this style of configuration gives us and what more should we want.

- Simple. Even Operations guy that does not know Python and the project intimately could read and understand this.
- No fussing around with imports like in module style of configuration.
- Powerful. Can execute Python code. (This can be abused obviously.)
- Configuration inheritance and possibility to redefine variables.
- Developer friendly. Most variables can have sensible defaults.

Pretty good for such simple implementation but it lacks certain features that are becoming more important deployment complexity grows.

### Ensuring variables are set

Maybe you noticed that `ProductionConfig` redefines `SECRET_KEY`. This is to prevent accidentally deploying without the variable set and running project with the default variable. Not all variables are critical in this regard but those that are can usually cause a lot of damage.

One problem with this implementation is that we must use `os.environ.get("SECRET_KEY")` instead of `os.environ["SECRET_KEY"]` otherwise code will fail even if we don't use `production` configuration. This means that error will show up later and not at start up. Maybe error won't even cause process to crash and will cause 500 errors and even then error messages will be pretty cryptic. If we could use `os.environ["SECRET_KEY"]` then the error would cause process to immediately crash and pretty reasonable error `KeyError: 'SECRET_KEY'` would show up. Person deploying this would have better idea what is wrong. In programming community this idea is also known as "Fail fast, fail hard.".

We can add this functionality really easily and make _huge_ security and operational gains. Changed `config.py` would look like this:

```python
class ProductionConfig(Config):
    def __init__(self):
        # if any parents define variables in __init__
        # we must call super().__init__() otherwise
        # those variables will not be inherited
        self.SECRET_KEY = os.environ["SECRET_KEY"]
        self._ensure_secure_secret_key()

    def _ensure_secure_secret_key(self):
        if len(self.SECRET_KEY) < 24:
            raise ValueError("Use stronger 'SECRET_KEY'")


_config = {
    "development": DevelopmentConfig,
    "testing": TestingConfig,
    "production": ProductionConfig,
}

def get_config(name):
    return _config[name]()
```

We can even insert additional checks in. Application factory would then use `get_config` function instead of using `config` dictionary directly. This idea can be then upgraded with properties and other fancy Python tools if needs arise.

Such simple change that can prevent a lot of mistakes and angry looks between Ops and Dev teams.
