# HHVM Extension Writing, [Part I](http://blog.golemon.com/2015/01/hhvm-extension-writing-part-i.html) #

I've written a number of blogposts and even one book over the years on writing extensions for PHP, but very little documentation is available for writing HHVM extensions.  This is kinda sad since I built a good portion of the latter's API. Let's fix that, starting with this article.

All the code in this post (and its followups) will be found at https://github.com/sgolemon/hhvm-extension-writing.


## Setting up a build environment ##

The first thing you need to do is get all the dependencies in place.  I'm going to start from a clean install of Ubuntu 14.04 LTS and use the prebuilt HHVM binaries for Ubuntu. Other distros should work (with varying degrees of success), but sorting them all out is beyond the scope of this blog entry.

First, let's trust HHVM's package repo and pull in its package list. Then we can install the hhvm-dev package to pull in the binary along with all needed headers.

```console
$ wget -O - http://dl.hhvm.com/conf/hhvm.gpg.key | \
  sudo apt-key add -
$ echo deb http://dl.hhvm.com/ubuntu trusty main | \
  sudo tee /etc/apt/sources.list.d/hhvm.list
$ sudo apt-get update

$ sudo apt-get install hhvm-dev
```


## Creating an extension skeleton ##

The most basic, no-nothing extension imaginable requires two files. A C++ source file to declare itself, and a config.cmake file to describe what's being built. Let's start with the build file, which is a simple, single line:

```cpp
HHVM_EXTENSION(example1 ext_example1.cpp)
```

This macro declares a new extension named "example1" with a single source file named "ext_example1.cpp". If we had multiple source files, we'd delimit them with a space (`HHVM_EXTENSION(example1 ext_example1.cpp ex1lib.cpp utilex1.cpp etc.cpp)`)

The source file has a little more boilerplate, but fortunately it's also just a handful of lines:

```cpp
#include "hphp/runtime/base/base-includes.h"

namespace HPHP {

class Example1Extension : public Extension {
 public:
  Example1Extension(): Extension("example1", "1.0") {}
} s_example1_extension;

HHVM_GET_MODULE(example1);

} // namespace HPHP
```

All we're doing here is exposing a specialization of the "Extension" class which gives itself the name "example1". It doesn't do anything more than declare itself into the runtime environment. Those familiar with PHP extension development can think of this as the zend_module_entry struct, with all callbacks and the function table set to NULL.

The code so far is at commit: https://github.com/sgolemon/hhvm-extension-writing/tree/214e2e7be60747abef89fa13e5922ddf9b1ad60d


## Building an extension and testing it out ##

To build an extension, first run `hphpize` to generate a `CMakeLists.txt` file, then `cmake`. to generate a `Makefile` from that. Finally, issue make to actually build it. You should see output like the following:

```console
$ hphpize
** hphpize complete, now run 'cmake . && make` to build
$ cmake .
-- Configuring for HHVM API version 20140829
-- Configuring done
-- Generating done
-- Build files have been written to: /home/username/hhvm-ext-writing
$ make
Scanning dependencies of target example1
[100%] Building CXX object CMakeFiles/example1.dir/ext_example1.cpp.o
[100%] Built target example1
```

Now we're ready to load it into our runtime. Start by creating a simple test file:

```Hack
<?php
var_dump(extension_loaded('example1.php'));
```

Then fire up hhvm:

```console
hhvm -d extension_dir=. -d hhvm.extensions[]=example1.so tests/loaded.php
```

and you should see bool(true).


## Adding functionality ##

The simplest way to add functionality is to write some Hack code. You could write straight PHP code, but you'll see in a few moments why Hack is preferable for extension systemlibs. Let's introduce a new file: ext_example1.php and link it into our project:

```Hack
<?hh

function example1_hello() {
  echo "Hello World\n";
}
```

Then load it in during the `moduleInit()` (aka MINIT) phase:

```cpp
class Example1Extension : public Extension {
 public:
  Example1Extension(): Extension("example1", "1.0") {}
  void moduleInit() override {
    loadSystemlib();
  }
} s_example1_extension;
```

And finally, add the following to your `config.cmake` file to embed it into the .so, where HHVM can load it from at runtime.

```cpp
HHVM_EXTENSION(example1 ext_example1.cpp)
HHVM_SYSTEMLIB(example1 ext_example1.php)
```

Rebuild your extension according to the instructions above, then try it out:

```console
$ hhvm -d extension_dir=. -d hhvm.extensions[]=example1.so tests/hello.php
Hello World
```

The code so far is at commit: https://github.com/sgolemon/hhvm-extension-writing/tree/54782f157df7b30a8596e85b8bdf9dc2632fa22d


## Bridging the gap ##

If all you wanted to do was write PHP code implementations you could create a normal library for that. Extensions are for bridging PHP-script into native code, so let's do that. Make a new entry in your systemlib file using some hack specific syntax:

```Hack
<<__Native>>
function example1_greet(string $name, bool $hello = true): void;
```

The `<<__Native>> UserAttribute` tells HHVM that this is the declaration for an internal function. The hack types tell the runtime what C++ type to pair them with, and the usual rules for default arguments apply.

To pair it with an internal implementation, we'll add the following to ext_example1.cpp:

```Hack
void HHVM_FUNCTION(example1_greet, const String& name, bool hello) {
  g_context->write(hello ? "Hello " : "Goodbye ");
  g_context->write(name);
  g_context->write("\n");
}
```

And link it to the systemlib by adding `HHVM_FE(example1_greet);` to `moduleInit()`.

As you can see, internal functions are declared with the `HHVM_FUNCTION()` macro where the first arg is the name of the function, as exposed to userspace, and the remaining map to the userspace functions argument signature. The argument types map according the following table:

Hack type  | C++ type (argument) | C++ type (return type)
-----------|---------------------|-----------------------
void       | N/A                 | void
bool       | bool                | bool
int        | int64_t             | int64_t
float      | double              | double
string     | const String&       | String
array      | const Array&        | Array
resource   | const Resource&     | Resource
object     | const Object&       | Object
ClassName  | const Object&       | Object
mixed      | const Variant&      | Variant
mixed&     | VRefParam           | N/A

Since this is Hack syntax, you may declare the types as soft (with an `@`) or nullable (with a question mark), but since these types are not limited to a primitive, they need to be represented internally as the more generic const `Variant&` for arguments or `Variant` or return types (essentially, mixed).

Reference arguments use the VRefParam type noted above. An example of which can be seen below:

```Hack
<<__Native>>
function example1_life(mixed &$meaning): void;
void HHVM_FUNCTION(example1_life, VRefParam meaning) {
  meaning = 42;
}
```

The code so far is at commit: https://github.com/sgolemon/hhvm-extension-writing/tree/df16aca35ee00cf6f84b2a3dd870582cdbdd1de5


## Constants ##

Constants, like any other bit of PHP, may be declared in the systemlib file, or if they depend on some native value (such as a define from an external library), they may be declare in `moduleInit()` using the `Native::registerConstant()` template as with the following:
`const StaticString s_EXAMPLE1_YEAR("EXAMPLE1_YEAR")`;

```cpp
class Example1Extension: public Extension {
 public:
  Example1Extension(): Extension("example1", "1.0") {}
  void moduleInit() override {
    Native::registerConstant<KindOfInt64>(s_EXAMPLE1_YEAR.get(), 2015);
  }
} s_example1_extension;
```

The use of this function should be mostly obvious, in that it takes the name of a constant as a `StringData*` (which comes from a `StaticString`'s `.get()` accessor), and a value appropriate to the constant's type. The type, in turn, is given as the function's template parameter and is one of the DataType enum values. The kinds correspond roughly to the basic PHP data types.

DataType           | C++ type
-------------------|---------
KindOfNull         | N/A
KindOfBoolean      | bool
KindOfInt64	       | int64_t
KindOfDouble       | double
KindOfStaticString | StringData*

The code so far is at commit: https://github.com/sgolemon/hhvm-extension-writing/tree/ad9618ac8cff194a2e01a2bf9d59d0ae5ec93939

