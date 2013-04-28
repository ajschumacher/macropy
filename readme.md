MacroPy
=======

**MacroPy** is an implementation of [Macros](http://tinyurl.com/cmlls8v) in the [Python Programming Language](http://python.org/). MacroPy provides a mechanism for user-defined functions (macros) to perform transformations on the [abstract syntax tree](http://en.wikipedia.org/wiki/Abstract_syntax_tree)(AST) of Python code at _module import time_. This is an easy way to modify the semantics of a python program, and has been used to implement features such as:

- [Quasiquotes](#quasiquotes), a quick way to manipulate fragments of a program
- [String Interpolation](#string-interpolation), a common feature in many languages
- [Tracing](#tracing) and [Smart Asserts](#smart-asserts)
- [Case Classes](#case-classes), easy [Algebraic Data Types](https://en.wikipedia.org/wiki/Algebraic_data_type) from Scala
- [Pattern Matching](#pattern-matching) from the Functional Programming world
- [LINQ to SQL](#linq-to-sql) from C#
- [Quick Lambdas](#quick-lambdas) from Scala and Groovy,

All of these are advanced language features that each would have been a massive effort to implement in the [CPython](http://en.wikipedia.org/wiki/CPython) interpreter. Using macros, the implementation of each feature fits in a single file, often taking less than 40 lines of code.

*MacroPy is very much a work in progress, for the [MIT](http://web.mit.edu/) class [6.945: Adventures in Advanced Symbolic Programming](http://groups.csail.mit.edu/mac/users/gjs/6.945/). Although it is constantly in flux, all of the examples with source code represent already-working functionality. The rest will be filled in over the coming weeks.*

Rough Overview
--------------
Macro functions are defined in two ways:

```python
@expr_macro
def my_expr_macro(tree):
    ...
    return new_tree

@block_macro
def my_block_macro(tree):
    ...
    return new_tree
```

These two types of macros are called via

```python
val = my_expr_macro%(...)

with my_block_macro:
    ...
```

Any time either of these syntactic forms is seen, if a matching macro exists, the abstract syntax tree captured by these forms (the `...` in the code above) is given to the respective macro to handle. The macro can then return a new tree, which is substituted into the original code in-place.

MacroPy intercepts the module-loading workflow, via the functionality provided by [PEP 302: New Import Hooks](http://www.python.org/dev/peps/pep-0302/). The workflow is roughly:

- Intercept an import
- Parse the contents of the file into an AST
- walk the AST and expand any macros that it finds
- unparse the AST back into a string and resume loading it as a module

Below are a few example uses of macros that are implemented (together with test cases!) in the [macropy/macros](macropy/macros) folder.

Quasiquotes
-----------

```python
a = 10
b = 2
tree = q%(1 + u%(a + b))
print tree
#BinOp(Num(1), Add(), Num(12))
```

Quasiquotes are the foundation for many macro systems, such as that found in [LISP](http://en.wikipedia.org/wiki/LISP). Quasiquotes save you from having to manually construct code trees from the nodes they are made of. For example, if you want the code tree for 

```python
(1 + 2)
```

Without quasiquotes, you would have to build it up by hand:

```python
tree = BinOp(Num(1), Add(), Num(2))
```

But with quasiquotes, you can simply write the code `(1 + 2)`, quoting it with `q%` to lift it from an expression (to be evaluated) to a tree (to be returned):

```python
tree = q%(1 + 2)
```

Furthermore, quasiquotes allow you to _unquote_ things: if you wish to insert the **value** of an expression into the tree, rather than the **tree** making up the expression, you unquote it using `u%`. In the example above:

```python
tree = q%(1 + u%(a + b))
```

the expression `(a + b)` is unquoted. Hence `a + b` gets evaluated to the value of `12`, which is then inserted into the tree, giving the final tree:

```python
print tree
#BinOp(Num(1), Add(), Num(12))
```

String Interpolation
--------------------

```python
a, b = 1, 2
c = s%"%{a} apple and %{b} bananas"
print c
#1 apple and 2 bananas
```

Unlike the normal string interpolation in Python, MacroPy's string interpolation allows the programmer to specify the variables to be interpolated _inline_ inside the string. The macro `s%` then takes the string literal

```python
"%{a} apple and %{b} bananas"
```

and expands it into the expression

```python
"%s apple and %s bananas" % (a, b)
```

Which is evaluated at run-time in the local scope, using whatever the values `a` and `b` happen to hold at the time. The contents of the `%{...}` can be any arbitrary python expression, and is not limited to variable names.

Tracing
-------

```python
log%(1 + 2)
#(1 + 2) -> 3

log%("omg" * 3)
#('omg' * 3) -> 'omgomgomg'
```

Tracing allows you to easily see what is happening inside your code. Many a time programmers have written code like

```python
print "value", value
print "sqrt(x)", sqrt(x)
```

and the `log%` macro (shown above) helps remove this duplication by automatically expanding `log%(1 + 2)` into `wrap("(1 + 2)", (1 + 2))`. `wrap` then evaluates the expression, printing out the source code and final value of the computation.

In addition to simple logging, MacroPy provides the `trace%` macro. This macro not only logs the source and result of the given expression, but also the source and result of all sub-expressions nested within it:

```python
trace%[len(x)*3 for x in ["omg", "wtf", "b" * 2 + "q", "lo" * 3 + "l"]]
#('b' * 2) -> 'bb'
#(('b' * 2) + 'q') -> 'bbq'
#('lo' * 3) -> 'lololo'
#(('lo' * 3) + 'l') -> 'lololol'
#['omg', 'wtf', (('b' * 2) + 'q'), (('lo' * 3) + 'l')] -> ['omg', 'wtf', 'bbq', 'lololol']
#len(x) -> 3
#(len(x) * 3) -> 9
#len(x) -> 3
#(len(x) * 3) -> 9
#len(x) -> 3
#(len(x) * 3) -> 9
#len(x) -> 7
#(len(x) * 3) -> 21
#[(len(x) * 3) for x in ['omg', 'wtf', (('b' * 2) + 'q'), (('lo' * 3) + 'l')]] -> [9, 9, 9, 21]
```

As you can see, `trace%` logs the source and value of all sub-expressions that get evaluated in the course of evaluating the list comprehension.

Lastly, `trace` can be used as a block macro:


```python
with trace:
    sum = 0
    for i in range(0, 5):
        sum = sum + 5

    square = sum * sum
#sum = 0
#for i in range(0, 5):
#   sum = (sum + 5)
#range(0, 5) -> [0, 1, 2, 3, 4]
#sum = (sum + 5)
#(sum + 5) -> 5
#sum = (sum + 5)
#(sum + 5) -> 10
#sum = (sum + 5)
#(sum + 5) -> 15
#sum = (sum + 5)
#(sum + 5) -> 20
#sum = (sum + 5)
#(sum + 5) -> 25
#square = (sum * sum)
#(sum * sum) -> 625
```

Used this way, `trace` will print out the source code of every _statement_ that gets executed, in addition to tracing the evaluation of any expressions within those statements.

Smart Asserts
-------------
```python
require%(3**2 + 4**2 != 5**2)
#AssertionError: Require Failed
#(3 ** 2) -> 9
#(4 ** 2) -> 16
#((3 ** 2) + (4 ** 2)) -> 25
#(5 ** 2) -> 25
#(((3 ** 2) + (4 ** 2)) != (5 ** 2)) -> False
```

MacroPy provides a variant on the `assert` keyword called `require%`. Like `assert`, `require%` throws an `AssertionError` if the condition is false.

Unlike `assert`, `require%` automatically tells you what code failed the condition, and traces all the sub-expressions within the code so you can more easily see what went wrong. Pretty handy!

`require% can also be used in block form:

```python
a = 10
b = 2
with require:
    a > 5
    a * b == 20
    a < 2
#AssertionError: Require Failed
#(a < 2) -> False
```

This requires every statement in the block to be a boolean expression. Each expression will then be wrapped in a `require%`, throwing an `AssertionError` with a nice trace when a condition fails.


Case Classes
------------
```python
@case
class Point(x, y): pass

p = Point(1, 2)

print str(p)    #Point(1, 2)
print p.x       #1
print p.y       #2
print Point(1, 2) == Point(1, 2)
#True
```

[Case classes](http://www.codecommit.com/blog/scala/case-classes-are-cool) are classes with extra goodies:

- A nice `__str__` method is autogenerated
- An autogenerated constructor
- Structural equality by default

The reasoning being that although you may sometimes want complex, custom-built classes with custom features and fancy inheritance, very (very!) often you want a simple class with a constructor, pretty `__str__` and `__repr__` methods, and structural equality which doesn't inherit from anything. Case classes provide you just that, with an extremely concise declaration:

```python
@case
class Point(x, y): pass
```

As opposed to the equivalent class, written manually:

```python
class Point(object):
    def __init__(self, x, y):
        self.x = x
        self.y = y

    def __str__(self):
        return "Point(" + self.x + ", " + self.y + ")"

    def __repr__(self):
        return self.__str__()

    def __eq__(self, other):
        return self.x == other.x and self.y == other.y

    def __ne__(self, other):
        return not self.__eq__(other)
```

Whew, what a lot of boilerplate! This is clearly a pain to do, error prone to deal with, and violates [DRY](http://en.wikipedia.org/wiki/Don't_repeat_yourself) in an extreme way: each member of the class (`x` and `y` in this case) has to be repeated _6_ times, with loads and loads of boilerplate. Given how tedious writing all this code is, it is no surprise that most python classes do not come with proper `__str__` or useful `__eq__` functions! With case classes, there is no excuse, since all this will be generated for you.

Case classes also provide a convenient copy-constructor, which creates a shallow copy of the case class with modified fields, leaving the original unchanged:

```python
a = Point(1, 2)
b = a.copy(x = 3)
print a #Point(1, 2)
print b #Point(3, 2)
```

Like any other class, a case class may contain methods in its body:

```python
@case
class Point(x, y):
    def length(self):
        return (self.x ** 2 + self.y ** 2) ** 0.5

print Point(3, 4).length() #5
```

or class variables. The only restrictions are that only the `__init__`, `__repr__`, `___str__`, `__eq__` methods will be set for you, and it may not manually inherit from anything. Instead of manual inheritence, inheritence for case classes is defined by _nesting_, as shown below:

```python
@case
class List():
    def __len__(self):
        return 0

    def __iter__(self):
        return iter([])

    class Nil:
        pass

    class Cons(head, tail):
        def __len__(self):
            return 1 + len(self.tail)

        def __iter__(self):
            current = self

            while len(current) > 0:
                yield current.head
                current = current.tail

print isinstance(Cons(None, None), List)    # True
print isinstance(Nil(), List)               # True

my_list = Cons(1, Cons(2, Cons(3, Nil())))
empty_list = Nil()

print my_list.head              # 1
print my_list.tail              # Cons(2, Cons(3, Nil()))
print len(my_list)              # 5
print sum(iter(my_list))        # 6
print sum(iter(empty_list))     # 0
```

This is an implementation of a singly linked [cons list](http://en.wikipedia.org/wiki/Cons), providing both `head` and `tail` (LISP's `car` and `cdr`) as well as the ability to get the `len` or `iter` for the list.

As the classes `Nil` are `Cons` are nested within `List`, both of them get transformed into top-level classes which inherit from it. This nesting can go arbitrarily deep.

Pattern Matching
----------------
*Work-In-Progress*

LINQ to SQL
-----------
```python
print sql%(
    x.name for x in bbc
    if x.gdp / x.population > (
        y.gdp / y.population for y in bbc
        if y.name == 'United Kingdom'
    ) and x.region == 'Europe'
)
#SELECT name FROM bbc
#WHERE gdp/population > (
#    SELECT gdp/population FROM bbc
#    WHERE name = 'United Kingdom'
#)
#AND region = 'Europe'
```

This feature is inspired by [C#'s LINQ to SQL](http://msdn.microsoft.com/en-us/library/bb386976.aspx). In short, code used to manipulate lists is lifted into an AST which is then cross-compiled into a snippet of [SQL](http://en.wikipedia.org/wiki/SQL).

This allows you to write queries to a database in the same way you would write queries on in-memory lists. *WIP*

Quick Lambdas
-------------
```python
map(f%(_ + 1), [1, 2, 3])
#[2, 3, 4]

reduce(f%(_ + _), [1, 2, 3])
#6
```

Macropy provides a syntax for lambda expressions similar to Scala's [anonymous functions](http://www.codecommit.com/blog/scala/quick-explanation-of-scalas-syntax). Essentially, the transformation is:

```python
f%(_ + _) -> lambda a, b: a + b
```

where the underscores get replaced by identifiers, which are then set to be the parameters of the enclosing `lambda`. This works too:

```python
map(f%_.split(' ')[0], ["i am cow", "hear me moo"])
#["i", "hear"]
```

Quick Lambdas can be also used as a concise, lightweight, more-readable substitute for `functools.partial`

```python
import functools
basetwo = functools.partial(int, base=2)
basetwo('10010')
#18
```

is equivalent to

```python
basetwo = f%int(_, base=2)
basetwo('10010')
#18
```

Quick Lambdas can also be used entirely without the `_` placeholders, in which case they wrap the target in a no argument `lambda: ...` thunk:

```python
from random import random
thunk = f%random()
print thunk()
#0.5497242707566372
print thunk()
#0.3068253802774531
```

This cuts out reduces the number of characters needed to make a thunk from 7 to 2, making it much easier to use thunks to do things like emulating [by name parameters](http://locrianmode.blogspot.com/2011/07/scala-by-name-parameter.html).

Parser Combinators
------------------
```python
def reduce_chain(chain):
    chain = list(reversed(chain))
    o_dict = {
        "+": f%(_+_),
        "-": f%(_-_),
        "*": f%(_*_),
        "/": f%(_/_),
    }
    while len(chain) > 1:
        a, [o, b] = chain.pop(), chain.pop()
        chain.append(o_dict[o](a, b))
    return chain[0]

"""
PEG Grammer:
Value   <- [0-9]+ / '(' Expr ')'
Op      <- "+" / "-" / "*" / "/"
Expr <- Value (Op Value)*
"""
with peg:
    value = r('[0-9]+') * int | ('(', expr, ')') * (f%_[1])
    op = '+' | '-' | '*' | '/'
    expr = (value, ~(op, value)) ** (f%reduce_chain([_] + _))

print expr.parse_all("123") #[123]
print expr.parse_all("((123))") #[123]
print expr.parse_all("(123+456+789)") #[1368]
print expr.parse_all("(6/2)") #[3]
print expr.parse_all("(1+2+3)+2") #[8]
print expr.parse_all("(((((((11)))))+22+33)*(4+5+((6))))/12*(17+5)") #[1804]
```

[Parser Combinators](http://en.wikipedia.org/wiki/Parser_combinator) are a really nice way of building simple recursive descent parsers, when the task is too large for [regexes](http://en.wikipedia.org/wiki/Regex) but yet too small for the heavy-duty [parser generators](http://en.wikipedia.org/wiki/Comparison_of_parser_generators).

The above example describes a simple parser for arithmetic expressions, using our own parser combinator library which roughly follows the [PEG](http://en.wikipedia.org/wiki/Parsing_expression_grammar) syntax. Note how that in the example, the bulk of the code goes into the loop that reduces sequences of numbers and operators to a single number, rather than the recursive-descent parser itself!

In fact, the correspondence would be even closer if we stripped out the snippets of code which perform the actual arithmetic (as opposed to just the parsing):

```python
"""
PEG Grammer:
Value   <- [0-9]+ / '(' Expr ')'
Op      <- "+" / "-" / "*" / "/"
Expr <- Value (Op Value)*
"""
with peg:
    value = r('[0-9]+') | ('(', expr, ')')
    op = '+' | '-' | '*' | '/'
    expr = (value, ~(op, value))
```

Much of this conciseness arises from the use of macros to wrap tuples into `Seq()`s and strings into `Raw()`s, as well as wrapping each definition in a `Lazy()` thunk to allow recursive definitions to work:

```python
with peg:
    value = Lazy(lambda: r('[0-9]+') | Seq([Raw('('), expr, Raw(')')]))
    op = Lazy(lambda: Raw('+') | Raw('-') | Raw('*') | Raw('/'))
    expr = Lazy(lambda: Seq([value, ~Seq([op, value])]))
```

Although these modifications are pure [syntactic sugar](http://en.wikipedia.org/wiki/Syntactic_sugar), the grammar becomes completely obfuscated once the sugar is removed.

These parser combinators are applicable to more than just toy problems; below is a parser for JSON built using the same library:

```python
"""
JSON <- S? ( Object / Array / String / True / False / Null / Number ) S?

Object <- "{"
             ( String ":" JSON ( "," String ":" JSON )*
             / S? )
         "}"

Array <- "["
            ( JSON ( "," JSON )*
            / S? )
        "]"

String <- S? ["] ( [^ " \ U+0000-U+001F ] / Escape )* ["] S?

Escape <- [\] ( [ " / \ b f n r t ] / UnicodeEscape )

UnicodeEscape <- "u" [0-9A-Fa-f]{4}

True <- "true"
False <- "false"
Null <- "null"

Number <- Minus? IntegralPart FractionalPart? ExponentPart?

Minus <- "-"
IntegralPart <- "0" / [1-9] [0-9]*
FractionalPart <- "." [0-9]+
ExponentPart <- ( "e" / "E" ) ( "+" / "-" )? [0-9]+
S <- [ U+0009 U+000A U+000D U+0020 ]+

"""
with peg:
    json_exp = (opt(space), (obj | array | string | true | false | null | number), opt(space)) * (lambda x: x[1])

    obj = ('{', ((string, ':', json_exp), ~((',', string, ':', json_exp))) | space, '}')
    array = ('[', (json_exp, ~(',', json_exp)) | space, ']')

    string = (opt(space), '"', ~(r('[^"]') | escape) * ("".join), '"')
    escape = '\\', ('"' | '/' | '\\' | 'b' | 'f' | 'n' | 'r' | 't' | unicode_escape)
    unicode_escape = 'u', +r('[0-9A-Fa-f]')

    true = 'true' * (lambda x: True)
    false = 'false' * (lambda x: False)
    null = 'null' * (lambda x: None)

    number = (opt(minus), integral, opt(fractional), opt(exponent))
    minus = '-'
    integral = '0' | r('[1-9][0-9]*')
    fractional = ('.', r('[0-9]+'))
    exponent = (('e' | 'E'), opt('+' | '-'), r("[0-9]+"))

    space = r('\s+')
```

Not how closely it matches the PEG grammer shown above it! The parser shown only parses the JSON into a parse tree but does not convert it into python data structures (`dict`s, `list`s, `string`s, etc.). That can easily be fixed:

```python
with peg:
    json_exp = (opt(space), (obj | array | string | true | false | null | number), opt(space)) * (lambda x: x[1])

    obj = ('{', ((string, ':', json_exp), ~((',', string, ':', json_exp))) | space, '}') * (
        lambda x: dict([[x[1][0][0], x[1][0][2]]] + [[y[1], y[3]] for y in x[1][1]])
    )
    array = ('[', (json_exp, ~(',', json_exp)) | space, ']') * (lambda x: [x[1][0]] + [y[1] for y in x[1][1]])

    string = (opt(space), '"', ~(r('[^"]') | escape) * ("".join), '"') * (f%"".join(_[2]))
    escape = '\\', ('"' | '/' | '\\' | 'b' | 'f' | 'n' | 'r' | 't' | unicode_escape)
    unicode_escape = 'u', +r('[0-9A-Fa-f]')

    true = 'true' * (lambda x: True)
    false = 'false' * (lambda x: False)
    null = 'null' * (lambda x: None)

    number = (opt(minus), integral, opt(fractional), opt(exponent)) ** (f%float(_+_+_+_))
    minus = '-'
    integral = '0' | r('[1-9][0-9]*')
    fractional = ('.', r('[0-9]+')) ** (f%(_+_))
    exponent = (('e' | 'E'), opt('+' | '-'), r("[0-9]+")) ** (f%(_+_+_))

    space = r('\s+')

test_string = """
    {
        "firstName": "John",
        "lastName": "Smith",
        "age": 25,
        "address": {
            "streetAddress": "21 2nd Street",
            "city": "New York",
            "state": "NY",
            "postalCode": 10021
        },
        "phoneNumbers": [
            {
                "type": "home",
                "number": "212 555-1234"
            },
            {
                "type": "fax",
                "number": "646 555-4567"
            }
        ]
    }
"""
import json
print json_exp.parse_all(test_string)[0] == json.loads(test_string)
# True
```

As you can see, the full parser parses that non-trivial blob of JSON into an identical structure as the in-built `json` package
