# bytechomp

> *A pure python, declarative custom binary protocol parser using dataclasses and type hinting.*

`bytechomp` leverages Python's type hinting system at runtime to build binary protocol parsing schemas from dataclass implementations. Deserialization of the binary data is now abstracted away by `bytechomp`, leaving you to work in the land of typed and structured data.

**Features:**
- [x] Pure Python
- [x] Zero Dependencies
- [x] Uses native type-hinting & dataclasses
- [x] Supports lower-precision numerics
- [x] Supports `bytes` and `str` fields of known length
- [x] Supports `list` types for repeated, continuous fields of known length
- [x] Supports nested structures

## Installation

`bytechomp` is a small, pure python library with zero dependencies. It can be installed via PyPI:

```
pip install bytechomp
```

or Git for the latest unreleased code:

```
pip install https://github.com/AndrewSpittlemeister/bytechomp.git@main
```

## Example Usage

```python
from typing import Annotated
from dataclasses import dataclass

from bytechomp import Reader
from bytechomp.datatypes import U16, F32


@dataclass
class Header:
    timestamp: float  # native datatypes can be used when assuming full precision
    message_count: int  # similarly with 64-bit integers
    message_identity: U16  # custom datatypes are available and will be cast to native when deserialized


@dataclass
class Body:
    unique_id: Annotated[bytes, 5]  # use of typing.Annotated to denote length
    name: Annotated[str, 64]  # string values are decoded from byte streams too 
    balance: F32


@dataclass
class Message:
    header: Header  # nested data structures are allowed
    body: Body


@dataclass
class MessageBundle:
    messages: Annotated[list[Message], 8]  # so are lists of data structures!


def main() -> None:
    # build Reader object using the MessageBundle class as its generic argument
    reader = Reader[MessageBundle]().allocate()

    with open("my-binary-data-stream.dat", "rb") as fp:
        while (data := fp.read(4)):
            # feed stream into the reader
            reader.feed(data)

            # check if the structure has been saturated with enough data
            if reader.is_complete():
                # parse the stream and create your typed data structure!
                msg_bundle = reader.build()
                print(msg_bundle)
```

### Reader API


## How does this work?

While a binary stream is usually represented as a flat, continuous data, `bytechomp` can be used as a structural abstraction over this data. Therefore, if there was a message with the following structure for a message called `UserState`:

| Field | Type | Description |
| ----- | ---- | ----------- |
| `user_id` | uint64 | user's unique identity |
| `balance` | float32 | user's balance |

The resulting translation to a dataclass would be the following:

```python
from bytechomp import Reader, dataclass  # using re-export from within bytechomp
from bytechomp.datatypes import F32

@dataclass
class UserState:
    user_id: int
    balance: F32
```

When parsing messages that contain other messages, you will need to be aware of how the embedded messages are contained and how the resulting memory layout will look for the container message as whole. Since the container message is still represented as one set of continuous bytes, nested classes in bytechomp are constructed using a depth first search of the contained fields in nested structures to build out a flattened parsing pattern for Python's `struct` module.

Consider the following structures:

```python
from bytechomp import Reader, dataclass, Annotated  # using re-export from within bytechomp
from bytechomp.datatypes import F32

@dataclass
class UserState:
    user_id: int
    balance: F32

@dataclass
class Transaction:
    amount: F32
    sender: int
    receiver: int

@dataclass
class User:
    user_state: UserState
    recent_transactions: Annotated[list[Transaction], 3]
```

The `User` message would correspond to the following memory layout:

```
uint64, float32, float32, int64, int64, float32, int64, int64, float32, int64, int64
```

## Additional Notes

This package is based on a mostly undocumented feature in standard implementation of CPython. This is the ability to inspect the type information generic parameters via the `self.__orig_class__.__args__` structures. The information in this structure is only populated after initialization (hence the need for the `allocate()` method when instantiated a `Reader` object). Should this behavior change in future versions of Python, `bytechomp` will adapt accordingly. For now, it will stay away from passing a type object as a argument to initialization because that just seems hacky.