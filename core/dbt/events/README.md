# Events Module
The Events module is responsible for communicating internal dbt structures into a consumable interface. Because the "event" classes are based entirely on protobuf definitions, the interface is really clearly defined, whether or not protobufs are used to consume it. We use Betterproto for compiling the protobuf message definitions into Python classes.

# Using the Events Module
The event module provides types that represent what is happening in dbt in `events.types`. These types are intended to represent an exhaustive list of all things happening within dbt that will need to be logged, streamed, or printed. To fire an event, `events.functions::fire_event` is the entry point to the module from everywhere in dbt.

# Logging
When events are processed via `fire_event`, nearly everything is logged. Whether or not the user has enabled the debug flag, all debug messages are still logged to the file. However, some events are particularly time consuming to construct because they return a huge amount of data. Today, the only messages in this category are cache events and are only logged if the `--log-cache-events` flag is on. This is important because these messages should not be created unless they are going to be logged, because they cause a noticable performance degredation. These events use a "fire_event_if" functions.

# Adding a New Event
New events need to have a proto message definition created in core/dbt/events/types.proto. Every message must include EventInfo as the first field, named "info" and numbered 1. To update the proto_types.py file, in the core/dbt/events directory: ```protoc --python_betterproto_out . types.proto```

A matching class needs to be created in the core/dbt/events/types.py file, which will have two superclasses, the "Level" mixin and the generated class from proto_types.py. These classes will also generally have two methods, a "code" method that returns the event code, and a "message" method that is used to construct the "msg" from the event fields. In addition the "Level" mixin will provide a "level_tag" method to set the level (which can also be overridden using the "info" convenience function from functions.py)

Note that no attributes can exist in these event classes except for fields defined in the protobuf definitions, because the betterproto metaclass will throw an error. Betterproto provides a to_dict() method to convert the generated classes to a dictionary and from that to json. However some attributes will successfully convert to dictionaries but not to serialized protobufs, so we need to test both output formats.


## Required for Every Event

- a method `code`, that's unique across events
- assign a log level by using the Level mixin: `DebugLevel`, `InfoLevel`, `WarnLevel`, or `ErrorLevel`
- a message()

Example
```
@dataclass
class PartialParsingDeletedExposure(DebugLevel, pt.PartialParsingDeletedExposure):
    def code(self):
        return "I049"

    def message(self) -> str:
        return f"Partial parsing: deleted exposure {self.unique_id}"

```


# Adapter Maintainers
To integrate existing log messages from adapters, you likely have a line of code like this in your adapter already:
```python
from dbt.logger import GLOBAL_LOGGER as logger
```

Simply change it to these two lines with your adapter's database name, and all your existing call sites will now use the new system for v1.0:
```python
from dbt.events import AdapterLogger
logger = AdapterLogger("<database name>")
# e.g. AdapterLogger("Snowflake")
```

## Compiling types.proto

After adding a new message in types.proto, in the core/dbt/events directory: ```protoc --python_betterproto_out . types.proto```
