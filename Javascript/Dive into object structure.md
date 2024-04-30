## Preface

Objects in Javascript are very flexible and understandable. We can easily add, remove and modify object properties on our own. However, there are few people thinking about how objects are stored in memory and processed by Js engines. Can a developer's actions, directly or indirectly,impact performance and memory consumption? I think it's an interesting topic worthy of diving into.

## Object and its properties

As well known,Object properties are represented as a "key-value" structure,where the key is the property name, and the value is a set of attributes. All object properties can be conventionally divided into two types: data properties and accessor properties.

### Data properties

Properties which have the following attributes:
* \[[Value]] - the value of the property.
* \[[Writable]] - boolean,by default set to false. If false, the [[Value]] connot be changed.
* \[[Enumerable]] - boolean,by default set to false. If true,the property can be iterated through using "for-in".
* \[[Configurable]] - boolean,by default set to false. If false,the property cannot be deleted,its type cannot be changed from Data property to Accessor property,no attributes except for \[[Value]] and setting \[[Writable]] can be set to false.

### Accessor properties

Properties which have the following attributes:
* \[[Get]] - a function that returns the value of the object.
* \[[Set]] - a function called when an attempt to assign a value to the property is made.
* \[[Enumberable]] - identical to Data property.
* \[[Configurable]] - identical to Data property.

## Hidden Classes
Every object property will store information for its attributes. For Example,
```javascript
const obj1 = { a: 1, b: 2 };
```
This simple object, in the context of a Javascript engine,should look something like this:
```javascript
{
  a: {
    [[Value]]: 1,
    [[Writable]]: true,
    [[Enumerable]]: true,
    [[Configurable]]: true,
  },
  b: {
    [[Value]]: 2,
    [[Writable]]: true,
    [[Enumerable]]: true,
    [[Configurable]]: true,
  }
}
```
Let's imagine that we have two objects with similar structures.
```
const obj1 = { a: 1, b: 2 };
const obj2 = { a: 3, b: 4 };
```
If obj1 and obj2 both has a descriptor object to store information for their properties,that will be wasteful in terms of memory consumption. Obviously, except the \[[Value]], other data property is identical. To solve this problem, the popular Js engines using so-called **hidden classes**, which extracting meta-information and object properties into separate, reusable objects and binding such a class to the real object by reference. So the two objects example above will be described by hidden class like this:
```javascript
Map1 {
  a: {
    [[Writable]]: true,
    [[Enumerable]]: true,
    [[Configurable]]: true,
  },
  b: {
    [[Writable]]: true,
    [[Enumerable]]: true,
    [[Configurable]]: true,
  }
}

ob1 {
  map: Map1,
  values: { a: 1, a: 2 }
}

ob2 {
  map: Map1,
  values: { a: 3, a: 4 }
}
```
## Hidden Classes Inheritance
The concept of hidden classes looks good in the case of objects with the same structure. However, what to do if the second object has a different structure? for example:
```
const obj1 = { a: 1, b: 2 };
const obj2 = { a: 3, b: 4, c: 5 };
```
This two objects have different shapes but also have same properties. To avoid the problem of attribute duplication, the JS engines adopted the mechanism of hidden classes inheritance like this:
```
Map1 {
  a: {
    [[Writable]]: true,
    [[Enumerable]]: true,
    [[Configurable]]: true,
  },
  b: {
    [[Writable]]: true,
    [[Enumerable]]: true,
    [[Configurable]]: true,
  }
}

Map2 {
  back_pointer: Map1,
  с: {
    [[Writable]]: true,
    [[Enumerable]]: true,
    [[Configurable]]: true,
  }
}

ob1 {
  map: Map1,
  values: { a: 1, b: 2 }
}

ob2 {
  map: Map2,
  values: { a: 3, b: 4, c: 5 }
}
```
Here we see that the class Map2 describes only one property and a reference to an object with a more "specific" shape.

Something must be notable is that the shape of the hidden class is influenced not only by the set of properties but also by their order. In other words,the following objects will have different shapes of hidden classes.
```javascript
Map1 {
  a: {
    [[Writable]]: true,
    [[Enumerable]]: true,
    [[Configurable]]: true,
  },
  b: {
    [[Writable]]: true,
    [[Enumerable]]: true,
    [[Configurable]]: true,
  }
}

Map2 {
  b: {
    [[Writable]]: true,
    [[Enumerable]]: true,
    [[Configurable]]: true,
  },
  a: {
    [[Writable]]: true,
    [[Enumerable]]: true,
    [[Configurable]]: true,
  }
}

ob1 {
  map: Map1,
  values: { a: 1, b: 2 }
}

ob2 {
  map: Map2,
  values: { b: 3, a: 4 }
}
```
If we change the shape of the object after initialization,this will also lead to the creation of a new hidden subclass.
```javascript
const ob1 = { a: 1, b: 2 };
obj1.c = 3;

const obj2 = { a: 4, b: 5, c: 6 };
```
This example results in the following structure of hidden classes.
```javascript
Map1 {
  a: {
    [[Writable]]: true,
    [[Enumerable]]: true,
    [[Configurable]]: true,
  },
  b: {
    [[Writable]]: true,
    [[Enumerable]]: true,
    [[Configurable]]: true,
  }
}

Map2 {
  back_pointer: Map1,
  с: {
    [[Writable]]: true,
    [[Enumerable]]: true,
    [[Configurable]]: true,
  }
}

Map3 {
  back_pointer: Map1,
  с: {
    [[Writable]]: true,
    [[Enumerable]]: true,
    [[Configurable]]: true,
  }
}

ob1 {
  map: Map2,
  values: { a: 1, b: 2, c: 3 }
}

ob2 {
  map: Map3,
  values: { a: 4, b: 5, c: 6 }
}
```
The process of creating a new hidden class with the addition or modification of any property is called a **transition**.

However, the interpreter does not recognize this until it starts initializing the second object. In order to create a new class tree, the engine would have to search for a class in the heap that matches the form, break it down into parts. To avoid this,the developers of V8 decided to divide the class into a set of minimal forms right away,starting with an empty class during the first object initialization.
```javascript
{}
{ a }
{ a, b }
{ a, b, c }
```
This approach allows for the engine easily find a suitable initial form for the new object, there is no need to rebuild anything,just create a new class with a reference to the appropriate minimal form.

## Fast and Slow Properties

Let's consider the following example:
```javascript
const obj1 = { a: 1 };
obj1.b = 2;
```
If we use the V8's built-in internal method - %DebugPrint to see the detailed information about the object, we will get some information like this:
```
d8> %DebugPrint(obj1);
DebugPrint: 0x2387001c942d: [JS_OBJECT_TYPE]
 - map: 0x2387000dabb1 <Map[16](HOLEY_ELEMENTS)> [FastProperties]
 - prototype: 0x2387000c4b11 <Object map = 0x2387000c414d>
 - elements: 0x2387000006cd <FixedArray[0]> [HOLEY_ELEMENTS]
 - properties: 0x2387001cb521 <PropertyArray[3]>
 - All own properties (excluding elements): {
    0x238700002a21: [String] in ReadOnlySpace: #a: 1 (const data field 0), location: in-object
    0x238700002a31: [String] in ReadOnlySpace: #b: 2 (const data field 1), location: properties[0]
 }
0x2387000dabb1: [Map] in OldSpace
 - map: 0x2387000c3c29 <MetaMap (0x2387000c3c79 <NativeContext[285]>)>
 - type: JS_OBJECT_TYPE
 - instance size: 16
 - inobject properties: 1
 - unused property fields: 2
 - elements kind: HOLEY_ELEMENTS
 - enum length: invalid
 - stable_map
 - back pointer: 0x2387000d9ca1 <Map[16](HOLEY_ELEMENTS)>
 - prototype_validity cell: 0x2387000dabd9 <Cell value= 0>
 - instance descriptors (own) #2: 0x2387001cb4f9 <DescriptorArray[2]>
 - prototype: 0x2387000c4b11 <Object map = 0x2387000c414d>
 - constructor: 0x2387000c4655 <JSFunction Object (sfi = 0x238700335385)>
 - dependent code: 0x2387000006dd <Other heap object (WEAK_ARRAY_LIST_TYPE)>
 - construction counter: 0
```
We can see that the property a is marked as in-object, while the property b is marked as an element of the properties array.
```javascript
- All own properties (excluding elements): {
    ... #a: 1 (const data field 0), location: in-object
    ... #b: 2 (const data field 1), location: properties[0]
 }
```
This example demonstrates that some properties are stored directly inside the object itself("in-object"), while others are stored in an external storage of properties. This is related to the fact that according to [ECMA-262](https://262.ecma-international.org/#sec-object-type) specification, Javascript objects do not have a fixed size. By adding or removing properties from an object,its size changes. This raises the question: how much memory should be allocated for the object?Furthermore, how can we expand the already allocated memory for the object? Developers of V8 addressed these issues by the so-called external properties.

Properties that were added after initialization can no longer be placed inside the object since memory for the object is already allocated. To avoid wasting resources on reallocating the entire object, the engine places such properties in an external storage. In this case, in an external array of properties, a reference to which already exists inside the object. These properties are called external or normal. Access to such properties is slightly slower,as it requires resolving the reference to the storage and obtaining the property by index. However,this is much more efficient than reallocating the entire object.

Except for the addtion or modification case, we can also remove some properties for an object, when executing a remove action, the tree of object shapes has become shorter, so the descriptors and value arrays must also be reconstructed. All these operations require certain time resources. To avoid this problem, V8 will place the properties in a separate dictionary object together with all their attributes. Access to both the values and attributes of such properties is done by their name,which serves as the key to the dictionary and it is slower.
```javascript
d8> delete obj1.a;
d8>
d8> %DebugPrint(obj1)
DebugPrint: 0x2387001c942d: [JS_OBJECT_TYPE]
 - map: 0x2387000d6071 <Map[12](HOLEY_ELEMENTS)> [DictionaryProperties]
 - prototype: 0x2387000c4b11 <Object map = 0x2387000c414d>
 - elements: 0x2387000006cd <FixedArray[0]> [HOLEY_ELEMENTS]
 - properties: 0x2387001cc1d9 <NameDictionary[30]>
 - All own properties (excluding elements): {
   b: 2 (data, dict_index: 2, attrs: [WEC])
 }
0x2387000d6071: [Map] in OldSpace
 - map: 0x2387000c3c29 <MetaMap (0x2387000c3c79 <NativeContext[285]>)>
 - type: JS_OBJECT_TYPE
 - instance size: 12
 - inobject properties: 0
 - unused property fields: 0
 - elements kind: HOLEY_ELEMENTS
 - enum length: invalid
 - dictionary_map
 - may_have_interesting_properties
 - back pointer: 0x238700000061 <undefined>
 - prototype_validity cell: 0x238700000a31 <Cell value= 1>
 - instance descriptors (own) #0: 0x238700000701 <DescriptorArray[0]>
 - prototype: 0x2387000c4b11 <Object map = 0x2387000c414d>
 - constructor: 0x2387000c4655 <JSFunction Object (sfi = 0x238700335385)>
 - dependent code: 0x2387000006dd <Other heap object (WEAK_ARRAY_LIST_TYPE)>
 - construction counter: 0
```


## Conclusion

In this article, we have delved deeper into the object structure, including the methods of storing object properties, concepts of hidden classes, object shapes, fast and slow properties. The main rules include:
* Every object in Javascript has its main internal class and a hidden class that describes its shape.
* Hidden classes inherit from each other and are organized into class trees. The shape of an object { a: 1 } will be the parent for the shape of an object 
{ a: 1, b: 2 }
* The order of properties matters. Objects { a: 1, b: 2 } and { b: 2, a: 1 } will have two different shapes.
* In the class tree of each other, the number of levels is not less than the number of properties in the object.
* The fastest properties of an object will be those declared at initialization. In the following example, access to the property obj1.a will be faster than to obj2.a.
```javascript
const obj1 = { a: undefined };
obj1.a = 1; // <- "a" - in-object property

const obj2 = {};
obj2.a = 1; // <- "a" - external property
```
* Atypical changes in the object's structure, such as property removal, can lead to a change in the storage type of properties to a slower one. In the following example,obj1 will change its type to NamedDictionary, and accessing its properties will be significantly slower than accessing the properties of obj2.
```javascript
const obj1 = { a: 1, b: 2 };
delete obj1.a; // cheanges a storage type to NameDictionary 

const obj2 = { a: 1, b: 2 };
obj2.a = undefined; // a storage type is not changed
```
* If an object has external properties but the internal ones are less 4, such an object can be slightly optimized,as an empty object by default has 4 slots for in-object properties.
```javascript
const obj1 = { a: 1 };
obj1.b = 2;
obj1.c = 3;
obj1.d = 4;
obj1.e = 5;
obj1.f = 6;

%DebugPrint(obj1);
...
- All own properties (excluding elements): {
    ...#a: 1 (const data field 0), location: in-object
    ...#b: 2 (const data field 1), location: properties[0]
    ...#c: 3 (const data field 2), location: properties[1]
    ...#d: 4 (const data field 3), location: properties[2]
    ...#e: 5 (const data field 4), location: properties[3]
    ...#f: 6 (const data field 5), location: properties[4]
 }

const obj2 = Object.fromEntries(Object.entries(obj1));

%DebugPrint(obj2);
...
 - All own properties (excluding elements): {
    ...#a: 1 (const data field 0), location: in-object
    ...#b: 2 (const data field 1), location: in-object
    ...#c: 3 (const data field 2), location: in-object
    ...#d: 4 (const data field 3), location: in-object
    ...#e: 5 (const data field 4), location: properties[0]
    ...#f: 6 (const data field 5), location: properties[1]
 }
```
