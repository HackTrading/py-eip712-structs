# EIP-712 Structs  [![Build Status](https://travis-ci.org/ajrgrubbs/py-eip712-structs.svg?branch=master)](https://travis-ci.org/ajrgrubbs/py-eip712-structs) [![Coverage Status](https://coveralls.io/repos/github/ajrgrubbs/py-eip712-structs/badge.svg?branch=master)](https://coveralls.io/github/ajrgrubbs/py-eip712-structs?branch=master)

A python interface for simple EIP-712 struct construction.

In this module, a "struct" is structured data as defined in the standard.
It is not the same as the Python Standard Library's struct (e.g., `import struct`).

Read the proposal:<br/>
https://github.com/ethereum/EIPs/blob/master/EIPS/eip-712.md

#### Supported Python Versions
- `3.6`
- `3.7`

## Install
```bash
pip install eip712-structs
```

## Quickstart
Say we want to represent the following struct, convert it to a message and sign it:
```text
struct MyStruct {
    string some_string;
    uint some_number;
}
```

With this module:
```python
# Make a unique domain
from eip712_structs import make_domain
domain = make_domain(name='Some name', version='1.0.0')

# Define your struct type
from eip712_structs import EIP712Struct, String, Uint
class MyStruct(EIP712Struct):
    some_string = String()
    some_number = Uint(256)

# Create an instance with some data
mine = MyStruct(some_string='hello world', some_number=1234)

# Into a message dict (serializable to JSON) - domain required
my_msg = mine.to_message(domain)

# Into signable bytes - domain required
my_bytes = mine.signable_bytes(domain)
```

#### Dynamic construction
Attributes may be added dynamically as well. This may be necessary if you
want to use a reserved keyword like `from`.

```python
from eip712_structs import EIP712Struct, Address
class Message(EIP712Struct):
    pass

Message.to = Address()
setattr(Message, 'from', Address())
```

#### The domain separator
EIP-712 specifies a domain struct, to differentiate between identical structs that may be unrelated.
A helper method exists for this purpose.
All values to the `make_domain()`
function are optional - but at least one must be defined. If omitted, the resulting
domain struct's definition leaves out the parameter entirely.

The full signature: <br/>
`make_domain(name: string, version: string, chainId: uint256, verifyingContract: address, salt: bytes32)`

##### Setting a default domain
Constantly providing the same domain can be cumbersome. You can optionally set a default, and then forget it.
It is automatically used by `.to_message()` and `.signable_bytes()`

```python
import eip712_structs

foo = SomeStruct()

my_domain = eip712_structs.make_domain(name='hello world')
eip712_structs.default_domain = my_domain

assert foo.to_message() == foo.to_message(my_domain)
assert foo.signable_bytes() == foo.signable_bytes(my_domain)
```

## Member Types

### Basic types
EIP712's basic types map directly to solidity types.

```python
from eip712_structs import Address, Boolean, Bytes, Int, String, Uint

Address()  # Solidity's 'address'
Boolean()  # 'bool'
Bytes()    # 'bytes'
Bytes(N)   # 'bytesN' - N must be an int from 1 through 32
Int(N)     # 'intN' - N must be a multiple of 8, from 8 to 256
String()   # 'string'
Uint(N)    # 'uintN' - N must be a multiple of 8, from 8 to 256
```

Use like:
```python
from eip712_structs import EIP712Struct, Address, Bytes

class Foo(EIP712Struct):
    member_name_0 = Address()
    member_name_1 = Bytes(5)
    # ...etc
```

### Struct references
In addition to holding basic types, EIP712 structs may also hold other structs!
Usage is almost the same - the difference is you don't "instantiate" the class.

Example:
```python
from eip712_structs import EIP712Struct, String

class Dog(EIP712Struct):
    name = String()
    breed = String()

class Person(EIP712Struct):
    name = String()
    dog = Dog  # Take note - no parentheses!

# Dog "stands alone"
Dog.encode_type()     # Dog(string name,string breed)

# But Person knows how to include Dog
Person.encode_type()  # Person(string name,Dog dog)Dog(string name,string breed)
```

Instantiating the structs with nested values may be done a couple different ways:

```python
# Method one: set it to a struct
dog = Dog(name='Mochi', breed='Corgi')
person = Person(name='E.M.', dog=dog)

# Method two: set it to a dict - the underlying struct is built for you
person = Person(
    name='E.M.',
    dog={
        'name': 'Mochi',
        'breed': 'Corgi',
    }
)
```

### Arrays
Arrays are also supported for the standard.

```python
array_member = Array(<item_type>[, <optional_length>])
```

- `<item_type>` - The basic type or struct that will live in the array
- `<optional_length>` - If given, the array is set to that length.

For example:
```python
dynamic_array = Array(String())      # String[] dynamic_array
static_array  = Array(String(), 10)  # String[10] static_array
struct_array = Array(MyStruct, 10)   # MyStruct[10] - again, don't instantiate structs like the basic types
```

## Development
Contributions always welcome.

Install dependencies:
- `pip install -r requirements.txt && pip install -r test_requirements.txt`

Run tests:
- `python setup.py test`
- Some tests expect an active local ganache chain. Compile contracts and start the chain using docker:
    - `docker-compose up -d`
    - If the chain is not detected, then they are skipped.
