# Type System

The GraphQL Type system describes the capabilities of a GraphQL server and is
used to determine if a query is valid. The type system also describes the
input types of query variables to determine if values provided at runtime
are valid.

A GraphQL server's capabilities are referred to as that server's "schema".
A schema is defined in terms of the types and directives it supports.

A given GraphQL schema must itself be internally valid. This section describes
the rules for this validation process where relevant.

A GraphQL schema is represented by a root type for each kind of operation:
query and mutation; this determines the place in the type system where those
operations begin.

All types within a GraphQL schema must have unique names. No two provided types
may have the same name. No provided type may have a name which conflicts with
any built in types (including Scalar and Introspection types).

All directives within a GraphQL schema must have unique names. A directive
and a type may share the same name, since there is no ambiguity between them.

## Types

The fundamental unit of any GraphQL Schema is the type. There are eight kinds
of types in GraphQL.

The most basic type is a `Scalar`. A scalar represents a primitive value, like
a string or an integer. Oftentimes, the possible responses for a scalar field
are enumerable. GraphQL offers an `Enum` type in those cases, where the type
specifies the space of valid responses.

Scalars and Enums form the leaves in response trees; the intermediate levels are
`Object` types, which define a set of fields, where each field is another
type in the system, allowing the definition of arbitrary type hierarchies.

GraphQL supports two abstract types: interfaces and unions.

An `Interface` defines a list of fields; `Object` types that implement that
interface are guaranteed to implement those fields. Whenever the type system
claims it will return an interface, it will return a valid implementating type.

A `Union` defines a list of possible types; similar to interfaces, whenever the
type system claims a union will be returned, one of the possible types will be
returned.

All of the types so far are assumed to be both nullable and singular: e.g. a scalar
string returns either null or a singular string. The type system might want to
define that it returns a list of other types; the `List` type is provided for
this reason, and wraps another type. Similarly, the `Non-Null` type wraps
another type, and denotes that the result will never be null. These two types
are refered to as "wrapping types"; non-wrapping types are refered to as
"base types". A wrapping type has an underlying "base type", found by
continually unwrapping the type until a base type is found.

Finally, oftentimes it is useful to provide complex structs as inputs to
GraphQL queries; the `Input Object` type allows the schema to define exactly
what data is expected from the client in these queries.

### Scalars

As expected by the name, a scalar represents a primitive value in GraphQL.
GraphQL responses take the form of a hierarchal tree; the leaves on these trees
are GraphQL scalars.

All GraphQL scalars are representable as strings, though depending on the
response format being used, there may be a more appropriate primitive for the
given scalar type, and server should use those types when appropriate.

GraphQL provides a number of built-in scalars, but type systems can add
additional scalars with semantic meaning. For example, a GraphQL system could
define a scalar called `Time` which, while serialized as a string, promises to
conform to ISO-8601. When querying a field of type `Time`, you can then rely on
the ability to parse the result with an ISO-8601 parser and use a
client-specific primitive for time. Another example of a potentially useful
custom scalar is `Url`, which serializes as a string, but is guaranteed by
the server to be a valid URL.

**Result Coercion**

A GraphQL server, when preparing a field of a given scalar type, must uphold the
contract the scalar type describes, either by coercing the value or
producing an error.

For example, a GraphQL server could be preparing a field with the scalar type
`Int` and encounter a floating-point number. Since the server must not break the
contract by yielding a non-integer, the server should truncate the fractional
value and only yield the integer value. If the server encountered a boolean
`true` value, it should return `1`. If the server encountered a string, it may
attempt to parse the string for a base-10 integer value. If the server
encounters some value that cannot be reasonably coerced to an `Int`, then it
must raise an field error.

Since this coercion behavior is not observable to clients of the GraphQL server,
the precise rules of coercion are left to the implementation. The only
requirement is that the server must yield values which adhere to the expected
Scalar type.

**Input Coercion**

If a GraphQL server expects a scalar type as input to a field argument, coercion
is observable and the rules must be well defined. If an input value does not
match a coercion rule, a query error must be raised.

GraphQL has different constant literals to represent integer and floating-point
input values, and coercion rules may apply differently depending on which type
of input value is encountered. GraphQL may be parameterized by query variables,
the values of which are often serialized when sent over a transport like HTTP. Since
some common serializations (ex. JSON) do not discriminate between integer
and floating-point values, they are interpretted as an integer input value if
they have an empty fractional part (ex. `1.0`) and otherwise as floating-point
input value.

#### Built-in Scalars

GraphQL provides a basic set of well-defined Scalar types. A GraphQL server
should support all of these types, and a GraphQL server which provide a type by
these names must adhere to the behavior described below.

##### Int

The Int scalar type represents a signed 32-bit numeric non-fractional values.
Response formats that support a 32-bit integer or a number type should use
that type to represent this scalar.

**Result Coercion**

GraphQL servers should coerce non-int raw values to Int when possible
otherwise they must raise a field error. Examples of this may include returning
`1` for the floating-point number `1.0`, or `2` for the string `"2"`.

**Input Coercion**

When expected as an input type, only integer input values are accepted. All
other input values, including strings with numeric content, must raise a query
error indicating an incorrect type. If the integer input value represents a
value less than -2<sup>31</sup> or greater than or equal to 2<sup>31</sup>, a
query error should be raised.

Note: Numeric integer values larger than 32-bit should either use String or a
custom-defined Scalar type, as not all platforms and transports support
encoding integer numbers larger than 32-bit.

##### Float

The Float scalar type represents signed double-precision fractional values
as specified by [IEEE 754](http://en.wikipedia.org/wiki/IEEE_floating_point).
Response formats that support an appropriate double-precision number type
should use that type to represent this scalar.

**Result Coercion**

GraphQL servers should coerce non-floating-point raw values to Float when
possible otherwise they must raise a field error. Examples of this may include
returning `1.0` for the integer number `1`, or `2.0` for the string `"2"`.

**Input Coercion**

When expected as an input type, both integer and float input values are
accepted. Integer input values are coerced to Float by adding an empty
fractional part, for example `1.0` for the integer input value `1`. All
other input values, including strings with numeric content, must raise a query
error indicating an incorrect type. If the integer input value represents a
value not representable by IEEE 754, a query error should be raised.

##### String

The String scalar type represents textual data, represented as UTF-8 character
sequences. The String type is most often used by GraphQL to represent free-form
human-readable text. All response formats must support string representations,
and that representation must be used here.

**Result Coercion**

GraphQL servers should coerce non-string raw values to String when possible
otherwise they must raise a field error. Examples of this may include returning
the string `"true"` for a boolean true value, or the string `"1"` for the
integer `1`.

**Input Coercion**

When expected as an input type, only valid UTF-8 string input values are
accepted. All other input values must raise a query error indicating an
incorrect type.

##### Boolean

The Boolean scalar type represents `true` or `false`. Response formats should
use a built-in boolean type if supported; otherwise, they should use their
representation of the integers `1` and `0`.

**Result Coercion**

GraphQL servers should coerce non-boolean raw values to Boolean when possible
otherwise they must raise a field error. Examples of this may include returning
`true` for any non-zero number.

**Input Coercion**

When expected as an input type, only boolean input values are accepted. All
other input values must raise a query error indicating an incorrect type.

##### ID

The ID scalar type represents a unique identifier, often used to refetch an
object or as key for a cache. The ID type is serialized in the same way as
a `String`; however, it is not intended to be human-readable. While it is
often numeric, it should always serialize as a `String`.

**Result Coercion**

GraphQL is agnostic to ID format, and serializes to string to ensure consistency
across many formats ID could represent, from small auto-increment numbers, to
large 128-bit random numbers, to base64 encoded values, or string values of a
format like [GUID](http://en.wikipedia.org/wiki/Globally_unique_identifier).

GraphQL servers should coerce as appropriate given the ID formats they expect,
when coercion is not possible they must raise a field error.

**Input Coercion**

When expected as an input type, any string (such as `"4"`) or integer (such
as `4`) input value should be coerced to ID as appropriate for the ID formats
a given GraphQL server expects. Any other input value, including float input
values (such as `4.0`), must raise a query error indicating an incorrect type.


### Objects

GraphQL queries are hierarchal and composed, describing a tree of information.
While Scalar types describe the leaf values of these hierarchal queries, Objects
describe the intermediate levels.

GraphQL Objects represent a list of named fields, each of which yield a value of
a specific type. Object values are serialized as unordered maps, where the
queried field names (or aliases) are the keys and the result of evaluating
the field is the value.

For example, a type `Person` could be described as:

```
type Person {
  name: String
  age: Int
  picture: Url
}
```

Where `name` is a field that will yield a `String` value, and `age` is a field
that will yield an `Int` value, and `picture` a field that will yield a
`Url` value.

A query of an object value must select at least one field. This selection of
fields will yield an unordered map containing exactly the subset of the object
queried. Only fields that are declared on the object type may validly be queried
on that object.

For example, selecting all the fields of `Person`:

```
{
  name,
  age,
  picture
}
```

Would yield the object:

```
{
  "name": "Mark Zuckerberg",
  "age": 30,
  "picture": "http://some.cdn/picture.jpg"
}
```

While selecting a subset of fields:

```
{
  name,
  age
}
```

Must only yield exactly that subset:

```
{
  "name": "Mark Zuckerberg",
  "age": 30
}
```

A field of an Object type may be a Scalar, Enum, another Object type,
an Interface, or a Union. Additionally, it may be any wrapping type whose
underlying base type is one of those five.

For example, the `Person` type might include a `relationship`:

```
type Person {
  name: String
  age: Int
  picture: Url
  relationship: Person
}
```

Valid queries must supply a nested field set for a field that returns
an object, so this query is not valid:

```!
{
  name,
  relationship
}
```

However, this example is valid:

```
{
  name,
  relationship {
    name
  }
}
```

And will yield the subset of each object type queried:

```
{
  "name": "Mark Zuckerberg",
  "relationship": {
    "name": "Priscilla Chan"
  }
}
```

**Result Coercion**

Determining the result of coercing an object is the heart of the GraphQL
executor, so this is covered in that section of the spec.

**Input Coercion**

Objects are never valid inputs.

#### Object Field Arguments

Object fields are conceptually functions which yield values. Occasionally object
fields can accept arguments to further specify the return value. Field arguments
are defined as a list of all possible argument names and their expected input
types.

For example, a `Person` type with a `picture` field could accept an argument to
determine what size of an image to return.

```
type Person {
  name: String
  picture(size: Int): Url
}
```

GraphQL queries can optionally specify arguments to their fields to provide
these arguments.

This example query:

```
{
  name,
  picture(size: 600)
}
```

May yield the result:

```
{
  "name": "Mark Zuckerberg",
  "picture": "http://some.cdn/picture_600.jpg"
}
```

The type of a field argument can be a Scalar, as it was in this example, or an
Enum. It can also be an Input Object, covered later in this document, or it can
be any wrapping type whose underlying base type is one of those three.

#### Object Field deprecation

Fields in an object may be marked as deprecated as deemed necessary by the
application. It is still legal to query for these fields (to ensure existing
clients are not broken by the change), but the fields should be appropriately
treated in documentation and tooling.

#### Object type validation

Object types have the potential to be invalid if incorrectly defined. This set
of rules must be adhered to by every Object type in a GraphQL schema.

1. The fields of an Object type must have unique names within that Object type;
   no two fields may share the same name.
2. An object type must be a super-set of all interfaces it implements.
   1. The object type must include a field of the same name for every field
      defined in an interface.
      1. The object field must include an argument of the same name for every
         argument defined by the interface field.
         1. The object field argument must accept the same type (invariant) as
            the interface field argument.
      2. The object field must be of a type which is equal to
         the interface field.


### Interfaces

GraphQL Interfaces represent a list of named fields and their arguments. GraphQL
object can then implement an interface, which guarantees that they will
contain the specified fields.

Fields on a GraphQL interface have the same rules as fields on a GraphQL object;
their type can be Scalar, Object, Enum, Interface, or Union, or any wrapping
type whose base type is one of those five.

For example, a interface may describe a required field and types such as
`Person` or `Business` may then implement this interface.

```
interface NamedEntity {
  name: String
}

type Person : NamedEntity {
  name: String
  age: Int
}

type Business : NamedEntity {
  name: String
  employeeCount: Int
}
```

Fields which yield an interface are useful when one of many Object types are
expected, but some fields should be guaranteed.

To continue the example, a `Contact` might refer to `NamedEntity`.

```
type Contact {
  entity: NamedEntity
  phoneNumber: String
  address: String
}
```

This allows us to write a query for a `Contact` that can select the
common fields.

```
{
  entity {
    name
  },
  phoneNumber
}
```

When querying for fields on an interface type, only those fields declared on
the interface may be queried. In the above example, `entity` returns a
`NamedEntity`, and `name` is defined on `NamedEntity`, so it is valid. However,
the following would not be a valid query:

```!
{
  entity {
    name,
    age
  },
  phoneNumber
}
```

because `entity` refers to a `NamedEntity`, and `age` is not defined on that
interface. Querying for `age` is only valid when the result of `entity` is a
`Person`; the query can express this using a fragment or an inline fragment:

```
{
  entity {
    name,
    ... on User {
      age
    }
  },
  phoneNumber
}
```

**Result Coercion**

The interface type should have some way of determining which object a given
result corresponds to. Once it has done so, the result coercion of the interface
is the same as the result coercion of the object.

**Input Coercion**

Interfaces are never valid inputs.

#### Interface type validation

Interface types have the potential to be invalid if incorrectly defined.

1. The fields of an Interface type must have unique names within that Interface
   type; no two fields may share the same name.


### Unions

GraphQL Unions represent an object that could be one of a list of GraphQL
Object types, but provides for no guaranteed fields between those types.
They also differ from interfaces in that Object types declare what interfaces
they implement, but are not aware of what unions contain them.

With interfaces and objects, only those fields defined on the type can be
queried directly; to query other fields on an interface, typed fragments
must be used. This is the same as for unions, but unions do not define any
fields, so **no** fields may be queried on this type without the use of
typed fragments.

For example, we might have the following type system:

```
Union SearchResult = Photo | Person

type Person {
  name: String
  age: Int
}

type Photo {
  height: Int
  width: Int
}

type SearchQuery {
  firstSearchResult: SearchResult
}
```

When querying the `firstSearchResult` field of type `SearchQuery`, the
query would ask for all fields inside of a fragment indicating the appropriate
type. If the query wanted the name if the result was a Person, and the height if
it was a photo, the following query is invalid, because the union itself
defines not fields:

```!
{
  firstSearchResult {
    name
    height
  }
}
```

Instead, the query would be:

```
{
  firstSearchResult {
    ... on Person {
      name
    }
    ... on Photo {
      height
    }
  }
}
```

**Result Coercion**

The union type should have some way of determining which object a given result
corresponds to. Once it has done so, the result coercion of the union is the
same as the result coercion of the object.

**Input Coercion**

Unions are never valid inputs.

#### Union type validation

Union types have the potential to be invalid if incorrectly defined.

1. The member types of an Union type must all be Object base types;
   Scalar, Interface and Union types may not be member types of a Union.
   Similarly, wrapping types may not be member types of a Union.
2. A Union type must define two or more member types.

When expected as an input type, array input values are accepted only when each
item in the input array can be accepted by the item type. If the input value is
not an array but can be accepted by the item type, it is coerced by creating an
array with the input value as the sole item. Other input values must raise a
query error indicating an incorrect type.

### Enums

GraphQL Enums are a variant on the Scalar type, which represents one of a
finite set of possible values.

GraphQL Enums are not references for a numeric value, but are unique values in
their own right. They serialize as a string: the name of the represented value.

**Result Coercion**

GraphQL servers must return one of the defined set of possible values, if a
reasonable coercion is not possible they must raise a field error.

**Input Coercion**

GraphQL has a constant literal to represent enum input values. GraphQL string
literals must not be accepted as an enum input and instead raise a query error.

Query variable transport serializations which have a different representation
for non-string symbolic values (for example, [EDN](https://github.com/edn-format/edn))
should only allow such values as enum input values. Otherwise, for most
transport serializations that do not, strings may be interpretted as the enum
input value with the same name.


### Input Objects

Fields can define arguments that the client passes up with the query,
to configure their behavior. These inputs can be Strings or Enums, but
they sometimes need to be more complex than this.

The `Object` type defined above is inappropriate for re-use here, because
`Object`s can contain fields that express circular references or references
to interfaces and unions, neither of which is appropriate for use as an
input argument.  For this reason, input objects have a separate type in the
system.

An `Input Object` defines a set of input fields; the input fields are either
scalars, enums, or other input objects. This allows arguments to accept
arbitrarily complex structs.

**Result Coercion**

An input object is never a valid result.

**Input Coercion**

The input to an input object should be an unordered map, otherwise an error
should be thrown. The result of the coercion is an unordered map, with an
entry for each input field, whose key is the name of the input field.
The value of an entry in the coerced map is the result of input coercing the
value of the entry in the input with the same key; if the input does not have a
corresponding entry, the value is the result of coercing null. The input
coercion above should be performed according to the input coercion rules of the
type declared by the input field.


### Lists

A GraphQL list is a special collection type which declares the type of each
item in the List (referred to as the *item type* of the list). List values are
serialized as ordered lists, where each item in the array is serialized as per
the item type.

**Result Coercion**

GraphQL servers must return an ordered list as the result of a list type. Each
item in the list must be the result of a result coercion of the item type. If a
reasonable coercion is not possible they must raise a field error. In
particular, if a non-list is returned, the coercion should fail, as this
indicates a mismatch in expectations between the type system and the
implementation.

**Input Coercion**

When accepted as an input, each item in the list should be coerced as per
the input coercion of the item type.

If the value passed as an input to a list type is *not* as list, it should be
coerced as though the input was a list of size one, where the value passed is
the only item in the list. This is to allow inputs that accept a "var args"
to declare their input type as a list; if only one argument is passed (a common
case), the client can just pass that value rather than constructing the list.

### Non-Null

By default, all types in GraphQL are nullable; the {null} value is a valid
response for all of the above types. To declare a type that disallows null,
the GraphQL Non-Null type can be used. This type declares an underlying type,
and this type acts identically to that underlying type, with the exception
that `null` is not a valid response for the wrapping type.

**Result Coercion**

In all of the above result coercion, `null` was considered a valid value.
To coerce the result of a Non Null type, the result coercion of the
underlying type should be performed. If that result was not `null`, then the
result of coercing the Non Null type is that result. If that result was `null`,
then an error should be raised.

**Input Coercion**

When accepted as an input, each item in the list should be coerced as per
the input coercion of the item type.

If the value passed as an input to a list type is *not* as list, it should be
coerced as though the input was a list of size one, where the value passed is
the only item in the list. This is to allow inputs that accept a "var args"
to declare their input type as a list; if only one argument is passed (a common
case), the client can just pass that value rather than constructing the list.


## Directives

A GraphQL schema includes a list of supported directives, each of which has
a name. Directives can apply to operations, fragments, or fields; each directive
indicates which of those it applies to.

Directives can optionally take an argument. The type of the argument to
a directive has the same restrictions as the type of an argument to a field. It
can be a Scalar, an Enum, an Input Object, or any wrapping type whose underlying
base type is one of those three.

## Starting types

A GraphQL schema includes types, indicating where query and mutation
operations start. This provides the initial entry points into the
type system. The query type must always be provided, and is an Object
base type. The mutation type is optional; if it is null, that means
the system does not support mutations. If it is provided, it must
be an object base type.

The fields on the query type indicate what fields are available at
the top level of a GraphQL query. For example, a basic GraphQL query
like this one:

```graphql
query getMe {
  me
}
```

Is valid when the type provided for the query starting type has a field
named "me". Similarly

```graphql
mutation setName {
  setName(name: "Zuck") {
    newName
  }
}
```

Is valid when the type provided for the mutation starting type is not null,
and has a field named "setName" with a string argument named "name".
