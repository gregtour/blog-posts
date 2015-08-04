### Designing a Programming Language: II
> ![Duck-logo](small.png?raw=true "Duck programming language logo")  Designing a minimal compiler from simple steps.



#### Part 1: The Premise

This is a continuation of our earlier discussion of programming languages in general. In the first part of this
series, we discussed what is necessary to design a language from its roots and the process involved in converting that
language specification into a working machine through the process of constructing an interpreter. Because of the
strategies we used in building this interactive environment for our programming language, we made certain design
decisions about the function of our language that both gave us additional flexibility and also limited our choices. 

Originally, we wanted to ease the process of development, both as tool writers and for the programmers who would be
using our language. We also wanted to include a certain power of expressiveness in the source code itself, allowing a
relatively small number of statements, commands, or instructions to cover a great deal of functionality. And finally,
we wanted to get up and running as soon as possible. For those reasons, after we had sketched out the design of our
language in terms of its grammatical structure and fleshed out its core design and features, we went ahead and
continued from developing a lexer and parser into building a dynamic interpreter. This gave us a lot of power in terms
of determining events at runtime, such as the value of variables or expressions, which might otherwise be determined
at compile time. It also allowed us to forgo any sort of advanced type checking when parsing our source code and
instead delayed that until evaluating expressions. This made things easier to develop but also skimmed out a number
of static guarantees that we might have been able to make about our program. It also provides our programmer with
little assurance that a given program will operate as expected, besides being able to be parsed.

As we said, this made the step of implementation much easier, but it put some constraints on our language and also
limited our performance as we were then limited to using our own constructed environment. These are some of the 
trade-offs of using a dynamic, weakly typed language like the *Duck programming language*.

In this the second part of our series on programming languages, we will go in a completely opposite direction. Instead
of building a language that is as flexible as possible, we will put constraints on our problem domain this time.
Instead of working with dynamic language features, we will limit ourselves to static code. Instead of having a loose
type system, we will opt to use strong typing. And while our initial project resulted in us building a language
interpreter, this time we will build a compiler for x86 computers. 


#### Part 2: Motivations

Part of the purpose of this study on programming languages is to find out what the best method and practice for
building a do-it-yourself language is without getting bogged down in research. While we always like to make our
lives easier, we are choosing to create almost everything from scratch. We want our code to be self-reliant and we
want to to be able to claim ownership to everything we have written and built. We also need the option of distributing
our software ourselves, and if we are responsible for handcrafting all of the components to our programming language,
this gives us an understanding of the details well enough then that we can provide implementations for any platform.

Our target audience is still relatively general; we want to include as many developers as possible. Still, we don't
want to dumb down any material so we will attempt to explain things at just the level that is necessary.

There are a number of tools and resources that we developed in the first part of this series, in developing the *Duck 
programming language*, that we will continue to use for this project. This gives us somewhere to start from and
provides us with familiar tools that we can use. What we will be building from includes: the Duck lexer, the parser
generator, and the parser. Although we might need to make subtle modifications, we can generally use these parts
as-is.

We should also define the scope of our project. In the first half, we expected to make a general programming language:
a language that would be well suited for any task. That is a perfect goal when dealing with a project that has no 
constraints. Here, we are looking to complete our task as soon as possible so that we can reflect on the method that
we used, provide improvements, and move on to other projects, so we are going to cut-back on the scale significantly.
This also gives us the option to make drastic modifications or change the scope of what we are doing later on.

For this reason, we will be making a compiler for a language that only operates on integer types. We will provide no
support for objects (or object oriented programming) and instead only implement a simple imperative language. The
fewer syntactic features we provide, the smaller our resulting code will be, and the more we can analyze our results.
We will still use a generalized LR parser/parser generator, because it allows us to make arbitrary changes to the
syntax of our language, if we need to make broad changes.


#### Part 3: First Steps

Let's outline the structure and grammar of our programming language. Having already discussed the concerns of language design (to a small degree) in the first segment, we can sort of brush over or breeze through some of the larger parts
of this design without having to do an excessive amount of clarification. Luckily, we also won't discuss technical
problems with the frontend, like constructing our parser-generator. Instead, all of our technical work will be on the
backend.

First of all, we will support the same operations in logic and arithmetic that our earlier programming language
supported, the *Duck programming language*, and we will offer support for the same orders of operation. To remind, the
CFG (context-free grammar) form of this syntax looked like the following.

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

For the main structure of our language, we will define a source program as being a sequence of top-level statements.
Our statements can either be include statements, which define other source files to include on a literal level,
type-declarations, which define static variables, or function definitions.

The program's execution will start with the function declared with the identifier `main`. Otherwise, functions will
have arbitrary identifier names. As a sort of creative syntax, we will be formatting our programs in a way that's
similar to C or Java, but instead of declaring types before identifiers, we will declare them after, separated by a
colon ':'. Our statements will be delimited by semicolons, and unlike our previous language, we will be ignoring
whitespace like newlines. 

Then from an above view our grammar looks like this:

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
effort for us to implement as language features, so we will forgo implementing them for now.

Alternatively, we have included support for both if and else statements, and the phrasing of our grammar allows for
else-if statements to occur. Not only that, but with our optional inline syntax, allowing `<block>`'s to take on the
form of a single statement, there are ambiguous parsings for our if, if-else, else, and else-if statements.

Originally, we said this was something we wanted to avoid, so that we could have a deterministic parsing for our
source. However, here we will recognize that our parser will by default match these with their innermost parsings, due
to the design of how our parser resolves conflicts. 

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
```
{
if (check_condition) {
    if (condition)
        a();
    } else
        b();
}
```

If we determine we want to reverse these parsing rules, we can define our own 'correct' form for this based on our
definition. That is what we will choose to do here. In this case, the default behavior for the original code sample
would be, with indentation showing the parsed form, this.

```
{
if (check_condition)
    if (condition)
        a();
else
    b();
}
```

In this case, matching dangling else clauses with the outermost matching if, the way to express the same functionality
as our original example, as it would operate in C or Java, would be:

```
{
if (check_condition) {
    if (condition)
        a();
else
    b();
}
}
```

In any case, we allow for either behavior to be supported, it just depends on the semantics of how it is expressed.
Additionally it should be noted that issues like these don't arise if one uses the best practices in their code in the
first place, not allowing for any ambiguity in the expression of logical statements.

This instead becomes a unique design decision.

As was said, our parser will still naïvely produce parse trees in the C-style form. In order to remedy this, we will
need to process our abstract syntax tree and operate on it to re-structure if-statements. While possibly a bit
difficult, it's good for us to begin understanding how we might manipulate our source code on a macro level, before it
has been processed for static runtime concerns and before we have begun to form an intermediate representation or
done any code generation.

In order to do this, we will traverse the AST or syntax tree. For each node, we will first process the node itself and
then process any children. We will first identify if-statements that are lacking an else-clause. In this case, the
else clause will reduce to `<epsilon>`. Then we will traverse the inner code-block and find any else-clauses that
could be re-paired with our outside production. In this case, we will only study inline if-statements, where the code
inside reduces to a single statement.

Returning to the syntax of our language, our other concerns are the types allowed in our type system, and the
definition of `<final>` statements. As was said before, we will only allow integer types in our language. Our
functions may not return variables, so we will also have a named void type as well.

In our grammar this will look like:
```
<type> ::= int
<type> ::= void
```

And finally, our base level expressions will either be parenthesized expressions, referencing a variable by identifier
name, a function call, or an integer or Boolean value.

In grammatical form:
```
<final> ::= ( <expr> )
<final> ::= <identifier>
<final> ::= <identifier> ( <arg*> )
<final> ::= <integer>
<final> ::= <boolean>
<boolean> ::= true
<boolean> ::= false
```

And this is the entire skeleton of our language's syntax and grammar. As was said before, we can go back and modify
this to expand our functionality *after* we have a working compiler for our language. For this, we need to be sure
that we factor in our changes in a way that preserves the working system we have.

#### Part 4: Type Checking

From a high-level view, we can describe the way our compiler will operate.

Given a source file, the compiler will produce an executable that we can run on our system. We will include the linker
as part of the compiler, because we only intend to use two standard built-in functions. Because we don't intend to
write an assembler ourselves, we might as well say that our compiler will produce assembly for our executable. In
reality, our compiler will use such a small subset of intstructions that it would be very easy for us to include the
assembler functionality as well, but that leaves us with a number of abritrary and obscure concerns for each platform
that we would rather avoid.

So, our compiler will convert source code to target assembly. In the process, we will go from source code to an
abstract syntax tree representation which will process, then eventually we will convert this to an intermediate
representation, and finally we will generate our "code" which in this case will be assembly.

The phases our compiler will go through are: 1) parsing, 2) static analysis, 3) IR generation, and 4) code generation.

We've already covered parsing in the first part of our series, so next we have to cover static analysis. Static
analysis involves analyzing the source code from our context-free language and making sure that our code makes sense
*in context*. This involves checking that the variables we use as identifiers have been defined, and have been defined
as the right types. Usually this involves checking that variables or functions are defined before they are used,
however this is really unnecessary for us. Instead, we will just make sure that they are only defined once.

As we are compiling and analyzing our code *before* it is being run, we have the advantage of being able to study the
entire program before generating code. For this reason, it would be just as convenient for us to declare variables at
the end of a function as at the beginning, and why not, because it creates just as interesting of a coding style. (Our
language will have a lot of charm for being rather simple.)

Remember, our parser generator is capable of creating boiler-plate code for us. This means it can create a skeleton
framework for any operations we want to perform on a per-node basis for us. This is similar to how other free tools
work, for generating parsers, but we will separate the steps of defining a language and providing implementation for
it because this is cleaner from an implementation perspective. For our purposes, we will generate our boiler code
which we will duplicate across two modules to use the same processing scheme twice. The first will be for the static
analyzer and the second will be for the intermediate code generator. 

Given that after parsing we know our source code is well-formed, the only task we have is to ensure that our types are correct. Again, using multiple passes aids us in performing this work. In our first pass, we will identify all
variable and function  declarations and record their types. If a variable or function is defined in the same scope
with an identical name, our static analysis phase will fail and report the error. We can also determine that variables
are only declared as integer types, as they cannot be null.

In the second phase, we will identify if all variables are used with their proper types. Given our restraints, this
boils down to checking that all function calls used as expressions must return an integer if their result is used. We
will also check that functions are called with the proper number of arguments and that the arguments are of matching
types.

Again, by limiting ourselves to a single type, this drastically decreases the amount of work we need to do when
checking types.

This completes type-checking and leaves us with some static information about our program. Specifically, the names and
locations of variables and functions and their types.

#### Part 5: Intermediate Code Generation

This next step becomes important in moving closer to our running executable. We need to take the program that we have
parsed and analyzed and convert it to a form that is closer to machine code. There are a number of ways to do this 
that have varying strengths and weaknesses, but we will choose the easiest method for our task, after outlining two
of these options.

The intermediate representation in a compiler is in general a simplified language of instructions for a program. It
moves from being a structured representation in a parse tree to being less structured as either a stream of
instructions or a directed graph. The psuedocode instructions that we use will also be important.

As we break up instructions and expressions into a simple operations, there will be a number of temporary variables or
temporary values along the way. Consider the case of adding three variables and storing the result in another
variable. As we break this up into two additions and an assignment, we can choose a representation that introduces
named values or one that does not. In an example, we would see:

```
d = a + b + c
...
Named temporary values:
t1 <- a + b
d <- t1 + c
...
Anonymous temporary values:
a + b
+ c
<- d
```

Perhaps the first instance makes more sense when formatted as it is. Explaining this, we would say we introduce a
temporary named variable whenever we have to split up an expression in order to use the result. This becomes important
as we look at more complicated expressions, such as expressions where the results of invoked functions are used as
parameters to functions, and where function parameters become complex expressions themselves.

For some architectures, like MIPS or PowerPC, which use a reduced instruction set, there are often a large number of 
registers to assign to and from. This could be useful for evaluating large expressions. If we have fewer than 16 or 32
different temporary variables or vlaues, we would simply store them in one of these registers before using them.

We aren't targeting a reduced instruction set (RISC) computer, though. We're targeting x86 assembly. In the case that
we are making a 32-bit executable, that leaves us with 8 registers we can use, and in the case of 64-programs that
gives us at most 16 registers. This is including some special purpose registers that we don't want to assign to.

Still, introducing new temporary variables could be logical, considering that we will be storing variables on the 
stack in our program, and we can reference them there. However, when generating code, counting off, enumerating, or
naming variables would take time and effort.

Therefore, we will choose to use the second form, which is an anonymous stack-based representation.

Rewriting this pseudocode in a more simplified and more intelligble form this would be:
```
PUSH A
PUSH B
ADD
PUSH C
ADD
STORE D
```

Here our intermediate operations are basically as simple as they could be. Even though we are targeting a complex
instruction-set CPU (CISC), this will be a huge boon to us and our productivity in compiler writing.

Although we could start talking about x86 assembly and how our design correlates with the machine we are targeting, at
this point we will deal only in this abstract representation. We will leave every detail to the code generation
implementation. Instead, we are working with the most simplified computer design we could ever consider. This helps us
because a) it becomes a trivial task to write the intermediate code generator and b) it becomes very easy to write our
(initial) code generator. We will see the limitations this leaves us with, particularly with performance, but we will
eventually be able to overcome these in a big way and being able to work our way to a working state will be great for
our motivation.

So, given the design of our language, let us consider what operations our stack-based intermediate representation will
need. Given our arithmetic and conditional logic expressions we will want stack operations for all of the following:

> NOT, MINUS, AND, OR, ADD, SUB, MULTIPLY, DIVIDE, MODULUS, COMPARE EQUAL, COMPARE UNEQUAL, COMPARE LESS THAN, COMPARE
GREATER THAN, COMPARE LESS THAN OR EQUAL, COMPARE GREATER THAN OR EQUAL

Each of these will consume one or two parameters from the stack (popping off the stack) and produce one result
(pushing on to the stack). Given that some of our logic expressions in comparision are duplicitous, we will remove
some of them to simplify and build them from composite statements.

So instead we will have:

> NOT, MINUS, AND, OR, ADD, SUB, MULTIPLY, DIVIDE, MODULUS, COMPARE EQUAL, COMPARE LESS THAN, COMPARE GREATER THAN

Using our NOT operation, we can implement the other three comparisons with a single extra stack instruction.

This basically covers the implementation of all of our basic expression rules.

To cover assignment statements and retrieving values from variables we will add stack instructions for

> STORE, LOAD

that take a named variable (including scope) as a parameter.

We will use the stack to pass parameters to functions, instead of needing to bind them by name.

We will also use the stack to return values from a function. When a function is called, it will consume its parameters
on the stack and produce a value if the function returns a value, again on the stack.

We will need an instruction to invoke a function. For this we will use:

> CALL

which will again take a named parameter.

Here we've covered all functionality for our programs except for if statements and while loops.

For these we will implement the instructions

> JUMP, JZ

Which will branch to labels unconditionally, or if the value on the stack is zero.

We will also need a form of labels, so we will add the psuedo-instruction

> LABEL

which will take a unique ID as a parameter, which we will generate when generating our intermediate representation.

Here we've outlined one of the simplest intermediate reperesentations imaginable. What we will do is process each
function individually, creating a stream of IR instructions that match the given function on a statement by statement
level. Our code generator will then be able to work from these function blocks to create the assembly code that we
need. This IR representation is also flexible enough that we could easily add other target platforms to our compiler,
if we were to optimize code generation for them. We also have the possibility of doing some trivial back-end
optimizations, but we will save this as a tertiary concern.

Let's implement instructions then.

Given our boiler plate code, we can emit IR code on a production by production basis. As we are operating on function
bodies, we can start at a block level. Recalling that our instruction `<block>`'s are either `<fstmt>`'s or `<fstmt*>`'s, we should provide IR generation for each kind of `<fstmt>`.

```
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
