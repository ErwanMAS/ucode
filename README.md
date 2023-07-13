# The ucode Scripting Language

The ucode language is a tiny general purpose scripting language featuring a
syntax closely resembling ECMAScript. It can be used in a stand-alone manner
by using the ucode command line interpreter or embedded into host applications
by linking libucode and utilizing its C language API. Additionally, ucode can
be invoked in template mode where control flow and expression logic statements
are embedded in Jinja-like markup blocks.

Besides aiming for small size, the major design goals of ucode are the ability
to trivially read and write JSON data, good embeddability into C applications,
template capabilities for output formatting, extensiblity through loadable
native extension modules and a straightforward set of built-in functions
mimicking those found in the Perl 5 language.

## History and Motivation

In spring 2021 it has been decided to rewrite the OpenWrt firewall framework on
top of nftables with the goal to replace the then current C application with a
kind of preprocessor generating nftables rulesets using a set of templates
instead of relying on built-in hardcoded rules like its predecessor.

That decision spurred the development of *ucode*, initially meant to be a
simple template processor solely for the OpenWrt nftables firewall but quickly
evolving into a general purpose scripting language suitable for a wider range
of system scripting tasks.

Despite OpenWrt predominantly relying on POSIX shell and Lua as system
scripting languages already, a new solution was needed to accomodate the needs
of the new firewall implementation; mainly the ability to efficiently deal with
JSON data and complex data structures such as arrays and dictionaries and the
ability to closely interface with OpenWrt's *ubus* message bus system.

Throughout the design process of the new firewall and its template processor,
the following design goals were defined for the *ucode* scripting language:

 - Ability to embed code logic fragments such as control flow statements,
   function calls or arithmetic expressions into plain text templates, using
   a block syntax and functionality roughly inspired by Jinja templates
 - Built-in support for JSON data parsing and serialization, without the need
   for external libraries
 - Distinct array and object types (compared to Lua's single table datatype)
 - Distinct integer and float types and guaranteed 64bit integer range
 - Built-in support for bit operations
 - Built-in support for (POSIX) regular expressions
 - A comprehensive set of built-in standard functions, inspired by the core
   functions found in the Perl 5 interpreter
 - Staying as close to ECMAScript syntax as possible due to higher developer
   familiarity and to be able to reuse existing tooling such as editor syntax
   highlighting
 - Bindings for all relevant Linux and OpenWrt APIs, such as *ubus*, *uci*,
   *uloop*, *netlink* etc.
 - Procedural, synchronous programming flow
 - Very small executable size (the interpreter and runtime is currently around
   64KB on ARM Cortex A9)
 - Embeddability into C host applications

Summarized, *ucode* can be described as synchronous ECMAScript without the
object oriented standard library.


## Installation

### OpenWrt

In OpenWrt 22.03 and later, *ucode* should already be preinstalled. If not,
it can be installed via the package manager, using the `opkg install ucode`
command.

### MacOS

To build on MacOS, first install *cmake* and *json-c* via
[Homebrew](https://brew.sh/), then clone the ucode repository and execute
*cmake* followed by *make*:

    $ brew install cmake json-c
    $ git clone https://github.com/jow-/ucode.git
    $ cd ucode/
    $ cmake -DUBUS_SUPPORT=OFF -DUCI_SUPPORT=OFF -DULOOP_SUPPORT=OFF .
    $ make
    $ sudo make install

### Debian

The ucode repository contains build recipes for Debian packages, to build .deb
packages for local installation, first install required development packages,
then clone the repository and invoke *dpkg-buildpackage* to produce the binary
package files:

    $ sudo apt-get install build-essential devscripts debhelper libjson-c-dev cmake pkg-config
    $ git clone https://github.com/jow-/ucode.git
    $ cd ucode/
    $ dpkg-buildpackage -b -us -uc
    $ sudo dpkg -i ../ucode*.deb ../libucode*.deb

### Other Linux systems

To install ucode from source on other systems, ensure that the json-c library
and associated development headers are installed, then clone and compile the
ucode repository:

    $ git clone https://github.com/jow-/ucode.git
    $ cd ucode/
    $ cmake -DUBUS_SUPPORT=OFF -DUCI_SUPPORT=OFF -DULOOP_SUPPORT=OFF .
    $ make
    $ sudo make install


## Syntax

### Template mode

By default, *ucode* is executed in *raw mode*, means it expects a given source
file to only contain script code. By invoking the ucode interpreter with the
`-T` flag or by using the `utpl` alias, the *ucode* interpreter is switched
into *template mode* where the source file is expected to be a plaintext file
containing *template blocks* containing ucode script expressions or comments.

#### Block types

There are three kinds of blocks; *expression blocks*, *statement blocks* and
*comment blocks*. The former two embed code logic using ucode's JavaScript-like
syntax while the latter comment block type is simply discarded during
processing.


##### 1. Statement block

Statement blocks are enclosed in an opening `{%` and a closing `%}` tag and
may contain any number of script code statements, even entire programs.

It is allowed to omit the closing `%}` of a statement block to parse the
entire remaining source text after the opening tag as ucode script.

By default, statement blocks produce no output and the entire block is
reduced to an empty string during template evaluation but contained script
code might invoke functions such as `print()` to explicitly output contents.

For example the following template would result in `The epoch is odd` or
`The epoch is even`, depending on the current epoch value:

`The epoch is {% if (time() % 2): %}odd{% else %}even{% endif %}!`


##### 2. Expression block

Expression blocks are enclosed in an opening `{{` and a closing `}}` tag and
may only contain a single expression statement (multiple expressions may be
chained with comma). The implicit result of the rightmost evaluated expression
is used as output when processing the block.

For example the template `Hello world, {{ getenv("USER") }}!` would result in
the output "Hello world, user!" where `user` would correspond to the name of
the current user executing the ucode interpreter.


##### 3. Comment block

Comment blocks, which are denoted with an opening `{#` and a closing `#}` tag
may contain arbitrary text except the closing `#}` tag itself. Comments blocks
are completely stripped during processing and are replaced with an empty string.

The following example template would result in the output "Hello world":

`Hello {# mad #}word`


#### Whitespace handling

Each block start tag may be suffixed with a dash to strip any whitespace
before the block and likewise any block end tag may be prefixed with a dash
to strip any whitespace following the block.

Without using whitespace stripping, the following example:

```
This is a first line
{% for (x in [1, 2, 3]): %}
This is item {{ x }}.
{% endfor %}
This is the last line
```

Would result in the following output:

```
This is a first line

This is item 1.
This is item 2.
This is item 3.

This is the last line
```

By adding a trailing dash to apply whitespace stripping after the block, the
empty lines can be eliminated:

```
This is a first line
{% for (x in [1, 2, 3]): -%}
This is item {{ x }}.
{% endfor -%}
This is the last line
```

Output:

```
This is a first line
This is item 1.
This is item 2.
This is item 3.
This is the last line
```

By applying whitespace stripping before the block, all lines can be joined
into a single output line:

```
This is a first line
{%- for (x in [1, 2, 3]): -%}
This is item {{ x }}.
{%- endfor -%}
This is the last line
```

Output:

```
This is a first lineThis is item 1.This is item 2.This is item 3.This is the last line
```

### Script syntax

The ucode script language - used either within statement and expression blocks
or throughout the entire file in *raw mode*, uses untyped variables and employs
a simplified JavaScript like syntax.

The language implements function scoping and differentiates between local and
global variables. Each function has its own private scope while executing and
local variables declared inside a function are not accessible in the outer
calling scope.

#### 1. Data types

Ucode supports seven different basic types as well as two additional special
types; function values and ressource values. The supported types are:

 - Boolean values (`true` or `false`)
 - Integer values (`-9223372036854775808` to `+9223372036854775807`)
 - Double values (`-1.7e308` to `+1.7e308`)
 - String values (e.g. `'Hello world!'` or `"Sunshine \u2600!"`)
 - Array values (e.g. `[1, false, "foo"]`)
 - Object values (e.g. `{ foo: true, "bar": 123 }`)
 - Null value (`null`)

Ucode utilizes reference counting to manage memory used for variables and values
and frees data automatically as soon as values go out of scope.

Numeric values are either stored as signed 64bit integers or as IEEE 756 double
value. Conversion between integer and double values can happen implicitly, e.g.
through numeric operations, or explicitely, e.g. by invoking functions such as
`int()`.

#### 2. Variables

Variable names must start with a letter or an underscore and may only contain
the characters `A`..`Z`, `a`..`z`, `0`..`9` or `_`. By prefixing a variable
name with the keyword `let`, it is declared in the local block scope only
and not visible outside anymore.

Variables may also be declared using the `const` keyword. Such variables follow
the same scoping rules as `let` declared ones but they cannot be modified after
they have been declared. Any attempt to do so will result in a syntax error
during compilation.

```javascript
{%

  a = 1;  // global variable assignment

  function test() {
    let b = 2;  // declare `b` as local variable
    a = 2;        // overwrite global a
  }

  test();

  print(a, "\n");  // outputs "2"
  print(b, "\n");  // outputs nothing

  const c = 3;
  print(c, "\n");  // outputs "3"

  c = 4;           // raises syntax error
  c++;             // raises syntax error

  const d;         // raises syntax error, const variables must
                   // be initialized at declaration time

%}
```

#### 3. Control statements

Similar to JavaScript, ucode supports `if`, `for` and `while` statements to
control execution flow.

##### 3.1. Conditional statement

If/else blocks can be used to execute statements depending on a condition.

```javascript
{%

  user = getenv("USER");

  if (user == "alice") {
      print("Hello Alice!\n");
  }
  else if (user == "bob") {
      print("Hello Bob!\n");
  }
  else {
      print("Hello guest!\n");
  }

%}
```

If only a single statement is wrapped by an if or else branch, the enclosing
curly braces may be omitted:

```javascript
{%

  if (rand() == 3)
      print("This is quite unlikely\n");

%}
```

##### 3.2. Loop statements

Ucode script supports three different flavors of loop control statements; a
`while` loop that executes enclosed statements as long as the loop condition is
fulfilled, a `for in` loop that iterates keys of objects or items of arrays and
a counting `for` loop that is a variation of the `while` loop.

```javascript
{%

  i = 0;
  arr = [1, 2, 3];
  obj = { Alice: 32, Bob: 54 };

  // execute as long as condition is true
  while (i < length(arr)) {
      print(arr[i], "\n");
      i++;
  }

  // execute for each item in arr
  for (n in arr) {
      print(n, "\n");
  }

  // execute for each key in obj
  for (person in obj) {
      print(person, " is ", obj[person], " years old.\n");
  }

  // execute initialization statement (j = 0) once
  // execute as long as condition (j < length(arr)) is true
  // execute step statement (j++) after each iteration
  for (j = 0; j < length(arr); j++) {
      print(arr[j], "\n");
  }

%}
```

##### 3.3. Alternative syntax

Since conditional statements and loops are often used for template formatting
purposes, e.g. to repeat a specific markup for each item of a list, ucode
supports an alternative syntax that does not require curly braces to group
statements but that uses explicit end keywords to denote the end of the control
statement body for better readability instead.

The following two examples first illustrate the normal syntax, followed by the
alternative syntax that is more suitable for statement blocks:

```
Printing a list:
{% for (n in [1, 2, 3]) { -%}
  - Item #{{ n }}
{% } %}
```

The alternative syntax replaces the opening curly brace (`{`) with a colon
(`:`) and the closing curly brace (`}`) with an explicit `endfor` keyword:

```
Printing a list:
{% for (n in [1, 2, 3]): -%}
  - Item #{{ n }}
{% endfor %}
```

For each control statement type, a corresponding alternative end keyword is defined:

  - `if (...): ... endif`
  - `for (...): ... endfor`
  - `while (...): ... endwhile`


#### 4. Functions

Ucode scripts may define functions to group repeating operations into reusable
operations. Functions can be both declared with a name, in which case they're
automatically registered in the current scope, or anonymously which allows
assigning the resulting value to a variable, e.g. to build arrays or objects of
functions:

```javascript
{%

  function duplicate(n) {
       return n * 2;
  }

  let utilities = {
      concat: function(a, b) {
          return "" + a + b;
      },
      greeting: function() {
          return "Hello, " + getenv("USER") + "!";
      }
  };

-%}

The duplicate of 2 is {{ duplicate(2) }}.
The concatenation of 'abc' and 123 is {{ utilities.concat("abc", 123) }}.
Your personal greeting is: {{ utilities.greeting() }}.
```

##### 4.1. Alternative syntax

Function declarations support the same kind of alternative syntax as defined
for control statements (3.3.)

The alternative syntax replaces the opening curly brace (`{`) with a colon
(`:`) and the closing curly brace (`}`) with an explicit `endfunction`
keyword:

```
{% function printgreeting(name): -%}
  Hallo {{ name }}, nice to meet you.
{% endfunction -%}

<h1>{{ printgreeting("Alice") }}</h1>
```


#### 5. Operators

Similar to JavaScript and C, ucode scripts support a range of different
operators to manipulate values and variables.

##### 5.1. Arithmetic operations

The operators `+`, `-`, `*`, `/`, `%`, `++` and `--` allow to perform
additions, substractions, multiplications, divisions, modulo, increment or
decrement operations respectively where the result depends on the type of
involved values.

The `++` and `--` operators are unary, means that they only apply to one
operand. The `+` and `-` operators may be used in unary context to either
convert a given value to a numeric value or to negate a given value.

If either operand of the `+` operator is a string, the other one is converted
to a string value as well and a concatenated string is returned.

All other arithmetic operators coerce their operands into numeric values.
Fractional values are converted to doubles, other numeric values to integers.

If either operand is a double, the other one is converted to a double value as
well and a double result is returned.

Divisions by zero result in the special double value `Infinity`. If an operand
cannot be converted to a numeric value, the result of the operation is the
special double value `NaN`.

```javascript
{%
  a = 2;
  b = 5.2;
  s1 = "125";
  s2 = "Hello world";

  print(+s1);      // 125
  print(+s2);      // NaN
  print(-s1);      // -125
  print(-s2);      // NaN
  print(-a);       // -2

  print(a++);      // 2 (Return value of a, then increment by 1)
  print(++a);      // 4 (Increment by 1, then return value of a)

  print(b--);      // 5.2 (Return value of b, then decrement by 1)
  print(--b);      // 3.2 (Decrement by 1, then return value of b)

  print(4 + 8);    // 12
  print(7 - 4);    // 3
  print(3 * 3);    // 9

  print(10 / 4);   // 2 (Integer division)
  print(10 / 4.0); // 2.5 (Double division)
  print(10 / 0);   // Infinity

  print(10 % 7);   // 3
  print(10 % 7.0); // NaN (Modulo is undefined for non-integers)
%}
```

##### 5.2. Bitwise operations

The operators `&`, `|`, `^`, `<<`, `>>` and `~` allow to perform bitwise and,
or, xor, left shift, right shift and complement operations respectively.

The `~` operator is unary, means that is only applies to one operand.

```javascript
{%
  print(0 & 0, 0 & 1, 1 & 1);  // 001
  print(0 | 0, 0 | 1, 1 | 1);  // 011
  print(0 ^ 0, 0 ^ 1, 1 ^ 1);  // 010
  print(10 << 2);              // 40
  print(10 >> 2);              // 2
  print(~15);                  // -16 (0xFFFFFFFFFFFFFFF0)
%}
```

An important property of bitwise operators is that they're coercing their
operand values to whole integers:

```javascript
{%
  print(12.34 >> 0);   // 12
  print(~(~12.34));    // 12
%}
```

##### 5.3. Relational operations

The operators `==`, `!=`, `<`, `<=`, `>` and `>=` test whether their operands
are equal, inequal, lower than, lower than/equal to, higher than or higher
than/equal to each other respectively.

If both operands are strings, their respective byte values are compared, if
both are objects or arrays, their underlying memory addresses are compared.

In all other cases, both operands are coerced into numeric values and the
resulting values are compared with each other.

This means that comparing values of different types will coerce them both to
numbers.

The result of the relational operation is a boolean indicating truishness.

```javascript
{%
  print(123 == 123);     // true
  print(123 == "123");   // true!
  print(123 < 456);      // true
  print(123 > 456);      // false
  print(123 != 456);     // true
  print(123 != "123");   // false!
  print({} == {});       // false (two different anonymous objects)
  a = {}; print(a == a); // true (same object)
%}
```

##### 5.4. Logical operations

The operators `&&`, `||`, `??` and `!` test whether their operands are all true,
partially true, null or false respectively.

In the case of `&&` the rightmost value is returned while `||` results in the
first truish and `??` in the first non-null value.

The unary `!` operator will result in `true` if the operand is not trueish,
otherwise it will result in `false`.

Operands are evaluated from left to right while testing truishness, which means
that expressions with side effects, such as function calls, are only executed
if the preceeding condition was satisifed.

```javascript
{%
  print(1 && 2 && 3);                 // 3
  print(1 || 2 || 3);                 // 1
  print(2 > 1 && 3 < 4);              // true
  print(doesnotexist ?? null ?? 42);  // 42
  print(1 ?? 2 ?? 3);                 // 1
  print(!false);                      // true
  print(!true);                       // false

  res = test1() && test2();  // test2() is only called if test1() returns true
%}
```

##### 5.5. Assignment operations

In addition to the basic assignment operator `=`, most other operators have a
corresponding shortcut assignment operator which reads the specified variable,
applies the operation and operand to it, and writes it back.

The result of assignment expressions is the assigned value.

```javascript
{%
  a = 1;     // assign 1 to variable a
  a += 2;    // a = a + 2;
  a -= 3;    // a = a - 3;
  a *= 4;    // a = a * 4;
  a /= 5;    // a = a / 5;
  a %= 6;    // a = a % 6;
  a &= 7;    // a = a & 7;
  a |= 8;    // a = a | 8;
  a ^= 9;    // a = a ^ 9;
  a <<= 10;  // a = a << 10;
  a >>= 11;  // a = a >> 11;
  a &&= 12;  // a = a && 12;
  a ||= 13;  // a = a || 13;
  a ??= 14;  // a = a ?? 14;

  print(a = 2);  // 2
%}
```

##### 5.6. Miscellaneous operators

Besides the operators described so far, ucode script also supports a `delete`
operator which removes a property from an object value.

```javascript
{%
  a = { test: true };

  delete a.test;         // true
  delete a.notexisting;  // false

  print(a);              // { }
%}
```

##### 5.7. Precedence

Operator precedence determines the order in which operators are evaluated in an
expression. In ucode, operators have different precedence levels, as outline
in the table below.

| Precedence | Operator type                     | Associativity  |
|------------|-----------------------------------|----------------|
| 19         | Grouping `( … )`                  | n/a            |
| 18         | Property access `… . …`           | left-to-right  |
| 18         | Optional chaining `… ?. …`        | left-to-right  |
| 18         | Computed propery access `… [ … ]` | n/a            |
| 18         | Function call `… (…)`             | n/a            |
| 17         | Postfix increment `… ++`          | n/a            |
| 17         | Postfix decrement `… --`          | n/a            |
| 16         | Logical not `! …`                 | n/a            |
| 16         | Bitwise not `~ …`                 | n/a            |
| 16         | Unary plus `+ …`                  | n/a            |
| 16         | Unary negation `- …`              | n/a            |
| 16         | Prefix increment `++ …`           | n/a            |
| 16         | Prefix decrement `-- …`           | n/a            |
| 16         | Property deletion `delete …`      | n/a            |
| 15         | Exponentiation `… ** …`           | right-to-left  |
| 14         | Multiplication `… * …`            | left-to-right  |
| 14         | Division `… / …`                  | left-to-right  |
| 14         | Remainder `… % …`                 | left-to-right  |
| 13         | Addition `… + …`                  | left-to-right  |
| 13         | Substraction `… - …`              | left-to-right  |
| 12         | Bitwise left shift `… << …`       | left-to-right  |
| 12         | Bitwise right shift `… >> …`      | left-to-right  |
| 11         | Less than `… < …`                 | left-to-right  |
| 11         | Less than or equal `… <= …`       | left-to-right  |
| 11         | Greater than `… > …`              | left-to-right  |
| 11         | Greater than or equal `… >= …`    | left-to-right  |
| 11         | In `… in …`                       | left-to-right  |
| 10         | Equality `… == …`                 | left-to-right  |
| 10         | Inequality `… != …`               | left-to-right  |
| 10         | Strict equality `… === …`         | left-to-right  |
| 10         | Strict inequality `… !== …`       | left-to-right  |
| 9          | Bitwise AND `… & …`               | left-to-right  |
| 8          | Bitwise XOR `… ^ …`               | left-to-right  |
| 7          | Bitwise OR `… | …`                | left-to-right  |
| 6          | Logical AND `… && …`              | left-to-right  |
| 5          | Logical OR `… || …`               | left-to-right  |
| 5          | Nullish coalescing `… ?? …`       | left-to-right  |
| 4          | Assignment `… = …`                | right-to-left  |
| 4          | Assignment `… += …`               | right-to-left  |
| 4          | Assignment `… -= …`               | right-to-left  |
| 4          | Assignment `… **= …`              | right-to-left  |
| 4          | Assignment `… *= …`               | right-to-left  |
| 4          | Assignment `… /= …`               | right-to-left  |
| 4          | Assignment `… %= …`               | right-to-left  |
| 4          | Assignment `… <<= …`              | right-to-left  |
| 4          | Assignment `… >>= …`              | right-to-left  |
| 4          | Assignment `… &= …`               | right-to-left  |
| 4          | Assignment `… ^= …`               | right-to-left  |
| 4          | Assignment `… |= …`               | right-to-left  |
| 4          | Assignment `… &&= …`              | right-to-left  |
| 4          | Assignment `… ||= …`              | right-to-left  |
| 4          | Assignment `… ??= …`              | right-to-left  |
| 3          | Ternary `… ? … : …`               | right-to-left  |
| 2          | Arrow `… => …`                    | right-to-left  |
| 2          | Spread `... …`                    | n/a            |
| 1          | Sequence `… , …`                  | left-to-right  |

Operators with a higher precedence value are evaluated before operators with a
lower precedence value. When operators have the same precedence, their
associativity determines the order of evaluation
(e.g., left-to-right or right-to-left).


## Memory Management

The ucode scripting language utilizes a reference count-based garbage collector
as its primary method of memory management. When assigning an array or object
value, the reference count is incremented. When a local variable holding a
reference goes out of scope, the reference count is decremented. If the
reference count reaches zero, a recursive traversal is performed to decrease the
reference count of any nested references. Once the traversal is complete, the
top-level array or object structure is freed.

Example 1:
```javascript
x = [ { a: 1 }, { b: 2 }, { c: 3 } ];
// `x` holds a reference to `[...]` (refcount 1)
// `x[0]` holds a reference to `{ a: 1 }` (refcount 1)
// `x[1]` holds a reference to `{ b: 2 }` (refcount 1)
// `x[2]` holds a reference to `{ c: 3 }` (refcount 1)

x = null;
// refcount of `[...]` drops to 0; refcount decreasing cascades
// down, `{ a: 1 }`, `{ b: 2 }` and { c: 3 }` refcounts reach
// zero as well; `{ a: 1 }`, `{ b: 2 }`, `{ c: 3 }` and `[ ... ]`
// are freed
```

Example 2:
```javascript
x = [ { a: 1 }, { b: 2 }, { c: 3 } ];
y = x[1];
// `x` holds a reference to `[...]` (refcount 1)
// `x[0]` holds a reference to `{ a: 1 }` (refcount 1)
// `x[1]` and `y` hold a reference to `{ b: 2 }` (refcount 2)
// `x[2]` holds a reference to `{ c: 3 }` (refcount 1)

x = null;
// refcount of `[...]` drops to 0, refcount decreasing cascades
// down, `{ a: 1 }` and `{ c: 3 }` refcounts reach zero while
// `{ b: 2 }` refcount is down to one.
// `{ a: 1 }`, `{ c: 3 }` and `[ ... ]` are freed
// `{ b: 2 }` is still alive with refcount 1, pointed to by `y`
```

Although the reference count-based garbage collector efficiently manages memory,
it cannot handle cyclic structures, leading to memory leaks.

Example 1:
```javascript
o = { }; o.x = o;
// `o` holds a reference to itself (refcount 1)

```

Example 2:
```javascript
a = [ ]; a[0] = a;
// `a` holds a reference to itself (refcount 1)
```

Example 3:
```javascript
x = { y: { z: [ ] } }; x.y.z = x;
// `x` holds a reference to itself through `x.y.z` (refcount 1)
```

In these examples, cyclic references are created where objects or arrays point
back to themselves or create a circular chain. Since each element within the
cycle maintains a reference, the reference count for each object or array never
reaches zero, resulting in a memory leak. The reference count-based garbage
collector cannot automatically reclaim memory in such cases.

To address cyclic structures and avoid memory leaks, ucode provides a secondary
mark-and-sweep garbage collector. This collector can be enabled by passing the
`-g` flag to the ucode interpreter or manually triggered using the
[`gc()`](/module-core.html#gc) function during runtime. The mark-and-sweep
collector identifies and frees unreachable objects, including those involved in
cyclic references.
