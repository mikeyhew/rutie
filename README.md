## Rutie

[![Build Status](https://travis-ci.org/danielpclark/rutie.svg?branch=master)](https://travis-ci.org/danielpclark/rutie)

Integrate Ruby with your Rust application.  Or integrate Rust with your Ruby application.
This project allows you to do either with relative ease.

You are highly encouraged to read the source code for this project.  Every method that has been
mapped from Ruby for public use in `src/class/*` is **very well documented** with example code.
This is the best way to take off running with Rutie.

This project is a continuation of [ruby-sys](https://github.com/steveklabnik/ruby-sys/) (licensed MIT) and [ruru](https://github.com/d-unseductable/ruru/) (licensed MIT).

## Using Ruby in Rust

First add the dependency to your `Cargo.toml` file.

```toml
[dependencies]
rutie = "0.2.0"
```

Then in your Rust program add `VM::init()` to the beginning of its code execution path
and begin to use Rutie.

```rust
extern crate rutie;

use rutie::{Object, RString, VM};

fn try_it(s: &str) -> String {
    let a = RString::new(s);

    // The `send` method returns an AnyObject type.
    let b = a.send("reverse", None);

    // We must try to convert the AnyObject
    // type back to our usable type.
    match b.try_convert_to::<RString>() {
        Ok(ruby_string) => ruby_string.to_string(),
        Err(_) => "Fail!".to_string(),
    }   
}

#[test]
fn it_works() {

    // Rust projects must start the Ruby VM
    VM::init();
    
    assert_eq!("selppa", try_it("apples"));
}

fn main() {}
```

Running `cargo test` should have this test pass.

## Using Rust in Ruby

You can start a Ruby project with `bundle gem rutie_ruby_example` and then once
you change into that directory run `cargo init`.  Remove the TODOs from the gemspec
file.  Add Rutie to the `Cargo.toml` file and define the lib type.

```toml
[dependencies]
rutie = "0.2.0"

[lib]
name = "rutie_ruby_example"
crate-type = ["dylib"]
```

Then edit your `src/lib.rs` file for your Rutie code.

```rust
#[macro_use]
extern crate rutie;

use rutie::{Class, Object, RString, VM};

class!(RutieExample);

methods!(
    RutieExample,
    _itself,

    fn pub_reverse(input: RString) -> RString {
        let ruby_string = input.
          map_err(|e| VM::raise_ex(e) ).
          unwrap();

        RString::new(
          &ruby_string.
          to_string().
          chars().
          rev().
          collect::<String>()
        )
    }
);

#[allow(non_snake_case)]
#[no_mangle]
pub extern "C" fn Init_rutie_ruby_example() {
    Class::new("RutieExample", None).define(|itself| {
        itself.def_self("reverse", pub_reverse);
    });
}
```

And that's it for the Rust side.  Now you just need to load the library in Ruby.  For typing
less code you may use [Thermite](https://github.com/malept/thermite) to handle the kind of library file that gets built for your
operating system.  Otherwise you'll need to load the library based on the operating system
with code similar to what follows here in your main ruby file `lib/rutie_ruby_example.rb`:

```ruby
require 'rutie_ruby_example/version'
require 'fiddle'

module RutieRubyExample
  module Platform
    class << self
      def ffi_library
        file = [lib_prefix,'rutie_ruby_example.',lib_suffix]

        File.join(rust_release, file.join())
      end

      def operating_system
        case host_os()
        when /linux|bsd|solaris/ then 'linux'
        when /darwin/ then 'darwin'
        when /mingw|mswin/ then 'windows'
        else host_os()
        end 
      end 

      def lib_prefix
        case operating_system()
        when /windows/ then ''
        when /cygwin/ then 'cyg'
        else 'lib'
        end 
      end 

      def lib_suffix
        case operating_system()
        when /darwin/ then 'dylib'
        when /linux/ then 'so'
        when /windows|cygwin/ then 'dll'
        else 'so'
        end 
      end 

      def rust_release
        File.expand_path('../target/release/', __dir__)
      end

      def host_os
        RbConfig::CONFIG['host_os'].downcase
      end
    end
  end

  LIBRARY = Platform.ffi_library()
  Fiddle::Function.
    new(Fiddle.dlopen(LIBRARY)['Init_rutie_ruby_example'], [], Fiddle::TYPE_VOIDP).
    call
end
```

That's all you need to load your Ruby things from Rust.  Now to write the test in
`test/rutie_ruby_example_test.rb`:

```ruby
require "test_helper"
        
class RutieRubyExampleTest < Minitest::Test
  def test_it_reverses
    assert_equal "selppa", RutieExample.reverse("apples")
  end   
end  
```

And to properly test it you will always need to run `cargo build --release` whenever
you make **any** changes to the Rust code.  Run the test with:

```bash
cargo build --release; rake test
```

Or better yet change your `Rakefile` to always run the `cargo build --release` before
every test suite run.  Feel free to change the test input to prove it fails because
the above test works as is.

## Custom Ruby Objects in Rust

To create a Ruby object in Rust that can be returned directly to Ruby
it needs just a few simple things.

Here's an example excerpt of code from [FasterPath](https://github.com/danielpclark/faster_path).

```rust
use rutie::types::{ Value, ValueType };
use rutie::{ RString, AnyObject, Object, Class, VerifiedObject };

pub struct Pathname {
    value: Value
}

impl Pathname {
    pub fn new(path: &str) -> Pathname {
        let mut instance = Class::from_existing("Pathname").allocate();
        instance.instance_variable_set("@path", RString::new(path).to_any_object());

        Pathname { value: instance.value() }
    }

    pub fn to_any_object(&self) -> AnyObject {
        AnyObject::from(self.value())
    }
}

impl From<Value> for Pathname {
    fn from(value: Value) -> Self {
        Pathname { value }
    }
}

impl Object for Pathname {
    #[inline]
    fn value(&self) -> Value {
        self.value
    }
}

impl VerifiedObject for Pathname {
    fn is_correct_type<T: Object>(object: &T) -> bool {
        object.value().ty() == ValueType::Class &&
          Class::from_existing("Pathname").case_equals(object)
    }

    fn error_message() -> &'static str {
        "Error converting to Pathname"
    }
}
```

If the class does not yet exist in Ruby you'll need to account for creating
it before generating a new instance of it.  This object is now compatible to
be returned into Ruby directly from Rust/Rutie.  _Note that this definition is
merely a Rust compatible representation of the Ruby object and doesn't define
any Ruby methods which can be used from Ruby._

## Variadic Functions / Splat Operator

A preferred way to integrate a dynamic amount of parameters has not yet been implemented in Rutie,
but you can still manage to get it done in the following way.

```rust
use rutie::{AnyObject, Array};
use rutie::types::{Argc, Value};
use rutie::util::str_to_cstring;
use rutie::rubysys::util;
use std::mem;

pub extern fn example_method(argc: Argc, argv: *const AnyObject, _: AnyObject) -> AnyObject {
    let args = Value::from(0);
  
    unsafe {
        let p_argv: *const Value = mem::transmute(argv);
  
        util::rb_scan_args(
            argc,
            p_argv,
            str_to_cstring("*").as_ptr(),
            &args
        )   
    };  
  
    let arguments = Array::from(args);

    let output = // YOUR CODE HERE.  Use arguments as you see fit.

    output.to_any_object() // When mapping a method to Ruby
                           // the return object needs to be AnyObject
}
```

This style of code is meant to be used outside of the `methods!` macro for now.
You may place this method on a class or module as you normally would from a `methods!` macro definition.

```rust
#[macro_use]
extern crate rutie;

use rutie::{Class, Object, VM};

class!(Example);

// Code from above

fn main() {
    # VM::init();
    Class::new("Example", None).define(|itself| {
        itself.def("example_method", example_method);
    });
}
```

The Rutie project has in its plans to remove the need for anyone to write unsafe code for
variadic support and will likely be updating the `methods!` macro to support this natively.

## Migrating from Ruru to Rutie

#### &lt;0.1

For using Rutie versions less than 0.1 the change is simple.  Replace all occurrences
of the string `ruru` with `rutie` in your program.  And if you would like to use
`ruby-sys` code from Rutie rather than requiring `ruby-sys` you can change all existing
references to `ruby_sys` to `rutie::rubysys`.

#### 0.1

You will have additional considerations to change like `Error` being removed.  For that; change instances of type `ruru::result::Error` to `rutie::AnyException`.

#### 0.2

Migrated `parse_arguments` from `VM` to `util`.

## Troubleshooting

#### rust signal: 11, SIGSEGV: invalid memory reference

This is an indication that you haven't started a Ruby VM in Rust yet with `VM::init();`.  Do this once
before using Ruby code from Rust.

#### error while loading shared libraries: libruby.so.#.#: cannot open shared object file: No such file or directory

This may happen when a Ruby program is trying to link with libruby via Rutie.  Simply disable linking
by setting the environment variable `NO_LINK_RUTIE` before the Rust code is compiled.  This is needed
to be done on the service TravisCI for example.

#### Calling methods from other methods within the `methods!` macro doesn't work

The way the macro is designed doesn't use the same parameter signatures you've provided and
therefore it is recommended to implement any methods you want to re-use in Rust with
functions outside of the `methods!` macro.  You can simply call that new external
method in the `methods!` macro when defining methods for Ruby to use.

#### Handling exceptions raised from Ruby in Rust code

If you're using any method that doesn't return a `Result<AnyObject, AnyException>` then
any exception raised from the Ruby side will interfere with that Ruby thread and cause
Rust to panic and stop.  Ruby internally uses exceptions to effect the entire thread through
an internal thread global value.  To handle places where Ruby may raise an exception during Rust 
code execution you should use methods that are designed to handle that.

* `VM::eval`
* `Object.protect_send`
* `Object.protect_public_send`

If you are writing lower level code and want to work more directly with the internal Ruby
exception you may use `VM::protect` and read the source code for `Object.protect_send` to
see how it's done.

## Additional Project History

If you need some more examples of usage or the git blame history please look at the [Ruru](https://github.com/d-unseductable/ruru)
project as Rutie has had the README completely rewritten and this first git commit is from Ruru.
Note that there are some fundamental changes which that README won't account for.
This project also had [ruby-sys](https://github.com/steveklabnik/ruby-sys) merged in which may have some additional beneficial git history.

## LICENSE

Both projects that were merged into this project contained identifiers under the MIT license.
This project follows with the same licensing.  **ruby-sys** marked MIT as the license in the
`Cargo.toml` file whereas **ruru** had that and included a LICENSE file.  This projects LICENSE
has credited the original author by preserving the MIT license author line and appending new
author(s) which is permitted by the MIT LICENSE.

MIT LICENSE — see [LICENSE](LICENSE)
