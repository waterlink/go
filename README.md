# The Go Programming Language

This is a fork of https://github.com/golang/go which tries to go into different
direction. Not intended to be merged back, because it goes against Go core
design.

Go is an open source programming language that makes it easy to build simple,
reliable, and efficient software.

## How is it different?

Or goals of this project:

### Global type inference

*Not implemented yet*

Since everything is inferred, operator `:=` is the same as `=`.

Type annotations in method signatures are optional:

```go
func sumAndProduct(a, b) (sum, prod) {
    return a + b, a * b
}

// Any types that has appropriate + and * implementation fit this method
// signature:
// sumAndProduct(3, 4) => {7, 12}
// sumAndProduct(3.1, 7) => {10.1, 21.7}
// sumAndProduct(dna{"AGGTA"}, dna{AATGT}) => {dna{"AGGTAAATGT"}, dna{"AGGGT"}}
```

### Compile time execution

*Not implemented yet*

Example which implements some sort of DSL:

```go
dependency("john.smith/example", "master")
dependency("john.smith/example_generator", "1.2")
dependency("john.smith/resources", "2.pre")
```

And code that makes use of it:

```go
func dependencies() (depsList) {
    // init dependency list
    depsList = makeEmptyDepsList()

    // create DSL function
    dependency = func(name, version) {
        depsList.add(makeDependency(name, version))
    }

    // at compile time (preprocessor stage):
    // read file contents into string and inject it in this method
    {{ string(ioutil.ReadFile("Dependencyfile")) }}

    // return depsList
    return
}
```

### Macros

*Not implemented yet*

Very expressive mocking example:

```go
// original version:
example.greet("world")    // => "hello, world!"

// with mocking
m.allow(example).toReceive(greet("world") => "hi, world!")
example.greet("world")    // => "hi, world!"
```

Where `toReceive` is in fact a macro:

```go
macro (allow mocks.Allow) toReceive(messages) {
    {{allow}}.register({
        // here messages is not a map, but MapAstNode
        // key is CallAstNode and value is arbitrary AstNode
        {% for key, value = range messages %} 
            mocks.MockedMessage{
                message: {{key.method_name.stringify}},
                arguments: {{key.args}},
                response: {{value}},
            },
        {% endfor %}
    })
}
```

The example will expand to:

```go
m.allow(example).register({
    mocks.MockedMessage{
        message: "greet",
        arguments: ["world"],
        response: "hi, world!",
    },
})
```

These macros do not manipulate strings, but Abstract syntax tree instead. Which
makes them extremely powerful, more expressive and less error-prone.

As you can see in the example up above, macros are scoped under
types/interfaces.  So no polluting of global scope.

### Union types

*Not implemented yet*

This is very good improvement to the infering engine. Consider this example:

```go
someVar = "hello"
// at this point, compiler infers someVar to be a string

someVar = 37.5
// but at this point, instead of compile error, you get
// something different: string|float32
// which makes it possible to do this:

someArray = {"hello", 37.5, "world"}
append(someArray, LeafClover{4})
// now you have an array of type [](string|float32|LeafClover)
```

### Generics

*Not implemented yet*

Since most of the problems are already taken care of by global type inference
and union types, generics are rather simple and they are mostly automatically
inferred too. Simple example:

```go
type EqualityExpectation(T) struct {
    Expected T,
    Actual,
}

EqualityExpectation{"hello, world!"}.match(example.greet("world"))   // => true
EqualityExpectation{"hello, world!"}.match(example.greet(95))        // => false

eq = EqualityExpectation{"test"}
eq.match(77)
eq.Actual             // => 77
eq.FailureMessage     // => `Expected 77:int32 to be equal to "test":string`
eq.(type)             // => EqualityExpectation(int32)
```

### Error handling

*Not implemented yet*

Instead of returning tuple of 2 values, just return `Result` entity:

```go
func iMayFail(value) {
    if iDontLikeIt(value) {
        return result.Error("I dont like this value :(")
    }

    return result.Ok(42)
}

// simple usage
iMayFail("hello")    // => Error("I dont like this value :(")
iMayFail("world")    // => Ok(42)

// awesome usage
func doSomething() {
    return iMayFail("hello").ifOk({
        // return is implicit here
        iMayFail("john").sureOk()
    }).sureOk()
}
```

`ifOk` is a macro, that accepts block. `Ok`'s `isOk` executes provided block.
`Error`'s `isOk` ignores provided block and returns itself.

`ifError` is an inversion of `ifOk`.

`sureOk` is a method. It returns value that `Ok` holds. Or panics with what
`Error` holds.

`sureError` is an inversion of `sureOk`. But it panics with descriptive
message, containing `Ok`'s value.

Referring the value `Result` holds inside of `ifOk` and `ifError`:

```go
iMayFail("hello").ifOk(func(x) {
    x + 7
})
```

If panic occurs inside of `isOk` and `isError` blocks, then they will return
`Error` with panic argument inside. Otherwise, they return `Ok` with what the
block returned. Which makes such chaining possible:

```go
func updateSettings(key, value) {
    return FsApi.open("config.json").ifOk(func(f) {
        f.readContents()

    }).ifOk(func(contents) {
        json.Decode(contents)

    }).ifOk(func(config) {
        json.Encode(config.update(key, value))

    }).ifOk(func(contents) {
        f.writeContents(contents)
    })
}
```

If any step fails, then all next calls to `ifOk` will be ignored and the
`Error` that happened at this step will eventually return from `updateSettings`
function. Otherwise, it will return `Ok` with value returned from the last
step.

### Result adapter

*Not implemented yet*

This example shows how to deal with any external code, that was written for
original `result, error := ...` design:

```go
result.adapt(someLib.someFunc(someArgs))     // => Ok(42)
result.adapt(someLib.someFunc(otherArgs))    // => Error("resource is unavailable")
```

`result.adapt` is a macro, that just fetches both values (`result` and `error`)
from the function result, and returns `Ok{result}` if error is `null` and
`Error{error}` otherwise.

To deal with `result, ok := ...` design use result.adaptOk(value, error):

```go
// it is Ok(result) if ok and Error("unable to execute otherFunc") if not ok 
result.adaptOk(someLib.otherFunc(), "unable to execute otherFunc")
```

## Building from source

*TODO*

## License

Unless otherwise noted, the Go source files are distributed under the BSD-style
license found in the LICENSE file.
