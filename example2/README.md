# HHVM Extension Writing, [Part II](http://blog.golemon.com/2015/01/hhvm-extension-writing-part-ii.html) #

In our [last installment](http://blog.golemon.com/2015/01/hhvm-extension-writing-part-i.html) I walked through setting up a dev environment and creating a simple HHVM extension which exposed some constants and global scope functions. Today, we'll expand on that by delving deeper into three of the five "Smart" types: 
[String](https://github.com/facebook/hhvm/blob/master/hphp/runtime/base/type-string.h),
[Array](https://github.com/facebook/hhvm/blob/master/hphp/runtime/base/type-array.h),
and [Variant](https://github.com/facebook/hhvm/blob/master/hphp/runtime/base/type-variant.h). The other two "Smart" types will be covered in [Parts III(Objects)](http://blog.golemon.com/2015/01/hhvm-extension-writing-part-iii.html) and [Part IV(Resources)](http://blog.golemon.com/2015/01/hhvm-extension-writing-part-iv.html) since they require a bit more explaining.

All code in the following examples can be found at https://github.com/sgolemon/hhvm-extension-writing and we'll be starting from where [Part I](http://blog.golemon.com/2015/01/hhvm-extension-writing-part-i.html) left off: commit https://github.com/sgolemon/hhvm-extension-writing/tree/ad9618ac8cff194a2e01a2bf9d59d0ae5ec93939.


## The String class ##

[HPHP::String](https://github.com/facebook/hhvm/blob/master/hphp/runtime/base/type-string.h) resembles C++'s `std::string` class in many ways, but also builds in several assumptions about how PHP strings should behave, is able to be encapsulated in a Variant (mixed) object, and performs common string related tasks, such as numeric conversion.

This post is going to highlight the most common features of the String class, but you should look through the header file yourself for a more in-depth exploration.

```cpp
/* Basic inspection */
class String {
 public:
  const char* c_str() const;
  int size() const;
  bool empty() const { return size() == 0; }
  int length() const ( return size(); }
  bool isNumeric() const;
  bool isInteger() const;
  bool isZero() const;
  bool toBoolean() const;
  char toByte() const;
  short toInt16() const;
  int toInt32() const;
  int64_t toInt64() const;
  double toDouble() const;
  std::string toCppString() const;

  char charAt(int pos) const;
  char operator[](int pos) const;
};
```

The meaning and use of these methods should all be straightforward. In practice, `c_str()`, `size()`, and `empty()` are going to cover 90% of your uses for reading values from the String class.

```cpp
/* Creation */
class String {
 public:
  String(); // empty string
  String(const char* cstr);
  String(const std::string& cppstr);
  String(const String& hphpstr);
  String(int64_t num);
  String(double num);

  static StaticString FromCStr(const char* cstr);

  String(size_t cap, ReserveStringMode mode);
  MutableSlice bufferSlice();
  uint32_t capacity() const;
  const String& setSize(int len);
};
```

The constructors, as you can see, are generally built around making new runtime string values from an existing string or numeric value, and are again straight-forward to use. `String::FromCStr()` is a somewhat special case in that it creates a StaticString, rather than a String. While a String is cleaned up at the end of the request it was created in, StaticStrings live forever, and can even be shared between multiple requests. Because overuse of `StaticString` could easily lead to memory bloat, they're typically only used for defining persistent features (such as constant names/values) as seen in Part I.

The most interesting part of this API is the `ReserveStringMode` and `MutableSlice`. Ordinarily, you shouldn't save the pointer you get from `String::c_str()` as it can potentially change between calls, and you generally shouldn't go modifying a `String` unless you know you own it anyway. If you do have need to modify a string, call `bufferSlice()` on it. The `MutableSlice` structure you get back will contain a pointer to a (relatively) stable block of memory which can be populated. Here's an example:

```cpp
String HHVM_FUNCTION(example1_count_preallocate) {
  /* 30 bytes: 3 per number: 'X, ' */
  String ret(30, ReserveString);
  auto slice = ret.bufferSlice();
  for (int i = 0; i < 10; ++i) {
    snprintf(slice.ptr + (i*3), 4, "%d, ", i);
  }
  /* Terminate just after the 9th digit, overwriting the ',' with a null byte */
  return ret.setSize((9*3) + 1);
}
```

This contrived example allocates enough space for 10 single-digit numbers, and a comma and space following them. It uses `snprintf()` to fill that buffer up, then it truncates it as 28 characters, since the final '`, `' wasn't actually necessary. You'll find this pattern in use anywhere an API expects you to provide it with a buffer for it to fill, such as in the intl extension where it calls into ICU.

Another approach to building up a string from parts would be to use the operator+ overload which allows you to simply concatenate Strings such as in the following:

```cpp
String HHVM_FUNCTION(example1_count_concatenate) {
  String ret, delimiter(", ");
  for (int i = 0; i < 10; ++i) {
    if (i > 0) {
      ret += delimiter;
    }
    ret += String(i);
  }
  return ret;
}
```

There are costs and benefits to both versions. The former is more efficient as it only does one allocation, as opposed to the latter which does at least 11, and far less copying around. On the other hand, the second version is far more readable and far less error prone. For the contrived example, I'd call the second version "better", but there are certainly cases where the first version is superior.
The code so far is at commit: https://github.com/sgolemon/hhvm-extension-writing/tree/fa82b3cd70c625b2790c920d95824ff83d7281ae


## The Array Class ##

Arrays are the do-all bucket of "stuff" of the PHP language. They can behave like vectors, maps, sets, or weird hybrid hodgepodge containers without rhyme or reason. You already know how to interact with them from userspace, so let's take a look at how to interact with them from C++. As with Strings, we're only going to go into the most common API calls here, check out the [header](https://github.com/facebook/hhvm/blob/master/hphp/runtime/base/type-array.h) for the full story.

```cpp
/* Core API */
class Array {
 public:
  static Array Create(); // array()
  static Array Create(const Variant& value); // array($value)
  static Array Create(const Variant& key, const Variant& value); // array($key => $value)

  /* Read */
  const Variant operator[](int64_t key) const;
  const Variant operator[](const String& key) const;
  const Variant operator[](const Variant& key) const;

  /* count($arr) */
  ssize_t count() const;

  /* array_key_exists($arr, $key); */
  bool exists(int64_t key) const;
  bool exists(const String& key, bool isKey = false) const;
  bool exists(const Variant& key, bool isKey = false) const;

  /* Write */
  void clear();

  /* $arr[$key] = $v; */
  void set(int64_t key, const Variant& v);
  void set(const String& key, const Variant& v, bool isKey = false);
  void set(const Variant& key, const Variant& v, bool isKey = false);

  void prepend(const Variant& v); // array_unshift($v);
  Variant dequeue();              // array_shift($v);
  void append(const Variant& v);  // array_push($v); aka => $arr[] = $v;
  Variant pop();                  // array_pop($v);

  /* $arr[$key] =& $v; */
  void setRef(int64_t key, const Variant& v);
  void setRef(const String& key, const Variant& v, bool isKey = false);
  void setRef(const Variant& key, const Variant& v, bool isKey = false);

  /* $arr[] =& $v; */
  void appendRef(Variant& v);

  /* unset($arr[$key]); */
  void remove(int64_t key);
  void remove(const String& key, bool isKey = false);
  void remove(const Variant& key);
};
```

As you can see, the `Array` APIs mirror PHP's userspace API very closely, down to the read API using square-bracket notation just like PHP code. Let's write a couple new methods dealing with arrays as arguments and return values.

```cpp
const StaticString
  s_name("name"),
  s_hello("hello"),
  s_Stranger("Stranger");

void HHVM_FUNCTION(example1_greet_options, const Array& options) {
  String name(s_Stranger);
  if (options.exists(s_name)) {
    name = options[s_name].toString();
  }
  bool hello = true;
  if (options.exists(s_hello)) {
    hello = options[s_hello].toBoolean();
  }
  g_context->write(greet ? "Hello " : "Goodbyte ");
  g_context->write(name);
  g_context->write("\n");
}

Array HHVM_FUNCTION(example1_greet_make_options, const String& name, bool hello) {
  Array ret = Array::Create();
  if (!name.empty()) {
    ret.set(s_name, name);
  }
  ret.set(s_hello, hello);
  return ret;
}
```

Pretty similar syntax to writing PHP code, yeah?

The code so far is at commit: https://github.com/sgolemon/hhvm-extension-writing/tree/3966bb1da16421cb653d15d38986be0a8098ea41


## The Variant Class ##

The last "smart" class doesn't represent a single PHP type, rather it represents all types in a sort of meta-container which knows what it's holding, and knows how to convert between the concrete types. `Variant` is useful when you need to accept and/or return multiple possible types. For a start, let's list out the core API. Remember that there are far more methods than I'll cover here, and you can find the reset in the header file.

```cpp
/* Creation/Assignment */
class Variant {
 public:
  Variant();
  Variant(bool bval);
  Variant(int64_t lval);
  Variant(double dval);
  Variant(const String& strval);
  Variant(const char* cstrval);
  Variant(const Array& arrval);
  Variant(const Resource& resval);
  Variant(const Object& objval);
  Variant(const Variant& val);

  template Variant &operator=(const T &v);
}
```

These APIs together mean that a `Variant` may be initialized or assigned from any other variable type supported by userspace code. This becomes especially powerful when looking at Variant return types.

```cpp
Variant HHVM_FUNCTION(example1_password, const String& guess) {
  if (guess.same(s_secret)) {
    return "Password accepted: A winner is you!";
  }
  return false;
}
```

These seemingly incompatible return types (`const char*` and `bool`) work because they are implicitly constructed into a `Variant` instance. Explicit types are generally preferred, because the IR can make better assumptions during optimization, but sometimes you just want your return values to be adaptable like that.

```cpp
/* Introspection and Unboxing */
class Variant {
 public:
  bool isNull() const;
  bool isBoolean() const;
  bool isInteger() const;
  bool isDouble() const;
  bool isNumeric(bool checkString = false) const;
  bool isString() const;
  bool isArray() const;
  bool isResource() const;
  bool isObject() const;

  bool toBoolean() const;
  int64_t toInt64() const;
  double toDouble() const;
  DataType toNumeric(int64_t &ival, double &dval, bool checkString = false) const;
  String toString() const;
  Array toArray() const;
  Resource toResource() const;
  Object toObject() const;
}
```

These APIs allow pulling a concrete data type out of a Variant so they can be operated on directly. Note that the `to*()` APIs will convert the type if necessary, even if the `is*()` call returned false, but that not all conversions make sense. Let's make a contrived example by implementing a simplistic `var_dump()`:

```Hack
<<__Native>>
function example1_var_dump(mixed $value): void;

void HHVM_FUNCTION(example1_var_dump, const Variant &value) {
  if (value.isNull()) {
    g_context->write("null\n");
    return;
  }
  if (value.isBoolean()) {
    g_context->write("bool(");
    g_context->write(value.toBoolean() ? "true" : "false");
    g_context->write(")\n");
    return;
  }
  if (value.isInteger()) {
    g_context->write("int(");
    g_context->write(String(value.toInt64()));
    g_context->write(")\n");
    return;
  }
  // etc...
}
```

The code so far is at commit: https://github.com/sgolemon/hhvm-extension-writing/tree/8e82e3e416be6ca358e496e78927feaba7519647
