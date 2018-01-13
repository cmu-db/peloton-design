# Array Type

## Overview
Array type is a type for `Value`, meaning that `Value` represents an variable-length array of a built-in or user-defined base type, enum type, or composite type. We add array type to replace hacks that convert an array to VARCHAR and further to support queries containing arrays.

## Scope
Modifications to replace the hacks is mainly based on `Type` and `Value`, and supporting queries also involves `expression`/`parser`/`planner`/`traffic_cop`. The corresponding type ids need to be added.

We create a `ArrayType` class inherited from `Type` class, incorporate array type into constructors and destructor of `Value`, and update `ValueFactory`.

The implementation can be broken down into the following:

### ArrayType Class ###
* Provides functions to express, access and manipulate arrays
* Overloads virtual comparison functions
* Defines how to serialize or deserialize arrays into or from a memory pool

### Supporting Queries ###
* *expression*: add `ArrayExpression` inherited from `AbstractExpression` to wrap a `Value` of array type.
* *parser*: add functions to take in Postgres ArrayExpr primnode and transfers it into Peloton `AbstractExpression`
* *planner*: add a branch in insert plan for Array Expression type
* *traffic_cop*: add selections for array field_types

## Architectural Design
Vectors are utilized to maintain arrays and we store a pointer to its vector inside `Value`.

`manage_data` can be indicated if we want to retain ownership of the vector when passing it to create a value. We should set it to `true` when we want to keep ownership, which means that we should make sure memory leaks will not happen by ourselves. For example, we can declare a vector variable, in which case the memory is allocated on stack and will be deallocated automatically. Or we have to delete a vector manually after newing one. On the other hand, it should only be `false` when memory of the vector is allocated on heap and we don't want to be responsible for deallocating it once the value it belongs to is created. The reason for the latter option is to improve performance since only pointers to vectors are passed through the whole routine in this case, rather than actual values. And the memory of vectors will be released when the value is destructed.

`SerializeTo` in `ArrayType` serializes a array value into a given storage space. The first 4 bytes is the total number of elements in the array and then the remaining bytes are the elements. The type of element can be determined by type id, an attribute in `Value`, which can be accessed during serialization and deserialization, so we don't have to store it in the space. `DeserializeFrom` deserializes a value of the given type from a given storage space in reverse order.

## Design Rationale
The goal of this design is to fit array type implementation smoothly into the existing type module.

We adopt the single type id scheme for array type to avoid function overloading and routine changes, that is to say, we have a separate type id for every primitive type array, sharing the same Array Class.

## Testing Plan
### Serialization and Deserialization ###
`SerializeTest` serializes array values and then deserialize them to check if they remain the same.

### Element Retrieval ###
`GetElementTest` creates vectors of different types and inserts *n* elements into each vector. Each vector is used to create a value, and each element retrieved from the value should be equal to matching element in the vector.

### Comparison Functions ###
`CompareTest` contains various comparison function tests.

### Client Queries ###
`QueryTest` tests queries coming in from SQL that contains arrays. 

## Trade-offs and Potential Problems

## Future Work
Array type are fully supported internally. For queries from clients, however, this is only for the traditional interpreter currently and additional work is required for codegen.

We can support multidimensional array by adding a `TypeId::ARRAYARRAY` type id, the `Value` which belongs to contains a vector of `Value`. For example, to represent a two-dimensional array of integer, each `Value` in the vector can have a `TypeId::INTEGERARRAY` type id, containing a vector of integers. If we want to represent a three-dimensional array, then each `Value` in the vector can have a `TypeId::ARRAYARRAY` type id again, and so on and so forth.

The following functions need to be overloaded if we want queries involving arrays to benefit from optimizers.
```
virtual bool operator==(const AbstractExpression &rhs) const;
virtual bool operator!=(const AbstractExpression &rhs) const {
       return !(*this == rhs);
}
virtual hash_t Hash() const;
virtual bool ExactlyEquals(const AbstractExpression &other) const;
virtual hash_t HashForExactMatch() const;
```
