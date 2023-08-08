---
title: "Typescript Learning Notes"
date: 2023-08-08T15:35:03+08:00
author: "Feng Xue"
draft: true
---

## Type

### `any` type

`any` type means the variable can be assigned to any type of value. With this value, typescript would think developer prefer to handling these codes self. And `any` type should only be used for limited reasons:

1. Need to close the type checking for some variables
2. To adopt the old javascript project, we could set it as `any` temporarily
3. For some values defined by some old library without type

#### Type infer (类型推断)

All the variables without types would be regarded as `any`, which makes type-checking meaningless.

#### Type pollution

Since the `any` type variable can be assigned to any other variable, typescript would not report error if the type is `any`.

### `unknown` type

You can regard `unknown` as a strict version of `any`. 

1. All types values can be assigned to the `unknown` type variable, which avoids the pollution problem
2. Methods and properties in the `unknown` type variable cannot be called
3. Only operators: `==`, `===`, `!=`, `!==`, `||`, `&&`, `?`, `!`, `typeof`, `instanceof` can be used with `unknown` type

To use the `unknown` type, we have to `narrow` the type, which means we have to specify the variable with the specific type, like `number`, `Record`, etc.

```ts
let a:unknown = 1;

if (typeof a === 'number') {
  let r = a + 10; // Correct
}
```

### `never` type

`never` means it should **never** have any value, so no value can be assigned to it, while it could be assigned to any other type value. So `unknown` and `never` are like inverses of each other.

### Wrapper Object Type

Javascript defines the basic 8 types of values:

* boolean
* string
* number
* bigint
* symbol
* object
* undefined
* null

But for these five types:

* boolean
* string
* number
* bigint
* symbol

All of them have a wrapper object, when they call a method, javascript would convert them to the related wrapper object automatically, like this:

```js
'hello'.charAt(1) // 'e'
```

So typescript would have two types for these basic values: `Wrapper Object` type and `Literal` Type.

* `Boolean` and `boolean`
* `String` and `string`
* `Number` and `number`
* `BigInt` and `bigint`
* `Symbol` and `symbol`

Be careful, Wrapper Object type cannot be assigned to Literal type:

```ts
const s2:String = new String('hello'); // Correct

const s4:string = new String('hello'); // Wrong
```

I will paste the [official table](https://www.typescriptlang.org/docs/handbook/type-compatibility.html#any-unknown-object-void-undefined-null-and-never-assignability) here. `->` means the row type is assigned to the column type.

![Typescript types relationship](/images/typescript-learning/typescript-types-relationship-table.png 'Typescript types relationship')

## Array

In typescript, array defined with `const` can be changed (since it's a reference). So if we want the unchanged array, we need the keyword `readonly`.

Or set the array with `const` assertion: 

```ts
const arr = [0, 1] as const;
```