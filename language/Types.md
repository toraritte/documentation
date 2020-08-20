# Type System

## Primitive/Built-in Types

The type system defines the following types:

> NOTE Should the `Prim` documentation be linked here from Pursuit?

- `Int`
- `Number`
- `String`
- `Function`
- `Char`
- `Boolean`
- `Array`
- Rows
- `Record`

The primitive types `String`, `Number` and `Boolean` correspond to their JavaScript equivalents at runtime.

### `Char`

> NOTE Copied from the `Prim` documentation on Pursuit, mostly to serve as a placeholder.

A single character (UTF-16 code unit). The JavaScript representation is a normal String, which is guaranteed to contain one code unit.

### Functions

Functions in PureScript are like their JavaScript counterparts, but always have exactly one argument.

### Integers

The `Int` type represents integer values. The runtime representation is also a normal JavaScript number; however, operations like `(+)` on `Int` values are defined differently in order to ensure that you always get `Int` values as a result.

### Arrays

PureScript arrays correspond to Javascript arrays at runtime, but all elements in an array must have the same type. The `Array` type takes one type argument to specify what type this is. For example, an array of integers would have the type `Array Int`, and an array of strings would have the type `Array String`.

### Rows

A row of types represents an unordered collection of named types, with duplicates. Duplicate labels have their types collected together in order, as if in a ``NonEmptyList``. This means that, conceptually, a row can be thought of as a type-level ``Map Label (NonEmptyList Type)``.

Rows are not of kind ``Type``: they have kind ``Row k`` for some kind ``k``, and so rows cannot exist as a value. Rather, rows can be used in type signatures to define record types or other type where labelled, unordered types are useful.

To denote a closed row, separate the fields with commas, with each label separated from its type with a double colon:

```purescript
( name :: String, age :: Number )
```

To denote an open row (i.e. one which may unify with another row to add new fields), separate the specified terms from a row variable by a pipe:

```purescript
( name :: String, age :: Number | r )
```

### Records

PureScript records correspond to JavaScript objects. They may have zero or more named fields, each with their own types. For example: `{ name :: String, greet :: String -> String }` corresponds to a JavaScript object with precisely two fields: `name`, which is a `String`, and `greet`, which is a function that takes a `String` and returns a `String`.

## Concepts

- Tagged Unions
- Type Synonyms
- Newtypes
- Polymorphic Types
- Constrained Types

### Type Annotations

Most types can be inferred (not including Rank N Types and constrained types), but annotations can optionally be provided using a double-colon, either as a declaration or after an expression:

```purescript
-- Defined in Data.Semiring
one :: forall a. (Semiring a) => a

-- one can be an Int, since Int is an instance of Semiring where one = 1
int1 :: Int
int1 = one -- same as int1 = 1
-- Or even a Number, which also provides a Semiring instance where one = 1.0
number1 = one :: Number -- same as number1 = 1.0
-- Or its polymorphism can be kept, so it will work with any Semiring
-- (This is the default if no annotation is given)
semiring1 :: forall a. Semiring a => a
semiring1 = one
-- It can even be constrained by another type class
equal1 = one :: forall a. Semiring a => Eq a => a
```

### Quantification

A type variable can refer to not only a type or a row, but a type constructor, or row constructor etc., and type variables with those kinds can be bound inside a ``forall`` quantifier.

### Kind System

The kinds of types are other types. The only types that are not supported in kinds are constraints, which have no kind-level equivalent. Kind polymorphism is supported through top-level kind signatures for ``data``, ``type``, ``newtype``, and ``class`` declarations:

```purescript
-- Defined in Type.Proxy
data Proxy :: forall k. k -> Type
data Proxy a = Proxy

proxy1 = Proxy :: Proxy Int -- k is Type
proxy2 = Proxy :: Proxy "foo" -- k is Symbol
proxy3 = Proxy :: Proxy (foo :: Int, bar :: String) -- k is Row Type
```

In type signatures, kind variables are quantified with ``forall`` like other type variables. Polymorphic kinds must be quantified before any kind annotated variables that reference them:

```purescript
proxy :: forall k (a :: k). Proxy a
proxy = Proxy
```

### Rank N Types

It is also possible for the ``forall`` quantifier to appear on the left of a function arrow, inside types record fields and data constructors, and in type synonyms.

In most cases, a type annotation is necessary when using this feature.

As an example, we can pass a polymorphic function as an argument to another function:

```purescript
poly :: (forall a. a -> a) -> Boolean
poly f = (f 0 < 1) == f true
```

Notice that the polymorphic function's type argument is instantiated to both ``Number`` and ``Boolean``.

An argument to ``poly`` must indeed be polymorphic. For example, the following fails:

```purescript
test = poly (\n -> n + 1)
```

since the rigid type variable ``a`` — that is, the type variable bound by the ``forall`` in the type signature for ``poly`` — does not unify with ``Int``.

### Tagged Unions

Tagged unions consist of one or more constructors, each of which takes zero or more arguments.

Tagged unions can only be created using their constructors, and deconstructed through pattern matching (a more thorough treatment of pattern matching will be provided later).

For example:

```purescript
data Foo = Foo | Bar String

runFoo :: Foo -> String
runFoo Foo = "It's a Foo"
runFoo (Bar s) = "It's a Bar. The string is " <> s

main = do
  log (runFoo Foo)
  log (runFoo (Bar "Test"))
```

In the example, Foo is a tagged union type which has two constructors. Its first constructor `Foo` takes no arguments, and its second `Bar` takes one, which must be a String.

`runFoo` is an example of pattern matching on a tagged union type to discover its constructor, and the last two lines show how to construct values of type `Foo`.

### Type Synonyms

For convenience, it is possible to declare a synonym for a type using the ``type`` keyword. Type synonyms can include type arguments but [cannot be partially applied](../errors/PartiallyAppliedSynonym.md#partiallyappliedsynonym-error). Type synonyms can be built with any other types but [cannot refer to each other in a cycle](../errors/CycleInTypeSynonym.md#cycleintypesynonym-error).

For example:

```purescript
-- Create an alias for a record with two fields
type Foo = { foo :: Number, bar :: Number }

-- Add the two fields together, since they are Numbers
addFoo :: Foo -> Number
addFoo o = o.foo + o.bar

-- Create an alias for a polymorphic record with the same shape
type Bar a = { foo :: a, bar :: a }
-- Foo is now equivalent to Bar Number

-- Apply the fields of any Bar to a function
combineBar :: forall a b. (a -> a -> b) -> Bar a -> b
combineBar f o = f o.foo o.bar

-- Create an alias for a complex function type
type Baz = Number -> Number -> Bar Number

-- This function will take two arguments and return a record with double the value
mkDoubledFoo :: Baz
mkDoubledFoo foo bar = { foo: 2.0*foo, bar: 2.0*bar }

-- Build on our previous functions to double the values inside any Foo
-- (Remember that Bar Number is the same as Foo)
doubleFoo :: Foo -> Foo
doubleFoo = combineBar mkDoubledFoo

-- Define type synonyms to help write complex Effect rows
-- This will accept further Effects to be added to the row
type RandomConsoleEffects eff = ( random :: RANDOM, console :: CONSOLE | eff )
-- This limits the Effects to just RANDOM and CONSOLE
type RandomConsoleEffect = RandomConsoleEffects ()
```

Unlike newtypes, type synonyms are merely aliases and cannot be distinguished from usages of their expansion. Because of this they cannot be used to declare a type class instance. For more see [``TypeSynonymInstance`` Error](../errors/TypeSynonymInstance.md#typesynonyminstance-error).

### Newtypes

Newtypes are like data types (which are introduced with the `data` keyword), but are restricted to a single constructor which contains a single argument. Newtypes are introduced with the `newtype` keyword:

```purescript
newtype Percentage = Percentage Number
```

The representation of a newtype at runtime is the same as the underlying data type. For example, a value of type `Percentage` is just a JavaScript number at runtime.

Newtypes are considered different from their underlying types by the type checker. For example, if you try to apply a function to a `Percentage` where it expects a `Number`, the type checker will reject your program.

Newtypes can be assigned their own type class instances, so for example, `Percentage` can be given its own `Show` instance:

```purescript
instance showPercentage :: Show Percentage where
  show (Percentage n) = show n <> "%"
```

### Polymorphic Types

Expressions can have polymorphic types:

```purescript
identity x = x
```

``identity`` is inferred to have (polymorphic) type ``forall t0. t0 -> t0``. This means that for any type ``t0``, ``identity`` can be given a value of type ``t0`` and will give back a value of the same type.

A type annotation can also be provided:

```purescript
identity :: forall a. a -> a
identity x = x
```

### Constrained Types

Polymorphic types may be predicated on one or more ``constraints``. See the chapter on type classes for more information.


### Row Polymorphism

> NOTE Either here or in "concepts", but probably there

Polymorphism is not limited to abstracting over types. Values may also be polymorphic in types with other kinds, such as rows or effects (see "[Kind System](#kind-system)").

For example, the following function accesses two properties on a record:

```purescript
addProps o = o.foo + o.bar + 1
```

The inferred type of ``addProps`` is:

```purescript
forall r. { foo :: Int, bar :: Int | r } -> Int
```

Here, the type variable ``r`` has kind ``Row Type`` - it represents a row of types. It can be instantiated with any row of named types.

In other words, ``addProps`` accepts any record which has properties ``foo`` and ``bar``, and *any other record properties*.

Therefore, the following application compiles:

```purescript
addProps { foo: 1, bar: 2, baz: 3 }
```

even though the type of ``addProps`` does not mention the property ``baz``. However, the following does not compile:

```purescript
addProps { foo: 1 }
```

since the ``bar`` property is missing.

