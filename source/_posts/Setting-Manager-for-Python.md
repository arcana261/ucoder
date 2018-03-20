title: Setting Manager for Python
author: Mohamad mehdi Kharatizadeh
tags:
  - Python
  - Configuration
  - Setting
categories: []
date: 2018-03-17 01:15:00
---
In this article, we'll discuss how to define an awesome setting manager for python which is configurable by environment variables, and can support multiple environments (like development, staging, production, etc.)

The setting manager which we are going to implement should conform to these requirements:

- ability to specify environment, e.g. `ENV=dev python app.py`
- ability to customize settings using environment variables, e.g. `ENV=dev PORT=3000 python app.py`
- ability to specify base defaults and environment defaults in python friendly manner (using modules). e.g. `settings/defaults.py`
and `settings/dev.py`.
- ability to load/unload configuration files on demand, e.g.  `settings.load('myfile.json')`.
- easy to use API e.g.

```python
import settings

print(settings.PORT)
```

## Proposed Solution

The solution I'm going to propose is a solution based on putting our setting manager inside our code/boilerplate. The directory structure looks like this:

```
<ROOT>
  |
  |- settings
        |
        - __init__.py
        - dev.py
        - prd.py
```

In order to keep API as simple as possible, we want to import settings and access setting properties as attributes! This requires overloading \_\_getattr\_\_ at module level! But wait! This is actually [PEP562](https://www.python.org/dev/peps/pep-0562/) which is intended for python 3.7 (which is just in RC at time of this writing) so we should perform a little hack in our \_\_init\_\_.py:

```python
class SettingManager:
    def __getattr__(self, item):
        pass
    def __contains__(self, item):
        pass
    def __iter__(self):
        pass
        
sys.modules[__name__] = SettingManager()
```

What this hack does is that it sets module \_\_name\_\_ to an instance of SettingManager. which in turn has \_\_getattr\_\_ implemented. So basically when someone tries to access
`settings.SOME_SETTING` we can handle this safely in \_\_getattr\_\_.

So now that we have basic mechanism of loading and populating settings it is now fairly trivial to do some loading and so forth. The full code is listed below, hopefully it can come in handy for someone.

```python
import os
import sys
import itertools
import json

_NONE = object()


class SettingManager:

    def __init__(self):
        # what is current environment?
        # assume default environment to be production
        self.env = os.getenv('ENV', 'prd')

        try:
            # try to read default settings
            self._default = __import__('settings.default', fromlist=['*'])
        except ModuleNotFoundError:
            # if default settings is not found, just assume empty object
            self._default = object()

        try:
            # try to read environment specific setting overrides
            self._env = __import__('settings.{}'.format(self.env), fromlist=['*'])
        except ModuleNotFoundError:
            # if environment specific settings is not found, just assume empty object
            self._env = object()

        # maintain list of other loaded settings
        self._loaded = []

    def load(self, filename, fmt='json'):
        """
        load settings from specified file
        :param filename:
        :param fmt: format of data to read
        :return:
        """
        filename = os.path.abspath(filename)
        if fmt == 'json':
            with open(filename) as f:
                self._loaded.append((filename, json.load(f)))

    def unload(self, filename):
        """
        unload settings that were read from file
        previously by calling load
        :param filename:
        :return:
        """
        filename = os.path.abspath(filename)
        self._loaded = [(f, v) for f, v in self._loaded if f != filename]

    def __getattr__(self, item):
        # empty object that can denote empty
        # or None result. It's better than using
        # None because None can be intended
        # value of a setting
        sentry = object()

        # assume result to be empty
        result = sentry

        # first try to read setting from loaded
        # files.
        # the files are traversed in order
        # so that newly added files
        # may override settings from older ones
        for _, values in self._loaded:
            if item in values:
                # don't break, continue because
                # some newer file may have setting
                # which overrides this file
                result = values[item]

        # try to read setting from environment
        # variable of same name
        # if environment variable is present
        # it can override setting defined
        # in manually loaded files
        result = os.getenv(item, result)

        if result is sentry:
            # if result is still sentry, then specified
            # setting is not in loaded config files
            # nor it is in environment variables
            # try to read setting using default data
            # and environment specific data
            result = getattr(self._env, item, getattr(self._default, item, sentry))
            if result is sentry:
                # if still not found
                # throw error
                raise AttributeError

        # return result
        return result

    def __contains__(self, item):
        try:
            self.__getattr__(item)
            return True
        except AttributeError:
            return False

    def get(self, item, default=_NONE):
        try:
            return self.__getattr__(item)
        except AttributeError:
            if default is not _NONE:
                return default
            raise AttributeError

    def __iter__(self):
        chained = itertools.chain(getattr(self._default, '__dict__', dict()).keys(),
                                  getattr(self._env, '__dict__', dict()).keys())

        for _, values in self._loaded:
            chained = itertools.chain(chained, values.keys())

        return iter(filter(lambda x: not x.startswith('_'), set(chained)))


sys.modules[__name__] = SettingManager()
```