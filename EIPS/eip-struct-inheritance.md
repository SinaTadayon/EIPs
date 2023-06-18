---
title: Struct Inheritance 
description: Creating struct inheritance through composition over inheritance
author: Sina Tadayon <sina.tadayyon@gmail.com>
discussions-to: <URL>
status: Draft
type: Standards Track
category: ERC
created: 2023-06-18
---

## Abstract
This proposal introduces struct inheritance by using struct composition. It describes the type-casting between 
a base struct and a derived struct in the mapping type, the dynamic array type, and also in functions. 
Other prominent features of inheritance, such as polymorphism and encapsulation, are not covered here 
because struct types currently do not have any member functions or member access modifiers.

## Motivation
There are several highlighted reasons to use struct inheritance:

* Converting similar structs into an inheritance hierarchy structure promotes code reusability and reduces code redundancy.
* Aggregating multiple mappings of similar struct types allows for a common mapping to be used for derived struct types. 
* Aggregating multiple dynamic arrays of struct types enables the creation of a common dynamic array for derived struct types (storage mode)
* Substituting function overloading for each similar struct type with a common function for derived struct types.


## Specification
The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", 
"MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

### Terms:
1. A **Base Struct** is a struct whose members are inherited in the inheritance hierarchy.
2. A **Derived Struct** is a struct that inherits those members in the inheritance hierarchy.
3. **Type-casting** includes Up-casting and Down-casting structs in the inheritance hierarchy and can be functional in both memory and storage. 
4. **Up-casting** Struct refers to casting a derived struct to a base struct in an upward direction in the inheritance hierarchy.
5. **Down-casting** Struct refers to casting a base struct to a derived struct in a downward direction in the inheritance hierarchy.
6. **Extended-Mapping** is a mapping with the base struct as the value type.
7. **Extended-Array** is a dynamic array with the base struct as the element type. 
8. **Extended-Function** is a substitute function for overloaded functions in the inheritance hierarchy.

#### NOTE:
The following specifications are based on Solidity versions ^0.7.0 and ^0.8.0. 


### Struct inheritance via composition
- Base Struct SHOULD be implemented by aggregating shared or common members of resemble structs and 
  removing them from their respective structs. Additionally, an enum type SHOULD be implemented 
  to distinguish between derived structs. Each enum type SHOULD correspond to a derived struct and
  SHOULD be added to the base struct.
- Derived Struct should be implemented by inheriting members from the base struct. 
  The base struct should be the first member of the derived struct, followed by any additional members. 

The following demonstrates the proposed implementation of struct inheritance.

```javascript
enum DerivedType {
  NONE,
  DERIVED_A,
  DERIVED_B
}

struct BaseStruct {
  DerivedType dtype;
  // shared members of other derived structs
}

struct Derived_A {
  BaseStruct base;
  // other members
}

struct Derived_B {
  BaseStruct base;
  // other members
}
```

### Extended-Mapping (storage mode)
- The Key type can be any built-in value type, bytes, string, or any contract or enum type.
- The Value type must be a base struct.
- Up-casting a derived struct to the base struct should be done by accessing or retrieving the base struct member within the derived struct.
- Down-casting the base struct to a derived struct should be done by invoking a casting function specifically designed 
  for this purpose. The cast function should locate the storage slot of the base struct by calculating 
  the slot number using the formula **_keccak256(h(k) . p)_**.
  Since the base struct is defined as the first member of the derived struct, 
  the identified slot can be assigned to the slot attribute of the derived struct. 
  This is possible because the first slot of the base struct coincides with the first slot of 
  the derived struct.
  It is important to validate the identified slot for both GET and SET actions to ensure proper functionality.

The following contract demonstrates the proposed implementation of the extended-mapping and down-casting function.

```javascript
contract inheritance {
  
  mapping(bytes32 => BaseStruct) internal extendedMaps;

  /// @notice Down-casting Base struct to Derived_A struct
  /// @dev find storage slot of Derived_A by keccak256 function then validate it   
  /// @param keyID the key of request Derived_A
  /// @param isGet the requested action SET or GET
  /// @return da is storage reference of Derived_A 
  function downCastingToDerived_A(bytes32 keyID, bool isGet) internal view returns(Derived_A storage da) {
    assembly {
      // find first free memory address
      let ptr := mload(0x40)

      // put keyID in the first free memory address
      mstore(add(ptr, 0x00), keyID)

      // put extendedMaps.slot in the second free memory address
      mstore(add(ptr, 0x20), extendedMaps.slot)

      // find storage slot with keccak256  
      da.slot := keccak256(ptr, 0x40)
    }

    // validating found storage slot by the DerivedType of baseStruct for reading, writing
    if (isGet) {
      // found storage slot should be DerivedType for getting or updating
      require(da.base.dtype == DerivedType.DERIVED_A, "Invalid DERIVED_A");
      
    // validating found storage slot by DerivedType of baseStruct for saving
    } else {
      // found empty storage slot for saving
      require(da.base.dtype == DerivedType.NONE, "Invalid Slot");
    }
  }
}
```

### Extended-Array (Dynamic array in storage mode)
- The element (array) type MUST be a base struct.
- The push(), push(x), pop() members functions and array subscripting operator (`<array>[<index>]`) MUST be ignored.
- The default formula **_(keccak256(p) + (index * size of element type))_** used to find the first storage slot of each array element MUST be ignored. 
  This formula assumes a fixed size of the element type (base struct) during compilation, which is incorrect for derived 
  structs as the array element size can dynamically change.
- A new push function MUST be defined for each derived struct to append a new element to the extended-array.
- A new get function MUST be defined for each derived struct to retrieve an element from the extended-array.
- A new pop function MUST be defined to remove the last element from the extended-array based on the derived struct type.
- The **_(keccak256(p) + (index * largest size of the derived struct)),_** formula is RECOMMENDED for managing storage slots 
  in a simple manner and ensuring compatibility between different slot sizes of derived structs when finding their storage slots.
- Up-casting a derived struct to the base struct SHOULD be done by accessing or getting the base struct member within the derived struct.
- Down-casting a base struct to a derived struct SHOULD be done by calling one of the new push, get, and pop functions 
  specifically defined for the derived struct.

The following contract demonstrates the proposed implementation of the common extended-array functions.

```javascript
contract inheritance {

  BaseStruct[] internal extendedArray;

  /// @notice Down-casting Base struct to Derived_A struct
  /// @dev find storage slot of Derived_A by keccak256 function then validate it   
  /// @param idx the array index
  /// @return da is storage reference of Derived_A 
  function getDerived_A(uint256 idx) internal view returns (Derived_A storage da) {
    // validating requested index
    require(idx < extendedArray.length, "Invalid Index");
    assembly {
      // find the first free memory slot
      let ptr := mload(0x40)
    
      // put array slot in the first free memory slot
      mstore(add(ptr, 0x00), extendedArray.slot)
    
      // find the storage slot of Derived_A by requested idx
      // X is the number of slots of the biggest derived struct 
      da.slot := add(keccak256(ptr, 0x20), mul(idx, X))
    }

    // validating found storage slot by DerivedType of baseStruct
    require(da.base.dtype == DerivedType.DERIVED_A, "Invalid DERIVED_A");
  }

  /// @notice appending Derived_A struct to array 
  /// @dev find storage slot of Derived_A by keccak256 function then validate it   
  /// @param derivedA appending to array 
  /// @return da is storage reference of Derived_A 
  function pushDerived_A(Derived_A memory derivedA) internal returns (Derived_A storage da) {
    assembly {
      // find the first free memory slot
      let ptr := mload(0x40)

      // find length of array
      let lastIndex := sload(extendedArray.slot)

      // put array slot in the first free memory slot
      mstore(add(ptr, 0x00), extendedArray.slot)

      // update length of array and save it
      sstore(extendedArray.slot, add(lastIndex, 0x01))

      // find the storage slot of Derived_A by requested idx
      // X is the number of slots of the biggest derived struct 
      da.slot := add(keccak256(ptr, 0x20), mul(lastIndex, X))
    }
    
    // assign data to array element
    // da = derivedA;
  }

  /// @notice removing last element of array   
  /// @dev find storage slot of Base struct then down-casting to related derived struct
  function popItem() internal {
    require(extendedArray.length > 0, "Invalid Pop");
    BaseStruct storage base;
    assembly {
      // find the first free memory slot
      let ptr := mload(0x40)

      // put array slot in the first free memory slot
      mstore(add(ptr, 0x00), extendedArray.slot)

      // loading of array length and minus 1 from it
      let length := sub(sload(extendedArray.slot), 0x01)

      // storing new array length              
      sstore(extendedArray.slot, length)

      // find the storage slot of BaseProposal by requested idx
      // X is the slot size of the biggest derived struct       
      base.slot := add(keccak256(ptr, 0x20), mul(length, X))
    }

    // checking element type    
    if(base.dtype == DerivedType.DERIVED_A) {
      Derived_A storage da;

      // down-casting baseStruct to DERIVED_A
      assembly { da.slot := base.slot }

      // Delete fields of DERIVED_A         
      // delete da;
      
      // checking element type
    } else if (base.dtype == DerivedType.DERIVED_B) {
      Derived_B storage db;

      // down-casting baseStruct to DERIVED_B
      assembly { db.slot := base.slot }

      // Delete fields of DERIVED_B
      // delete db;
    }
  }
}
```

### Extended-Function (memory mode)
- At least one parameter type Must be defined as a base struct.
- Up-casting a derived struct to the base struct SHOULD be done by accessing or getting the base struct member within the derived struct.
- Down-casting the base struct to a derived struct should be done by calling a specific function that performs the casting to the desired derived struct.
- The **_(base struct address pointer - (memory word size * member counts of the derived struct))_** formula is RECOMMENDED for down-casting in memory. 
  This formula allows for finding the memory address of the derived struct by calculating the address offset from the memory address of the base struct. 
  It is important to note that various struct memory layouts exist across different Solidity compiler series.
##### NOTE:
This formula has already been tested with the mentioned Solidity compiler. 

The following demonstrates the proposed implementation of the extended-function.

```javascript 
  /// @notice do something in memory
  /// @param baseStruct the base struct in the memory
  function extendedFunction(BaseStruct memory baseStruct) internal pure {
    // checking BaseStruct type
    if (baseStruct.dtype == DerivedType.DERIVED_A) {
      
      // down-casting to BaseStruct to Derived_A 
      Derived_A memory da = getDerived_A(baseStruct);
    
    } else if(baseStruct.dtype == DerivedType.DERIVED_B) {
      // down-casting to BaseStruct to Derived_B
    }
  }

  /// @notice down-casting base struct to derived struct in memory  
  /// @dev find memory address of derived struct with the base struct - offset
  /// @param bp the base struct in the memory
  /// @return da the Derived_A in the memory
  function getDerived_A(BaseStruct memory bp) internal pure returns(Derived_A memory da) {
    // checking BaseStruct type
    if (bp.dtype == DerivedType.DERIVED_A) {

      // Down-casting BaseStruct to Derived_A
      // offset is difference between the memory address of the base struct and 
      // the memory address of derived struct 
      assembly { da := sub(bp, offset) }
      
    } else { revert("Invalid"); }
  }
```

## Rationale
### Struct Inheritance and Type-casting
Struct inheritance enhances the capabilities of struct types and improves code quality. It enables code reusability, 
minimizes code redundancy, and simplifies code organization and maintenance over time. However, struct inheritance 
becomes meaningful when type-casting between base structs and derived structs is implemented. 
Type-casting between structs should be performed in both memory and storage, although 
this feature is not directly supported by the Solidity compiler.

### Extended-Mapping Type
Combining struct inheritance with the mapping type results in an extended-mapping that can be expanded with newly 
implemented derived struct types. The extended-mapping type adheres to the Open-Closed Principle of SOLID, 
meaning it supports extension by adding newly derived structs while remaining closed for modification of previously 
defined structs in the future. It only supports type-casting in the storage context.

### Extended-Array Type
Combining struct inheritance with the dynamic array type results in an extended-array that can be expanded with newly 
implemented derived struct types. However, it doesn't support updatability, which means it cannot be extended with newly 
derived structs after deployment based on the current design. The extended-array supports type-casting only in the storage context.

### Extended-Function
Using struct type-casting in functions allows for the creation of extended-functions capable of handling various newly 
derived structs. These functions have no limitations in terms of updatability or modification in the future. In essence, 
they serve as an alternative to function overloading, which is directly supported by Solidity. Struct type-casting is 
applicable in both the memory and storage contexts of extended-functions.


## Reference Implementation
### Article
[Medium - Struct Inheritance In Solidity](https://medium.com/@sina.tadayyon/struct-inheritance-in-solidity-143c712f9092)

### Implementation
[Github - Struct Inheritance](https://github.com/SinaTadayon/structInheritance.git) 

## Security Considerations

When implementing the down-cast functions, it is crucial to consider the security implications. 
Down-casting functions should be implemented using inline assembly to ensure precise control over 
the memory and storage operations.

Both in memory and storage, when calculating the address or slot number to be assigned to the derived struct, 
it is essential to validate the calculated value. This validation step is necessary because errors in 
the inline assembly implementation can inadvertently corrupt the data of another derived struct due to 
incorrect calculations.

Errors in the implementation of inline assembly can lead to severe consequences.
The prioritization of safety and security through precise testing and auditing 
is crucial when working with low-level operations in the inline assembly.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
