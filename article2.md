### Designing a Compiler
> ![Duck-logo](small.png?raw=true "Duck programming language logo")  Designing a Programming Language: II



#### Part 1: The Premise

This is a continuation of our earlier discussion of programming languages in general. In the first part of this series,
we discussed what is necessary to design a language from its roots and the process involved in converting that language
specification into a working machine through the process of constructing an interpreter. Because of the strategies we
used in building this interactive environment for our programming language, we made certain design decisions about the
function of our language that both gave us additional flexibility and also limited our choices. 

Originally, we wanted to ease the process of development, both as tool writers and for the programmers who would be using
our language. We also wanted to include a certain power of expressiveness in the source code itself, allowing a
relatively small number of statements, commands, or instructions to cover a great deal of functionality. And finally, we
wanted to get up and running as soon as possible. For those reasons, after we had sketched out the design of our language
in terms of its grammatical structure and fleshed out its core design and features, we went ahead and continued from
developing a lexer and parser into building a dynamic interpreter. This gave us a lot of power in terms of determining
events at runtime, such as the value of variables or expressions, which might otherwise be determined at compile time. It
also allowed us to forgoe any sort of advanced typechecking when parsing our source code and instead delayed that until
evaluating expressions. This made things easier to develop but also skipped out on a number of static guarantees that we
might have been able to make about our program. It also provides our programmer with little assurance that a given
program will operate as expected, besides being able to be parsed.

As we said, this made the step of implementation much easier, but it put some constraints on our language and also
limited our performance as we were then limited to using our own constructed environment. These are some of the 
trade-offs of using a dynamic, weakly typed language like the *Duck programming language*.

In this the second part of our series on programming languages, we will go in a completely opposite direction. Instead 
of building a language that is as flexible as possible, we will put constraints on our problem domain this time.
Instead of working with dynamic language features, we will limit ourselves to static code. Instead of having a loose
type system, we will opt to use strong typing. And while our initial project resulted in us building a language
interpreter, this time we will build a compiler for x86 computers. 

#### Part 2: Motivations

Part of the purpose of this study on programming languages is to find out the best method and practice for building a
DIY language without being bogged down in academic research. While we always like to make our lives easier, we are
opting to create almost everything from scratch because we want our code to be self-reliant and also because it is
easier to claim ownership of everything we have written, we have the option of distributing it ourselves, and we have
an understanding of the details well enough then that we can implement our system for any platform that need to.

Our target audience is still really general; we want to include as many developers as possible. Still, we don't want to
dumb down any material so we will attempt to explain things at just the level that it requires.

There are a number of tools and resources that we developed in the first part of this series, in developing the *Duck 
programming language* that we will continue to use for this project. This gives us somewhere to start and provides us
with familiar tools that we can use. What we will be building from includes: the lexer, the parser generator, and the
parser. Although we might need to make subtle modifications, we can generally use these parts as-is.

We should also define the scope of our project. In the first half, we expected to make a general programming language:
a language that would be well suited for any task. That is a perfect goal when dealing with a project that has no 
constraints. Here, we are looking to complete our task as soon as possible so that we can reflect on the method that we
used, provide improvements, and move on to other projects, so we are going to cut-back on the scale significantly. This
also gives us the options to make drastic modifications or change the scope of what we are doing to match our needs.

For this reason, we will be making a compiler for a language that only operates on integer types. We will provide no
support for objects (or object oriented programming) and instead only implement a simple imperative language. The fewer
syntactic features we provide, the smaller our resulting codebase can be. We will still use a generalized LR
parser/parser generator, because it allows us to make arbitrary changes, especially down the road. It makes it easier
for us to make broad expansions. 

#### Part 3: First Steps

Let's outline the structure and grammar of our programming language. Having discussed many of the concerns of language
design, we can sort of brush over or breeze through some of the larger parts of this design without having to do
an excessive amount of clarification. Luckily, we also won't discuss technical problems with the frontend, like
constructing our parser-generator. Instead, all of our technical work will be on the backend.

First of all, we will support the same operations in logic and arithmetic that our earlier prorgamming language
supported, and we will offer support for the same orders of operation. To remind, the CFG (context-free grammar) form
of this looked like the following:

```
<expr> ::= <condition>
<condition> ::= <condition> and <logic>
<condition> ::= <condition> or <logic>
<condition> ::= <logic>
<logic> ::= not <comparison>
<logic> ::= <comparison>
<comparison> ::= <comparison> == <arithmetic>
<comparison> ::= <comparison> != <arithmetic>
<comparison> ::= <comparison> \< <arithmetic>
<comparison> ::= <comparison> > <arithmetic>
<comparison> ::= <comparison> \<= <arithmetic>
<comparison> ::= <comparison> >= <arithmetic>
<comparison> ::= <arithmetic>
<arithmetic> ::= <arithmetic> + <term>
<arithmetic> ::= <arithmetic> - <term>
<arithmetic> ::= <term>
<term> ::= <term> * <factor>
<term> ::= <term> / <factor>
<term> ::= <term> mod <factor>
<term> ::= <factor>
<factor> ::= -<factor>
<factor> ::= !<factor>
<factor> ::= <final>
```

This gives us a rudimentary expression type grammar for combining `final` elements into arithmetical or combinational
logic statements. We will use these expressions at the core of our language.

For the main structure of our language, we will defined a source program as being a sequence of top-level statements.
Our statements can either be include statements, which define other source files to include on a literal level, type
declarations which define static variables, and function definitions.

The program's execution will start with the function declared with the identifier `main`. Otherwise, functions will
have arbitrary identifier names. As a sort of creative syntax, we will be formatting our programs in a way that's
similar to C or Java, but instead of declaring types before identifiers, we will declare them after, separated by a :
colon symbol. Our statements will be delimited by semicolons, and unlike our predecessor language, we will be ignoring
whitespace like newlines. 

Then from a top-level our grammar looks like:

```
<program> ::= <gstmt*>
<gstmt*> ::= <gstmt> <gstmt*>
<gstmt*> ::= <epsilon>
<gstmt> ::= include <string-list> ;
<string-list> ::= <string> , <string-list>
<string-list> ::= <string>
<gstmt> ::= <type-declaration> ;
<type-declaration> ::= <identifier> , <type-declaration>
<type-declaration> ::= <identifier> : <type>
<gstmt> ::= <identifier> : <type> <block>
<gstmt> ::= <identifier> ( <param*> ) : <type> <block>
<param*> ::= <identifier> : <type>
<param*> ::= <identifier> : <type> , <param*>
<param*> ::= <epsilon>
<block> ::= <fstmt>
<block> ::= { <fstmt*> }
```

Functions are built-up of `fstmt`s or function statements.

The statements that are included in functions include type-declarations for local variables, assignment statements,
standalone expressions, if/if-else statements, while loops, and command instructions for return, break, and continue.

In a formalized view this would be:

```
<fstmt*> ::= <fstmt> <fstmt*>
<fstmt*> ::= <epsilon>
<fstmt> ::= <type-declaration> ;
<fstmt> ::= <identifier> = <expr> ;
<fstmt> ::= <expr> ;
<fstmt> ::= if ( <expr> ) <block> <else>
<else> ::= else <block>
<else> ::= <epsilon>
<fstmt> ::= while ( <expr> ) <block>
<fstmt> ::= return ;
<fstmt> ::= break ;
<fstmt> ::= continue ; 
<fstmt> ::= return <expr> ;
```

You can see we are already making concessions to remove complexity from our language. While for loops are exceedingly
useful, they aren't strictly necessary as they offer no additional functionality over while loops and require more
effort for us to implement as language features, so we will forgoe implementing them for now.

Alternatively, we have included support for both if and else statements, and the phrasing of our grammar allows for
else-if statements to occur. Not only that, but with our optional inline syntax, allowing `<block>`'s to take on the
form of a single statement, there are ambiguous parsings for our if, if-else, else, and else-if statements.

Originally, we said this was something we wanted to avoid, so that we could have deterministic parsings. However, here
we will recognize that our parser will by default match these with their innermost parsings, due to the design of how
our parser resolves conflicts. 

This is fine, but this is also how every other language operates. Additionally, it doesn't necessarily make more sense
than the alternatives. Consider the following code, indented to show how a naïve language or parser would match these
statements.

```
{
if (check_condition)
    if (condition)
        a();
    else
        b();
}
```

In this case, in a language like C or Java, the functions a or b would only be called if `check_condition` were true,
and then the result would be determined by `condition`. What if, however, we have behavior that we only want to run
if our condition is true and the value of check_condition is true, and we want to run another function if 
check_condition is false.

Then, the proper way to implement this would be:

