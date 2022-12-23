# New Macro Transition

The old macro system used a global state to be able to automatically register
functions and classes when the `#[php_module]` attribute is used. However,
global state can cause problems with incremental compilation and is not
recommended.

To solve this, the macro system has been re-written but this will require
changes to user code. This document summarises the changes.

## Functions

Mostly unchanged in terms of function definition, however you now need to
register the function with the module builder:

```rs
use ext_php_rs::prelude::*;

#[php_function]
pub fn hello_world() -> &'static str {
    "Hello, world!"
}

#[php_module]
pub fn get_module(module: ModuleBuilder) -> ModuleBuilder {
    module
        .function(wrap_function!(hello_world))
}
```

## Classes

Mostly unchanged in terms of the class and impl definitions, however you now
need to register the classes with the module builder:

```rs
use ext_php_rs::prelude::*;

#[php_class]
#[derive(Debug)]
pub struct TestClass {
    #[prop]
    a: i32,
    #[prop]
    b: i32,
}

#[php_class]
#[extends(TestClass::get_class_entry)]
pub struct ExampleClassThatInheritsTestClass {}

#[php_impl]
impl TestClass {
    #[rename("NEW_CONSTANT_NAME")]
    pub const SOME_CONSTANT: i32 = 5;
    pub const SOME_OTHER_STR: &'static str = "Hello, world!";

    pub fn __construct(a: i32, b: i32) -> Self {
        Self { a: a + 10, b: b + 10 }
    }

    #[optional(test)]
    #[defaults(a = 5, test = 100)]
    pub fn test_camel_case(&self, a: i32, test: i32) {
        println!("a: {} test: {}", a, test);
    }

    fn x(&self) -> i32 {
        5
    }

    pub fn builder_pattern(
        this: &mut ZendClassObject<TestClass>,
    ) -> &mut ZendClassObject<TestClass> {
        dbg!(this)
    }
}

#[php_module]
pub fn get_module(module: ModuleBuilder) -> ModuleBuilder {
    module
        .class::<TestClass>()
}
```

## Constants

The `#[php_const]` attribute has been deprecated. Register the constant
directly with the module builder:

```rs
use ext_php_rs::prelude::*;

const SOME_CONSTANT: i32 = 100;

#[php_module]
pub fn get_module(module: ModuleBuilder) -> ModuleBuilder {
    module
        .constant(wrap_constant!(SOME_CONSTANT)) // SOME_CONSTANT = 100
        .constant(("CONST_NAME", SOME_CONSTANT)) // CONST_NAME = 100
}
```

## Extern

No changes.

```rs
use ext_php_rs::prelude::*;

#[php_extern]
extern "C" {
    fn phpinfo() -> bool;
}

fn some_rust_func() {
    let x = unsafe { phpinfo() };
    println!("phpinfo: {x}");
}
```

## Startup Function

The `#[php_startup]` macro has been deprecated. Instead, define a function with
the signature `fn(ty: i32, mod_num: i32) -> i32` and provide the function name
to the `#[php_module]` attribute:

```rs
use ext_php_rs::prelude::*;

fn startup(ty: i32, mod_num: i32) -> i32 {
    // register extra classes, constants etc
    5.register_constant("SOME_CONST", mod_num);
    0
}

#[php_module(startup = "startup")]
pub fn get_module(module: ModuleBuilder) -> ModuleBuilder {
    module
}
```

The arguments passed to the function are:

- `ty` - The type of the module
  - If 1, the module was loaded as "persistent" like a regular extension
  - If 2, the module was loaded temporarily such as via the [`dl()`] method
    (very uncommon)
- `mod_num` - The module number used as an identifier by the interpreter. This
  number increments for each extension loaded

[`dl()`]: https://www.php.net/manual/en/function.dl.php
