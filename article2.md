### Designing a Programming Language: II
> ![Duck-logo](small.png?raw=true "Duck programming language logo")  Designing a minimal compiler from simple steps.

#### Part 0: Designing a Compiler

```C
main: void 
{
  print("How to write a compiler.")
  count(3);
}

count(i: int): void
{
  if (i > 0)
  {
    count(i - 1);
    printi(i);
  }
}
```


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
want to be able to claim ownership to everything we have written and built. We also need the option of distributing
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

Let's outline the structure and grammar of our programming language. Having already discussed the concerns of language
design (to a small degree) in the first segment, we can sort of brush over or breeze through some of the larger parts
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
reality, our compiler will use such a small subset of instructions that it would be very easy for us to include the
assembler functionality as well, but that leaves us with a number of arbitrary and obscure concerns for each platform
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

The easiest way for us to do this is on a statement-by-statement basis. First, we will process each type declaration
statement and each function definition. These will be stored in records in the compiler. Only one declaration is
allowed for a given name in a specific scope, so if there are any duplicate entries this should produce an error.

Function declarations are stored in their own table of records containing information about the function's name, its
return type, its parameters, and the function body. The parameters must be enumerated and their types determined at
this phase. Parameter names must not be duplicated, so this is another error case. 

Finally, we will check the types of values in expressions. For each expression, we will compare the types of its
inputs. We will also compare the types of and counts of function arguments. Void values, from functions that have no
return value, must not be combined with other values in any operation. Assignment statements must assign an integer
value to a variable. After checking all of these error cases, then we know that the program's syntax and expressions
are valid and the type checker is finished.

#### Part 5: The Intermediate Representation

This next step becomes important in moving closer to our running executable. We need to take the program that we have
parsed and analyzed and convert it to a form that is closer to machine code. There are a number of ways to do this 
that have varying strengths and weaknesses, but we will choose the easiest method for our task, after outlining two
of these options.

The intermediate representation in a compiler is in general a simplified language of instructions for a program. It
moves from being a structured representation in a parse tree to being less structured as either a stream of
instructions or a directed graph. The pseudocode instructions that we use will also be important.

As we break up instructions and expressions into simple operations, there will be a number of temporary variables or
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
different temporary variables or values, we would simply store them in one of these registers before using them.

We aren't targeting a reduced instruction set (RISC) computer, though. We're targeting x86 assembly. In the case that
we are making a 32-bit executable, that leaves us with 8 registers we can use, and in the case of 64-programs that
gives us at most 16 registers. This is including some special purpose registers that we don't want to assign to.

Still, introducing new temporary variables could be logical, considering that we will be storing variables on the 
stack in our program, and we can reference them there. However, when generating intermediate code, counting off,
enumerating, or naming variables would take extra time and effort.

Therefore, we will choose to use the second form, which is an anonymous stack-based representation.

Rewriting this pseudocode in a more simplified and more intelligible form this would be:
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

> NOT, MINUS, AND, OR, ADD, SUBTRACT, MULTIPLY, DIVIDE, MODULUS, CMP EQUAL, CMP LESS, CMP GREATER

Each of these will consume one or two parameters from the stack (popping off of the stack) and produce one result
(pushing onto the stack). This basically covers the implementation of all of our basic expression rules. In the case
of comparing inequality, testing if values are greater than or equal, or testing if values are less than or equal,
we will use the operations CMP EQUAL, CMP LESS, or CMP GREATER followed by a NOT operation to negate the result.

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

We will also need a form of labels, so we will add the pseudo-instruction

> LABEL

which will take a unique ID as a parameter, which we will generate when generating our intermediate representation.

Here we've outlined one of the simplest intermediate representations imaginable. What we will do is process each
function individually, creating a stream of IR instructions that match the given function on a statement by statement
level. Our code generator will then be able to work from these function blocks to create the assembly code that we
need. This IR representation is also flexible enough that we could easily add other target platforms to our compiler,
if we were to optimize code generation for them. We also have the possibility of doing some trivial back-end
optimizations, but we will save this as a tertiary concern.

Let's implement instructions then.

Given our boiler plate code, we can emit IR code on a production by production basis. As we are operating on function
bodies, we can start at a block level. Recalling that our instruction `<block>`'s are either `<fstmt>`'s or
`<fstmt*>`'s, we should provide IR generation for each kind of `<fstmt>`.

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

...
...

#### Part 6: Code Generation

We have two choices when generating assembly code from our intermediate representation. The first would be a literal
translation and the second would be a recompilation. Given that our x86 target platform is a complex instruction set
computer, it can undoubtedly emulate all of the instructions that we are using in our intermediate representation.
That is part of why we chose it. On the other hand, directly translating our program as a set of stack-based 
instructions would hardly be as efficient as what we intended in compiling our source code.

Let's look at the instruction model of an x86-32 computer processor.

Our code and examples will use Intel style syntax for assembly.

We have eight 32-bit registers at our disposal. Namely
```
EAX EBX ECX EDX
ESI EDI EBP ESP
```

Let's outline the most important of these, foremost. EAX is the accumulator register and is used for returning values
from functions. ESP is the stack-pointer and will be used to track function parameters and local variables in a 
function call. Furthermore, EBP will also be used to manage stack addresses. We will consider the other 5 as being
general purpose registers for the moment. 

In 32-bit assembly on an x86 processor, parameters are passed to functions on the program's stack. Additionally the
stack is used for holding local variables. If a function is called with 4 long int local variables, each taking up
4-bytes of memory, then the stack pointer is subtracted by 16 to represent this memory, and assuming EBP points to the
top of the stack at the beginning of the function, these local variables are accessed with offsets from this address
in memory.

One of the most difficult tasks in writing a compiler backend is determining register assignment. We will attempt to
find a simple way to assign registers to each of our values.

If we were to provide a direct translation for our IR code to x86 assembly, it would take us only a few instructions
per IR opcode to accomplish. We could even limit ourselves to only using 3 registers if we had to. Obviously, this
wouldn't be using our computing resources to their full advantage.

Instead of viewing our machine as a stack-based computer, let's return again to thinking of a machine with registers
and a stack. Instead of thinking of the stack as something that grows and shrinks, as we generate code on a
function-by-function basis we will think of the actual stack as being fixed and we will instead think about local and
global values. If all we needed in performing operations for computation was the space allocated in local variables,
our job would be very easy. Instead, we must also consider the temporary storage space required. That's what led us
to design our IR representation as we did, to avoid naming the temporary variables we created.

Taking advantage of the fact that we know our IR code should be flat, with no changes in the size of the stack from
the beginning to end, we can perform a simple task to find out the maximum amount of local storage space we will need
to execute this function. We will start from the beginning with a count of 0 and consider each possible code path
individually, continuing from the beginning of the function to the end, and increase our count whenever we encounter
a push instruction and decrease it when we encounter a pop instruction. When we determine the maximum size of our
stack, that is the amount of temporary storage we will need, and we will add the amount of named storage to find the
total amount of space that we need on the stack.

Then, as we go about generating code for operations, we will internally to the compiler track a stack position that
represents the index in this virtual stack machine. An operation like ADD will result in an instruction that takes two
local variables and stores them in another.

Being that we are writing a compiler for x86, we will also need to consider the assembly writing conventions for this
architecture, and factor that in to register assignment. It's possible that, given many of our operations need to take
place in the EAX register, that we can save some transfers between registers. We also have the option to use EBX, ECX,
and EDX as additional temporary storage.

Using peephole optimizations, we may be able to better generate code from this IR representation that we have
internally.



#### Part 6: Trivial Code Generation
#### Part 7: Advanced Code Generation

#### Part 8: Standard Library
```
print
printi
```
input
