# Python CLIF



To describe a C++ API in Python CLIF uses a modified
[PYTD](https://github.com/google/pytypedecl) language described below.


## Example

CLIF has C++ API description in the .clif file:

```python
# 1. Load FileStat wrapper class from another CLIF wrapper.
from "file/base/python/filestat.h" import *  # FileStat
# 2. Load Status from hand-written C++ CLIF extension library.
from "util/task/python/clif.h" import *  # Status
# 3. Load Options protobuf from generated C++ CLIF extension library.
from  "file/base/options_pyclif.h" import *
# 4. Load pure-Python postprocessor function for Status
from clif.python.postproc import DropOkStatus

from "file/base/filesystem.h":
  namespace `file`:
    def ForEachMatch(pattern: str, options: Options,
                     match_handler: (filename:str, fs:FileStat)->bool
                    ) -> Status:
      return DropOkStatus(...)
    # define other API items here (if needed)
```

Line 1 gets a class wrapped by another CLIF module.
Line 2 gets a [custom wrap](/util/task/python/g3doc/status.md) for `Status` and
`StatusOr`.
Line 3 gets a wrapped option.proto (generated by `pyclif_proto_library` BUILD
rule).

**Note:** Callback signature above matches
`std::function<bool (StringPiece, file:Stats)>`.


## API specification language

From that example we see that .clif file has 2 sections:

 * preparation (import symbols we'll need in the next section),
 * API description (tell which C++ API we'll need from a particular header).

**Preparation** specifies which CLIF extension libraries are needed and
what C++ library we are wrapping.
It can have [c header import](#cimport), [python import](#pyimport),
[namespace](#namespace) and [use](#use) statements.

**API description** starts with _from_ statement that points to the C++ header
file we wrap and has an indented block describing the API.
That block might have the following statements:

 * [def](#def)
 * [const](#const)
 * [enum](#enum)
 * [class](#class)
 * [staticmethods](#staticmethods)

which are described below.


### c_header import statement {#cimport}

The _c header import_ statement makes types wrapped by another CLIF rule or by
a C++ CLIF extension library available to use in this .clif file.
Such library can be written [by hand](ext.md) or generated by a tool (like CLIF
protobuf wrapper - it generates a cc_library CLIF extension.)

```python
from "cpp/include/path/to/aCLIF/extension/library.h" import *
```

**Note** that _c header import_ requires a double-quoted string exactly as the
C++ `#include` directive.

Use _c header import_ statement to inform CLIF about wrapped C++ types that
needs to be available in the module being wrapped.

If you don't want to pollute .clif namespace with all names from that header,
you can prefix imported names with a variant of include statement:

```python
from "some/header.h" import * as prefix_name
```

Now all CLIF types defined in the `header.h` (with
``CLIF use `ctype` as clif_type``) will be available as `prefix_name.clif_type`.


### python import statement {#pyimport}

The _python import_ statement is a normal Python import to make a library
symbol available within the .clif file. Only a single symbol import allowed
(not a module). All imports must be absolute.

```python
from path.to.project.library.module import SomeClassOrFunction
```

This statement is typically used to load a Python [postprocessing function]
(#postprocessing).


### from statement {#from}

The _from_ statement tells CLIF what library file to wrap.
This statement allows top-level API name lookup in any namespace in the
specified file.

```python
from "cpp/include/path/to/some/library.h":
  # API description statements
```

### namespace statement {#namespace}

The _namespace_ statement tells CLIF what C++ namespace to use (backquotes
are required around the C++ name).
That namespace must be declared in the _from_'d file.
This statement limits top-level API name lookup to the specified namespace.

```python
from "cpp/include/path/to/some/library.h":
  namespace `my::namespace`:
    def Name()  # API description statements
```

WARNING: Namespace statements can't be nested.


### def statement {#def}

The _def_ statement describes a C++ function (or member function).

```python
def NAME ( INPUT_PARAMETERS ) OUTPUT_PARAMETERS
```

It has 3 main components: name, input parameters and output parameters.

**NAME** can be a simple alphanumeric name when we want it to be the same in C++
and Python. In some cases we want or need to rename the C++ name to have a
different name in Python wrapper. In those cases rename construct can be used:

```python
`cplusplus_name` as python_name
```

For example `` `size` as __len__``[^len] or `` `pass` as pass_``.
Such renaming can occur everywhere a NAME is used.

[^len]: When exposing a C++ function as `__len__` make sure it only returns a
        non-negative numbers or Python will raise a `SystemError`.
        

**INPUT_PARAMETERS** describes values to be converted from Python and passed to
the C++ function. It is a (potentially empty) comma-separated list of
`name:type` pairs, ie. `x:int, descriptive_name:str`. Both `name` and `type`
are required (Only `self` in class methods has no type.)
For a type you use a Python standard type. Python containers should also be
typed (like `list<int>` or `dict<bytes, int>`).

#### Default parameters

If C++ has a default argument (ie. with `= value` clause), it can also be
optional in PYTD. Just add `=default` to its `name:type` specification.

**OUTPUT_PARAMETERS** are more complex:

  * can be empty, meaning no return values (Python wrapper returns None)
  * can be a single value in the form `-> type`, or
  * can be multiple return values: `-> (name1:type1, name2:type2, ...)`
    like input parameters.

By Google [convention]
(https://google.github.io/styleguide/cppguide.html#Function_Parameter_Ordering)
C++ signature should have all input parameters before any
output parameter(s). The first output parameter is the function return value and
others are listed after inputs as C++ `TYPE*` (pointer to output type). CLIF
does not allow you to violate those conventions. To circumvent that restriction
write a helper C++ function and wrap it instead.

For example:

C++ function  | described as
------------- | ------------
void F()      | def F()
int F()       | def F() -> int
void F(int)   | def F(name_is_mandatory: int)
int F(int)    | def F(name_is_mandatory: int) -> int
int F(string*)| def F() -> (code: int, message: str)

#### Pointers, references and object ownership

CLIF wraps of C++ functions with output parameters or return values of type
`std::unique_ptr` transfer object ownership to Python. Wraps of
`std::unique_ptr` input parameters transfer ownership to C++.

Raw pointers are always assumed to be borrowed; if a C++ API treats a raw
pointer as an ownership transfer, a C++ wrapping function or overload using
`std::unique_ptr` can be added to make the transfer explicit to CLIF.

#### Postprocessing {#postprocessing}

Often C/C++ APIs return a status as one return value. Python users prefer to
not see a good status at all and get an exception on a bad status. To get that
behavior, CLIF supports Python postprocessor functions that will take return
value(s) from C++ and transform them.

The standard CLIF library comes with the following postprocessor functions:

 * `ValueErrorOnFalse`
   takes first return value as bool, drops it from output if True or raise a
   ValueError if it's False.

 * `chr`
   is Python built-in function useful to convert int/uint8 (from C++ char) to a
   Python 1-character string.

To use a postprocessor function you must first import it[^chr] with a [python import]
(#pyimport) statement but remember to import the proper Python name, not just
the module. And use the extended `def` syntax as shown below:

[^chr]: Except `chr` that is already 'imported' by CLIF.

```python
def NAME ( INPUT_PARAMETERS ) OUTPUT_PARAMETERS:
  return PostProcessorFunction(...)
```

where `...` are three dots verbatim for all OUTPUT_PARAMETERS to be passed
as args to PostProcessorFunction.

#### Asynchronous execution

To release the GIL during a C++ call mark the function with `@async` decorator.

If you ever need to release the GIL during C++ destructor run (this is not
common), use class decorator `@async__del__`.

#### Implementing (virtual) methods in Python

To allow a Python implementation of a derived class to be called from C++ (via a
pointer to the base class) mark the function with a `@virtual` decorator.

Do not decorate C++ virtual methods with @virtual unless you need to implement
them in Python.

#### Context manager

To use a wrapped C++ class as a Python [context manager](https://docs.python.org/2/library/stdtypes.html#typecontextmanager),
some methods must be wrapped as `__enter__` and `__exit__`.
However Python has a different calling convention. To help wrap such cases use
CLIF method decorators to force the Python API:

  * `@__enter__` to call the wrapped method on `__enter__` and return
    `self` as the context manager instance, and
  * `@__exit__` to take the required [PEP-343]
    (https://www.python.org/dev/peps/pep-0343/) (type, value, traceback) args
    on `__exit__`, call the wrapped method with no arguments, and return None.

However if the C++ method provides the needed API it can be simply renamed:

    def `c_implementation_of_exit` as __exit__(self,
        type: object, value: object, trace: object) -> bool


### const statement {#const}

The _const_ statement describes a C++ constant (const or constexpr).

```c++
const NAME: TYPE
```

It also makes sense to rename the constant to make it Python-style conformant:

```c++
const `kNumTries` as NUM_TRIES: int
```

### enum statement {#enum}

The _enum_ statement describes a C++ enum or enum class. This form will take all
enum values under the same names as they are known to C++.

```c++
enum NAME
```

It also makes sense to rename enum values to match expected Python style:

```c++
enum NAME with:
  `kDefault` as DEFAULT
  `kOptionOne` as OPTION_ONE
```

C++ enums will be presented as Python `Enum` or `IntEnum`[^enum] classes from
the standard `enum` module [backported to Python 2.7][pypi].

[^enum]: C++ 11 `class enum` converted to `Enum`, old-style `enum` to `IntEnum`.
[pypi]: https://pypi.python.org/pypi/enum34

### class statement {#class}

The _class_ statement describes a C++ struct or class. It must have
an indented block describing what class members are wrapped. That block has all
the statements that [from](#from) block has and a [var](#var) statement
for member variables.

Each member method should have a specific first argument:

  * `self` for regular C++ member functions
  * `cls` for static C++ member functions

The first argument (self/cls) should not have any type as the type is implicit
(it's the class that function is a member of).

Also static member functions should have `@classmethod` decorator or moved to
module level with the [_staticmethods_](#staticmethods) statement.

```python
class MyClass:
  def __init__(self, param: int, another_param: float)
  def Method(self) -> dict<int, int>
  @classmethod
  def StaticMethod(cls, param: int)
```

TIP: Better expose class static member functions as Python module-level
functions.

#### Inheritance

CLIF inheritance specification need not follow the C++ inheritance relationship.
Only specify the base class if it is important for Python API. CLIF is capable
to figure out C++ inheritance details even if the `.clif` file does not
specifies them.

If the C++ class has no parent, no parent should be in the CLIF specification.
If the C++ class has a parent but it's of no interest to a Python user, the
parent also should be omitted and relevant parent methods should be mentioned
in the child CLIF specification.

```c++
class Parent {
 public:
  void Something() = 0;
  void SomethingInteresting();
};

class Child : public Parent {
 public:
  void Useful();
};
```

A CLIF specification for that might look like the following.

```python
class Child:
  def SomethingInteresting(self)
  def Useful(self)
```

If the parent C++ class is already wrapped in CLIF specification, mention it as
you normally do with a parent Python class (`class Child(Parent):`).

Multiple inheritance in CLIF declaration is prohibited, but the C++ class
being wrapped may have multiple parents according to the
`static_cast<>`when converting to/from Python.


### var statement {#var}

The _var_ statement describes a C++ public member variable.

**Note** that _var_ is the only statement that has no keyword.

```
NAME: TYPE
```

Python always gets a copy of a C++ variable on each attribute access.
This is counterintuitive to how most people think about Python as such an
attribute access is not a simple reference.
Updating that copy without reassigning the variable back onto the class will
have no effect.
To remind the user about the copy instead of letting them incorrectly assume
that an attribute access is a reference, you might want to use
`@getter` (and `@setter`) function decorators to declare Python
methods to get (and set) the C++ variable instead of exposing an attribute.
That can be thought of as the reverse of the `property` feature seen below.
Both the getter and setter must use the C++ variable name as the C++ name of
the function.

```python
  class Stat:
    class Options:
      length: int
    @getter
    def `opt` as get_options(self) -> Options
    @setter
    def `opt` as set_options(self, o: Options)
```

If C++ class has getters and setters, consider using them as Python property
rather than calling getters and setters as functions from Python. Direct access
to instance variables is more Pythonic and makes programs more readable.

```
NAME: TYPE = property(`getter`, `setter`)
```

The _getter_ is a C++ function returning _TYPE_ (`TYPE getter();`) and
the _setter_ is a C++ function taking _TYPE_ (`void setter(TYPE);`).
To have a read-only property just use only the getter.

The _var_ statement is most useful in describing plain C structs.
If we have a struct with mostly data members
,it can be described as

```python
from "file/base/fileproperties_pyclif.h" import *

from "file/base/filestat.h":

  class FileStat:
    length: int
    mtime:  `time_t` as int
    # ...
    properties: FileProperties = property(`file_properties`)
    def IsDirectory(self) -> bool
    def Clear(self)
```

### staticmethods statement {#staticmethods}

The _staticmethods_ statement facilitates wrapping class static member
functions. It has a nested block that can only contain _def_ statements. Like
the _namespace_ statement, this statement limits function search inside the
named class.

```python
from "some/file.h":
  staticmethods from `Foo`:
    def Bar()
    def Baz()
```

In that example `Foo::Bar` and `Foo::Baz` must be static members of class `Foo`
and will be wrapped as functions file.Bar and file.Baz.

TIP: The class name can be fully qualified.

### pass statement {#pass}

The _pass_ statement allows you to wrap a derived C++ class that adds nothing
to the interface of the base class.

```python
from "some/file.h":

  class Base:
    def SomeApi(self)

  class Derived(Base):
    pass
```

In that example Derived has the same API as Base, ie. SomeApi().


### capsule statement {#capsule}

The _capsule_ statement declares a raw pointer to the C++ type to be stored in a
Python [capsule](https://docs.python.org/3/c-api/capsule.html) object.

```python
from "some/file.h":

  capsule MyData  # stores MyData*
```

MyData* returned from C++ can be passed around to another C++ function as an
input parameter.

Neither Python nor CLIF make any assumptions or give any guarantees about
[the lifetime or validity of] that pointer.

This statement is rarely needed.


### use statement {#use}

The _use_ statement reassigns a default C++ type for a given Python type:

```
use `std::string` as str
```

This statement is rarely needed.
See more on types below.


## Type correspondence

CLIF uses Python types in API descriptions. It's a CLIF job to find a proper C++
type for each signature.

CLIF is currently limited to a single C++ type per Python type. We apologize
for the inconvenience. This limitation will be lifted in future versions
.
To change what C++ type CLIF will use for a given Python type the user
can either

  * specify an exact C++ type you want to use (ie. `` -> `size_t` as int``) or
  * change the default with the [use](#use) statement


### Predefined types

CLIF knows some basic types (predefined in `clif/python/types.h`):

C++ type        | CLIF type[^type]
--------------- | ------------
int             | int
string          | bytes _or_ str
bool            | bool
double          | float
vector<>        | list<>
pair<>          | tuple<>
unordered_set<> | set<>
unordered_map<> | dict<>
PyObject*       | object 

[^type]: CLIF type named after corresponding Python type

CLIF also knows how to handle several compatible types

Python type | Compatible input C++ types
------------|---------------------------
float | float 
dict  | map
set   | set
list  | list, array, stack, deque, queue, priority_queue

To use a compatible type mention it explicitly in the declaration, e.g.

```python
def fsum(array: `std::list` as list<float>) -> `float` as float
```

for `float fsum(std::list<float>);` C++ function.

NOTE: CLIF will reject unknown types and produce an error.


### Unicode

Please note that we want the C++ API to be explicit and while C++ does not
distinguish between bytes and unicode, Python does. It means that Python .clif
files must specify what exact type (bytes or unicode) the C++ code expects or
produces.

However, CLIF

always takes Python unicode and implicitly encodes it using UTF-8 for C++.
To get unicode back to Python 2, use `unicode` as the return datatype.
In Python 3, `str` gets converted to unicode automatically.

That can be summarized as below.

CLIF type | On input | On output CLIF returns
----------|----------|-----------------------
bytes     | (*)      | bytes
str       | (*)      | native str
unicode   | (*)      | unicode

(*) CLIF will take bytes or unicode Python object and pass [UTF-8 encoded] data
to C++.

#### Encoding

UTF-8 encoding assumed on C++ side.
