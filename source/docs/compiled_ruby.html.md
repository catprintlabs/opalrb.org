---
title: Compiled Ruby Code
---

## Generated Javascript

Opal is a source-to-source compiler, so there is no VM as such and the
compiled code aims to be as fast and efficient as possible, mapping
directly to underlying javascript features and objects where possible.

### Literals

```ruby
nil         # => nil
true        # => true
false       # => false
self        # => self
```

**self** is mostly compiled to `this`. Methods and blocks are implemented
as javascript functions, so their `this` value will be the right
`self` value. Class bodies and the top level scope use a `self` variable
to improve readability.

**nil** is compiled to a `nil` javascript variable. `nil` is a real object
which allows methods to be called on it. Opal cannot send methods to `null`
or `undefined`, and they are considered bad values to be inside ruby code.

**true** and **false** are compiled directly into their native boolean
equivalents. This makes interaction a lot easier as there is no need
to convert values to opal specific values.

<div class="opal-callout opal-callout-info">
Because <code>true</code> and <code>false</code> compile to their native
javascript equivalents, they must share the same class: <code>Boolean</code>.
For this reason, they do not belong to their respective <code>TrueClass</code>
and <code>FalseClass</code> classes from ruby.
</div>

#### Strings & Symbols

```ruby
"hello world!"    # => "hello world!"
:foo              # => "foo"
<<-EOS            # => "\nHello there.\n"
Hello there.
EOS
```

Ruby strings are compiled directly into javascript strings for
performance as well as readability. This has the side effect that Opal
does not support mutable strings - i.e. all strings are immutable.

<div class="opal-callout opal-callout-info">
  Strings in Opal are immutable because they are compiled into regular
  javascript strings. This is done for performance reasons.
</div>

For performance reasons, symbols are also compiled directly into strings.
Opal supports all the symbol syntaxes, but does not have a real `Symbol`
class. Symbols and Strings can therefore be used interchangeably.

#### Numbers

In Opal there is a single class for numbers; `Numeric`. To keep opal
as performant as possible, ruby numbers are mapped to native numbers.
This has the side effect that all numbers must be of the same class.
Most relevant methods from `Integer`, `Float` and `Numeric` are
implemented on this class.

```ruby
42        # => 42
3.142     # => 3.142
```

#### Arrays

Ruby arrays are compiled directly into javascript arrays. Special
ruby syntaxes for word arrays etc are also supported.

```ruby
[1, 2, 3, 4]        # => [1, 2, 3, 4]
%w[foo bar baz]     # => ["foo", "bar", "baz"]
```

#### Hash

Inside a generated ruby script, a function `__hash` is available which
creates a new hash. This is also available in javascript as `Opal.hash`
and simply returns a new instance of the `Hash` class.

```ruby
{ :foo => 100, :baz => 700 }    # => __hash("foo", 100, "baz", 700)
{ foo: 42, bar: [1, 2, 3] }     # => __hash("foo", 42, "bar", [1, 2, 3])
```

#### Range

Similar to hash, there is a function `__range` available to create
range instances.

```ruby
1..4        # => __range(1, 4, true)
3...7       # => __range(3, 7, false)
```

### Logic and conditionals

As per ruby, Opal treats only `false` and `nil` as falsy, everything
else is a truthy value including `""`, `0` and `[]`. This differs from
javascript as these values are also treated as false.

For this reason, most truthy tests must check if values are `false` or
`nil`.

Taking the following test:

```ruby
val = 42

if val
  return 3.142;
end
```

This would be compiled into:

```javascript
var val = 42;

if (val !== false && val !== nil) {
  return 3.142;
}
```

This makes the generated truthy tests (`if` statements, `and` checks and
`or` statements) a little more verbose in the generated code.

### Instance variables

Instance variables in Opal work just as expected. When ivars are set or
retrieved on an object, they are set natively without the `@` prefix.
This allows real javascript identifiers to be used which is more
efficient then accessing variables by string name.

```ruby
@foo = 200
@foo  # => 200

@bar  # => nil
```

This gets compiled into:

```javascript
this.foo = 200;
this.foo;   // => 200

this.bar;   // => nil
```

<div class="opal-callout opal-callout-info">
If an instance variable uses the same name as a reserved javascript keyword,
then the instance variable is wrapped using the object-key notation:
<code>this['class']</code>.
</div>

## Compiled Files

As described above, a compiled ruby source gets generated into a string
of javascript code that is wrapped inside an anonymous function. This
looks similar to the following:

```javascript
(function($opal) {
  var $klass = $opal.klass, self = $opal.top;
  // generated code
})(Opal);
```

As a complete example, assuming the following code:

```ruby
puts "foo"
```

This would compile directly into:

```javascript
(function($opal) {
  var $klass = $opal.klass, self = $opal.top;
  self.$puts("foo");
})(Opal);
```

Most of the helpers are no longer present as they are not used in this
example.

### Using compiled sources

If you write the generated code as above into a file `app.js` and add
that to your HTML page, then it is obvious that `"foo"` would be
written to the browser's console.

### Debugging and finding errors

Because Opal does not aim to be fully compatible with ruby, there are
some instances where things can break and it may not be entirely
obvious what went wrong.

### Using javascript debuggers

As opal just generates javascript, it is useful to use a native
debugger to work through javascript code. To use a debugger, simply
add an x-string similar to the following at the place you wish to
debug:

```ruby
# .. code
`debugger`
# .. more code
```
The x-strings just pass the debugger statement straight through to the
javascript output.

<div class="opal-callout opal-callout-info">
  All local variables and method/block arguments also keep their ruby
  names except in the rare cases when the name is reserved in javascript.
  In these cases, a <code>$</code> suffix is added to the name
  (e.g. <code>try</code> => <code>try$</code>).
</div>

## Javascript from Ruby

Opal tries to interact as cleanly with javascript and its api as much
as possible. Ruby arrays, strings, numbers, regexps, blocks and booleans
are just javascript native equivalents. The only boxed core features are
hashes.


### Inline Javascript

As most of the corelib deals with these low level details, opal provides
a special syntax for inlining javascript code. This is done with
x-strings or "backticks", as their ruby use has no useful translation
in the browser.

```ruby
`window.title`
# => "Opal: Ruby to Javascript compiler"

%x{
  console.log("opal version is:");
  console.log(#{ RUBY_ENGINE_VERSION });
}

# => opal version is:
# => 0.6.0
```

Even interpolations are supported, as seen here.

This feature of inlining code is used extensively, for example in
Array#length:

```ruby
class Array
  def length
    `this.length`
  end
end
```

X-Strings also have the ability to automatically return their value,
as used by this example.


### Native Module

_Reposted from: [Mikamayhem](http://dev.mikamai.com/post/79398725537/using-native-javascript-objects-from-opal)_

Opal standard lib (stdlib) includes a `Native` module, let’s see how it works and wrap `window`:

```ruby
require 'native'

window = Native(`window`) # equivalent to Native::Object.new(`window`)
```

Now what if we want to access one of its properties?

```ruby
window[:location][:href]                         # => "http://dev.mikamai.com/"
window[:location][:href] = "http://mikamai.com/" # will bring you to mikamai.com
```

And what about methods?

```ruby
window.alert('hey there!')
```

So let’s do something more interesting:

```ruby
class << window
  # A cross-browser window close method (works in IE!)
  def close!
    %x{
      return (#@native.open('', '_self', '') && #@native.close()) ||
             (#@native.opener = null && #@native.close()) ||
             (#@native.opener = '' && #@native.close());
    }
  end

  # let's assign href directly
  def href= url
    self[:location][:href] = url
  end
end
```

That’s all for now, bye!

```ruby
window.close!
```

## Ruby from Javascript

Accessing classes and methods defined in Opal from the javascript runtime is
possible via the `Opal` js object. The following class:

```ruby
class Foo
  def bar
    puts "called bar on class Foo defined in ruby code"
  end
end
```

Can be accessed from javascript like this:

```javascript
Opal.Foo.$new().$bar();
// => "called bar on class Foo defined in ruby code"
```

Remember that all ruby methods are prefixed with a `$`.

In the case that a method name can't be called directly due to a javascript syntax error, you will need to call the method using bracket notation. For example, you can call `foo.$merge(...)` but not `foo.$merge!(...)`, `bar.$fetch('somekey')` but not `bar.$[]('somekey')`. Instead you would write it like this: `foo['$merge!'](...)` or `bar['$[]']('somekey')`.


### Hash

Since ruby hashes are implemented directly with an Opal class, there's no "toll-free" bridging available (unlike with strings and arrays, for example). However, it's quite possible to interact with hashes from Javascript:

```javascript
var myHash = Opal.hash({a: 1, b: 2});
// output of $inspect: {"a"=>1, "b"=>2}
myHash.$store('a', 10);
// output of $inspect: {"a"=>10, "b"=>2}
myHash.$fetch('b','');
// 2
myHash.$fetch('z','');
// ""
myHash.$update(Opal.hash({b: 20, c: 30}));
// output of $inspect: {"a"=>10, "b"=>20, "c"=>30}
myHash.$to_n(); // provided by the Native module
// output: {"a": 10, "b": 20, "c": 30} aka a standard Javascript object
```

<div class="opal-callout opal-callout-info">
  Be aware <code>Hash#to_n</code> produces a duplicate copy of the hash.
</div>

## Advanced Compilation

### Method Missing

Opal supports `method_missing`. This is a key feature of ruby, and opal wouldn't be much use without it! This page details the implementation of `method_missing` for Opal.

#### Method dispatches

Firstly, a ruby call `foo.bar 1, 2, 3` is compiled into the following javascript:

```javascript
foo.$bar(1, 2, 3)
```

This should be pretty easy to read. The `bar` method has a `$` prefix just to distinguish it from underlying javascript properties, as well as ruby ivars. Methods are compiled like this to make the generated code really readable.

#### Handling `method_missing`

Javascript does not have an equivalent of `method_missing`, so how do we handle it? If a function is missing in javascript, then a language level exception will be raised.

To get around this, we make use of our compiler. During parsing, we collect a list of all method calls made inside a ruby file, and this gives us a list of all possible method calls. We then add stub methods to the root object prototype (an opal object, not the global javascript Object) which will proxy our method missing calls for us.

For example, assume the following ruby script:

```ruby
first 1, 2, 3
second "wow".to_sym
```

After parsing, we know we only ever call 3 methods: `[:first, :second, :to_sym]`. So, imagine we could just add these 3 methods to `BasicObject` in ruby, we would get something like this:

```ruby
class BasicObject
  def first(*args, &block)
    method_missing(:first, *args, &block)
  end

  def second(*args, &block)
    method_missing(:second, *args, &block)
  end

  def to_sym(*args, &block)
    method_missing(:to_sym, *args, &block)
  end
end
```

It is obvious from here, that unless an object defines any given method, it will always resort in a dispatch to `method_missing` from one of our defined stub methods. This is how we get `method_missing` in opal.

#### Optimising generated code

To optimise the generated code slightly, we reduce the code output from the compiler into the following javascript:

```javascript
Opal.add_stubs(["first", "second", "to_sym"]);
```

You will see this at the top of all your generated javascript files. This will add a stub method for all methods used in your file.

#### Alternative approaches

The old approach was to inline `method_missing` calls by checking for a method on **every method dispatch**. This is still supported via a parser option, but not recommended.

