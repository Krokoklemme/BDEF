# Binary Data Exchange Format

Early draft of a general-purpose binary data format

## Goals

The main goal for this format are:  
* Low storage sizes
* Low parsing times
* Type safety

This is achieved by using a simple and compact binary format instead of plain-text

## File Format

The format consists of 4 components: A header, a string pool, type definitions and the actual data.

Each one is described in as much detail as possible, though if something still happens to be unclear/ambiguous just send me a message :)

Unless otherwise specified, there's no padding between adjacent sections of a file and all data is stored in little endian

### File Header

The file header is unsurprisingly the simplest part:

1. Byte signature
2. Version number
3. Settings

#### Byte Signature

The byte signature consists of the values 0x02 0x04 0x05 0x06 (B = 2. letter, E = 4. letter and so on)

#### Version Number

This is a 2-byte value with each of them representing the major and minor version respectively (major first, minor second)

#### Settings

The settings part is a 2-byte wide bitflag section stored in big endian (meaning the MSB is in the leading byte)  
With the order going from MSB to LSB, the individual bits represent the following things:

1. If set, treat strings as case-sensitive
2. _RESERVED_
3. _RESERVED_
4. _RESERVED_
5. _RESERVED_
6. _RESERVED_
7. _RESERVED_
8. _RESERVED_
9. _RESERVED_
10. _RESERVED_
11. _RESERVED_
12. _RESERVED_
13. _RESERVED_
14. _RESERVED_
15. _RESERVED_
16. _RESERVED_

Reserved bits should always be 0

#### Example

File header example

```
0x02 0x04 0x05 0x06 0x01 0x00 0x80 0x00
```

**Breakdown:**  
`0x02 0x04 0x05 0x06` byte signature  
`0x01 0x00` Version number (0x01 -> 1 Major, 0x00 -> 0 Minor)  
`0x80 0x00` Settings (10000000 -> First bit set, strings are case-sensitive)

### String Pool

The string pool starts with an `Int`, indicating the number of total strings.  
Each stored string starts with another `Int`, which holds the total number of
bytes in the following UTF-8 encoded byte sequence, excluding a potential
terminating character.

Indices in the string pool start at 0 and are represented by a 2-byte integer

#### Example

An example of a BDEF string pool
```
0x03 0x00 0x00 0x00 0x05 0x00 0x00 0x00
0x48 0x65 0x6C 0x6C 0x6F 0x08 0x00 0x00
0x00 0xD7 0xA9 0xD7 0x9C 0xD7 0x95 0xD7
0x9D 0x0F 0x00 0x00 0x00 0xE3 0x81 0x93
0xE3 0x82 0x93 0xE3 0x81 0xAB 0xE3 0x81
0xA1 0xE3 0x81 0xAF
```

**Breakdown:**  
`0x03 0x00 0x00 0x00` The size of the string pool (3 in this example)  
`0x05 0x00 0x00 0x00` Length of the first string (in bytes, no 0-byte)  
`0x48 0x65 0x6C 0x6C 0x6F` UTF-8 encoded "Hello"  
`0x08 0x00 0x00 0x00` Length of the second string  
`0xD7 0xA9 0xD7 0x9C 0xD7 0x95 0xD7 0x9D` UTF-8 encoded שלום ("Hello" in Hebrew)  
`0x0F 0x00 0x00 0x00` Length of the third string  
`0xE3 0x81 0x93 0xE3 0x82 0x93 0xE3 0x81 0xAB 0xE3 0x81 0xA1 0xE3 0x81 0xAF` UTF-8 encoded こんにちは ("Hello" in Japanese)  

This example pool would equate to the following JSON

```json
[
  "Hello",
  "שלום",
  "こんにちは"
]
```

### Type definitions

Besides the string pool, there's also a pool of defined types.
It's used to eliminate redundancies within the storage of data objects
and to directly map objects defined in the BDEF file to a type in the
language environment used.

A type definition starts with an index in the string pool, denoting the
name of the type, followed by a 2-byte value representing the total number
of properties of the type, which in turn is followed by the property definitions

Properties consist of two 2-byte values, where the first value is a string pool
index (the property's name) and the second one being a type index.

Similar to the string pool, type indices are 2-byte integers and start at 0

#### Example

Assuming we have the string pool

```
[0] = "Person"
[1] = "Name"
[2] = "Age"
```

then the definition for the `Person`-type would look like this

```
0x00 0x00 0x02 0x00 0x01 0x00 0x05 0x00
0x02 0x00 0x02 0x00
```

**Breakdown:**  
`0x00 0x00` String index for the type name ("Person" in this case)  
`0x02 0x00` Amount of properties (2, name and age)  
`0x01 0x00` String index naming the first property ("Name")  
`0x05 0x00` Type index of the `Name` property (a `String` in this case)  
`0x02 0x00` String index of the second property ("Age")  
`0x02 0x00` Type of `Age` (an `Int`)  

This type definition would be similar to this JSON

```json
{
  "Name": "Person"
  "Properties": [
    {
      "Name": "Name",
      "Type": 5
    },
    {
      "Name": "Age",
      "Type": 2
    }
  ]
}
```

#### Built-in types

_TI = Type Index; S = Size in bytes_

* **Object** (TI = 0, S = variable; at least 2)  
   Defines an arbitrary top-level object, can also be used to define a type "inline"

* **Byte** (TI = 1, S = 1)  
   A single byte. Could be used for logical values (`true`/`false`) or to represent a bytearray (to store an image, for example)

* **Int** (TI = 2, S = 4)  
   Ordinary 4-byte integer

* **Long** (TI = 3, S = 8)  
   8-byte integer for edge cases, where 4-bytes aren't sufficient. In most cases an `Int` is appropriate though

* **Real** (TI = 4, S = 8)  
   IEEE 754-2008 binary64 floating-point value (It's a `double`)

* **String** (TI = 5, S = 2)  
   A sequence of UTF-8 encoded characters

* **Sequence** (TI = 6, S = variable; at least 6)  
   A continuous collection of a specified type (Maps to any non-keyed container e.g. an `Array` or a `Queue`)

### Data Pool

The data pool holds all values stored in the current document.  
Unlike the previous two, the this pool doesn't announce the number
of elements it holds (it simply occupies the rest of the file).

A data object has 3 parts:  
1. The familiar string index for the type
2. A type index
3. Its value(s)

Values are defined in order of definition of the objects type

Stored objects have the structure `Name -> Type -> Value` and are directly adjacent to each other

#### Example

**String pool:**
```
[0] = "My Int"
[1] = "My String"
[2] = "I'm some random text"
```

So an `Int` and a `String` would look like this
```
0x00 0x00 0x02 0x00 0x12 0x23 0x45 0x67
0x01 0x00 0x05 0x00 0x02 0x00
```

**Breakdown:**  
`0x00 0x00` Name of first object ("My Int")  
`0x02 0x00` `Int` type index  
`0x12 0x23 0x45 0x67` Value 19088743  
`0x01 0x00` Name of second object ("My String")  
`0x05 0x00` `String` type index  
`0x02 0x00` Value is a string pool index ("I'm some random text")

The equivalent JSON
```json
{
  "My Int": 19088743,
  "My String": "I'm some random text"
}
```

#### Objects

The `Object` is an exception to the usual structure of data objects.

If a value has this type, it's structured in-place i.e. the value's type is
defined inline. It's similar to a normal type definition, except that the
type isn't named (It's practically a singleton).

A value of this type is made up of a 2-byte integer defining the number of
properties it has, followed by their further definitions and finished by their
values in order of definition.

##### Example

Say we want to encapsulate a server configuration, then we'd do it like this

**String pool:**
```
[0] = "Config"
[1] = "Address"
[2] = "127.0.0.1"
[3] = "MaxConnections"
[4] = "RunOnStartup"
```

```
0x00 0x00 0x00 0x00 0x03 0x00 0x01 0x00 
0x05 0x00 0x03 0x00 0x02 0x00 0x04 0x00 
0x01 0x00 0x02 0x00 0x29 0x23 0x00 0x00 
0x01 
```

**Breakdown:**  
`0x00 0x00` Name of the data object ("Config")  
`0x00 0x00` Type index of `Object`  
`0x03 0x00` Number of properties (3 here)  
`0x01 0x00` Property #1 Name ("Address")  
`0x05 0x00` Property #1 Type (`String`)  
`0x03 0x00` Property #2 Name ("MaxConnections")  
`0x02 0x00` Property #2 Type (`Int`)  
`0x04 0x00` Property #3 Name ("RunOnStartup")  
`0x01 0x00` Property #3 Type (`Byte`, used as a boolean here)  
`0x02 0x00` Value of the first property (`Address` = "127.0.0.1")  
`0x29 0x23 0x00 0x00` Value of the second property (`MaxConnections` = 9001)  
`0x01` Value of the third property (`RunOnStartup` = 1/`true`)

This example is equal to this JSON

```json
{
  "Config": {
    "Address": "127.0.0.1",
    "MaxConnections": 9001,
    "RunOnStartup": true
  }
}
```

#### Sequences

Sequences are another exception in that they're allowed to have multiple values

They're defined in the following style
```
Name -> Sequence Type -> Length -> Type -> Value -> ...
```

#### Example

**String Pool:**

```
[0] = "My Array"
[1] = "I'm"
[2] = "an"
[3] = "Array"
```

A `String` sequence

```
0x00 0x00 0x06 0x00 0x03 0x00 0x00 0x00
0x05 0x00 0x01 0x00 0x02 0x00 0x03 0x00
```

**Breakdown:**  
`0x00 0x00` Name of the `Sequence` ("My Array")  
`0x06 0x00` `Sequence` type index  
`0x03 0x00 0x00 0x00` Size of the sequence (3 in this case)  
`0x05 0x00` `String` type index, defining the type of the sequence  
`0x01 0x00` first value (index of "I'm")  
`0x02 0x00` second value (index of "an")  
`0x03 0x00` third value (index of "Array")

The opposite JSON would look like this
```json
{
  "My Array": [
    "I'm",
    "an",
    "Array"
  ]
}
```


## Implementations

If you have an implementation and want it to be listed here, either message me or open up an issue
