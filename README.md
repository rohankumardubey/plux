localstack-plugin-loader
========================

<p>
  <a href="https://pypi.org/project/localstack-plugin-loader/"><img alt="PyPI Version" src="https://img.shields.io/pypi/v/localstack-plugin-loader?color=blue"></a>
  <a href="https://img.shields.io/pypi/l/localstack-plugin-loader.svg"><img alt="PyPI License" src="https://img.shields.io/pypi/l/localstack-plugin-loader.svg"></a>
  <a href="https://github.com/psf/black"><img alt="Code style: black" src="https://img.shields.io/badge/code%20style-black-000000.svg"></a>
</p>

localstack-plugin-loader is the dynamic code loading framework used in [LocalStack](https://github.com/localstack/localstack).


Overview
--------

The localstack-plugin-loader builds a higher-level plugin mechanism around [Python's entry point mechanism](https://packaging.python.org/specifications/entry-points/).
It provides tools to load plugins from entry points at run time, and to discover entry points from plugins at build time (so you don't have to declare entry points statically in your `setup.py`).

### Core concepts

* `PluginSpec`: describes a `Plugin`. Each plugin has a namespace, a unique name in that namespace, and a `PluginFactory` (something that creates `Plugin` the spec is describing.
  In the simplest case, that can just be the Plugin's class).
* `Plugin`: an object that exposes a `should_load` and `load` method.
  Note that it does not function as a domain object (it does not hold the plugins lifecycle state, like initialized, loaded, etc..., or other metadata of the Plugin)
* `PluginFinder`: finds plugins, either at build time (by scanning the modules using `pkgutil` and `setuptools`) or at run time (reading entrypoints of the distribution using [stevedore](https://docs.openstack.org/stevedore/latest/))
* `PluginManager`: manages the run time lifecycle of a Plugin, which has three states:
  * resolved: the entrypoint pointing to the PluginSpec was imported and the `PluginSpec` instance was created
  * init: the `PluginFactory` of the `PluginSpec` was successfully invoked
  * loaded: the `load` method of the `Plugin` was successfully invoked

### Loading Plugins

At run time, a `PluginManager` uses a `PluginFinder` that in turn uses stevedore to scan the available entrypoints for things that look like a `PluginSpec`.
With `PluginManager.load(name: str)` or `PluginManager.load_all()`, plugins within the namespace that are discoverable in entrypoints can be loaded.
If an error occurs at any state of the lifecycle, the `PluginManager` informs the `PluginLifecycleListener` about it, but continues operating.

### Discovering entrypoints

At build time (e.g., with `python setup.py develop/install/sdist`), a special `PluginFinder` collects anything that can be interpreted as a `PluginSpec`, and creates from it setuptools entrypoints.
In the `setup.py` we can use the `plugin.setuptools.load_entry_points` method to collect a dictionary for the `entry_points` value of `setup()`.

```python
import plugin.setuptools.load_entry_points
setup(
    entry_points=load_entry_points(exclude=("tests", "tests.*",))
)
```

Examples
--------

To build something using the plugin framework, you will first want to introduce a Plugin that does something when it is loaded.
And then, at runtime, you need a component that uses the `PluginManager` to get those plugins.

### One class per plugin

This is the way we went with `LocalstackCliPlugin`. Every plugin class (e.g., `ProCliPlugin`) is essentially a singleton.
This is easy, as the classes are discoverable as plugins.
Simply create a Plugin class with a name and namespace and it will be discovered by the build time `PluginFinder`.

```python

# abstract case (not discovered at build time, missing name)
class CliPlugin(Plugin):
    namespace = "my.plugins.cli"

    def load(self, cli):
        self.attach(cli)

    def attach(self, cli):
        raise NotImplementedError

# discovered at build time (has a namespace, name, and is a Plugin)
class MyCliPlugin(CliPlugin):
    name = "my"

    def attach(self, cli):
        # ... attach commands to cli object

```

now we need a `PluginManager` (which has a generic type) to load the plugins for us:

```python
cli = #... needs to come from somewhere

manager: PluginManager[CliPlugin] = PluginManager("my.plugins.cli", load_args=(cli,))

plugins: List[CliPlugin] = manager.load_all()

# todo: do stuff with the plugins, if you want/need
#  in this example, we simply use the plugin mechanism to run a one-shot function (attach) on a load argument

```

### Re-usable plugins

When you have lots of plugins that are structured in a similar way, we may not want to create a separate Plugin class for each plugin.
Instead we want to use the same `Plugin` class to do the same thing, but use several instances of it.
The `PluginFactory`, and the fact that `PluginSpec` instances defined at module level are discoverable (inpired by [pluggy](https://github.com/pytest-dev/pluggy)), can be used to achieve that.


```python

class ServicePlugin(Plugin):

    def __init__(self, service_name):
      self.service_name = service_name
      self.service = None

    def should_load(self):
        return self.service_name in config.SERVICES:

    def load(self):
        module = importlib.import_module("localstack.services.%s" % self.service_name)
        # suppose we define a convention that each service module has a Service class, like moto's `Backend`
        self.service = module.Service()


def service_plugin_factory(name) -> PluginFactory:
    def create():
        return ServicePlugin(name)

    return create

# discoverable
s3 = PluginSpec("localstack.plugins.services", "s3", service_plugin_factory("s3"))

# discoverable
dynamodb = PluginSpec("localstack.plugins.services", "dynamodb", service_plugin_factory("dynamodb"))

# ... could be simplified with convenience framework code, but the principle will stay the same

```

Then we could use the `PluginManager` to build a Supervisor

```python

class Supervisor:
    manager: PluginManager[ServicePlugin]

    def start(self, service_name):
      plugin = manager.load(service_name)
      service = plugin.service
      service.start()

```

Install
-------

    pip install localstack-plugin-loader

Develop
-------

Create the virtual environment, install dependencies, and run tests

    make venv
    make test

Run the code formatter

    make format

Upload the pypi package using twine

    make upload
