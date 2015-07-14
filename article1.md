### Designing a Programming Language: I
> Designing a language and building an interpreter from concept to evolution.


#### Contents
#### Part 1: The Premise
#### Part 2: The Syntax
#### Part 3: The Setup
#### Part 4: The Lexer
#### Part 5: The Parser
#### Part 6: The Interpreter
#### Part 7: The Library
#### Part 8: The Garbage Collector
#### End Notes



#### Part 1: The Premise

Programming languages come in a variety of different paradigms. Even so, there tend to be two main camps along the 
language front. There are static languages and there are dynamic languages. To avoid too much history and any sort of 
in-depth analysis, this article will simplify things with a number of assumptions. One assumption we will make which 
might not always hold true is that programs written in static languages are compiled to machine code, while 
programs that are written in dynamic languages run in an interpreter. 


Testing the definitions of what makes a language static or dynamic might illuminate this more. In a static language,
variables and procedures have rigidly defined types. Attempting to access a value that has not been named in a
relevant scope leads to a syntax error issued at compile time. The programming language is organized in such a way
that, when it is analyzed at a source level, the locations of all variables and functions are known by lexical
address: i.e. given the relevant scope, a variable can be identified by the order it is defined in.


As an example, consider the case of defining a variable in Visual Basic:
```
Dim num1, num2 As Integer
Dim text As String
```


This would be the prototype for a static language. When variables are first declared, they must be paired with a type.
In order to compare with a dynamicly typed language, let us look at an example from ECMA script or JavaScript.


```javascript
var num1, num2;
var text;
```


In this example, no types are declared for these variables. The variable 'num1' could be used as any type by assigning
any value to it. Additionally, the declaration step is wholly unnecessary. The way that values are stored includes the
type information.


Additional examples of dynamically typed languages besides JavaScript include Python, Ruby, Lua, Scheme and Lisp, and
a number of others. Aside from the style of syntax and the degree of expressiveness, these programming languages are
largely similar because of their type system. So called dynamicly typed languages are often referred to as duck-typed
or 'Duck' languages.


Adopting many of these identifying features, and really as an exercise in constructing a language from basic parts, we
will build the Duck programming language from the ground up. We will explore this process without regard to its 
pragmatism or practicality. Indeed, any ideas of utility will only come after we have completed the process.


As this guide is minimal at best in explaining each of the complex steps involved in crafting a programming language,
many concepts are taken as given information. It may be best to approach this topic after having done a fair bit of
research and after gathering experience in programming in a number of different languages.


#### Part 2: The Syntax


At this point it might be easy to scrabble together a document of syntactic expressions, beginner how-to tutorials,
wikis, or grammar guides as an explanation of the language and drop the topic altogether, moving on to more productive
tasks. Sometimes it seems like the design of a language, down to its true grammar, is something more for linernotes on
a reference manual than something that exists that has been written, crafted, or changed over time. The purpose of
this explanation is not to suggest that your programming language should be chiseled into stone and that that is how
great concepts come to fruition. Instead, true to the nature of our task, we will at every step of the way look at how
things can be changed, expanded, and made to evolve over time. What might be suitable for one task is not the solution
to every problem, and having a way to change things or reuse our efforts is always good.


With that in mind, we will sort of gloss over what the fundamental mechanics of the 'Duck' language are at the atomic
level. Instead, we will explore what the essence is. We want to discover the intention of the language itself.
If we can get into the mind, the feng shui or the essential attitudes of the programming language, that would be
enough. Still, programming languages are systems that we use to express ideas. And for any manner ofcommunication to
be useful, we need to have some common ground.


We will create a language that is very neutral to the background of the programmer. We will also work in an imperative
style. Although certain ideas of functional nature may seep through eventually, we are trying to build the
basic language for the common programmer. 


To find a root concept we can work from, let's begin with statements. There are statements that form operations. Given
two values, add them together and assign the result. If an expression is true, execute a block of statements. Evaluate
expressions and then make a procedure call with arguments. Et cetera. A block of statements will be described in terms
of a statement list. Given a function declaration, a kind of statement, there is a name for the function, a list of parameter names, and a list of statements making up the body. An If/Else statement has a similar nature. It is a statement that contains additional statements.


We won't put any limitations on the placement of these. Functions can be defined inside of functions. In this case
they will be local to where they are defined. We don't want to introduce limitations when forming our definitions from
the start, because they might be based on expectations that aren't true. If the limitations we impose on ourselves are
artificial, then we will be putting time and effort into enforcing artificial limitations, and that becomes wasted
effort.


Our language must have some common ground with existing languages, so we will use familiar constructs like for loops
and while loops. For loops will use ranges in an explicit way. There will be a beginning value and ending value, and
the loop body will be executed for both of these and every value in between. This is assuming we are starting with
integers, or possibly floating-point numbers, but always increasing by increments of one. 


We will allow all of the basic operations we have come to expect. Namely addition, subtraction, multiplication, and
division, as well as modulo division (something we could implement ourselves), string concatenation for convenience,
negation, not, and boolean expressions. These are fundamentals we've come to expect.


What we are trying to accomplish is creating the most flexible and malleable language, so we must also think of
distincly dynamic elements that we can add. Arrays are fundamentally useful. Let's add arrays that allow the use of
any index, without specifying a container size. Let's expand this array functionality to allow for values of any type
to be used as indices. Furthermore, let's create dictionary types with a familiar object notation. These will be
similar to classes or structs from related languages. To increase flexibility, these will be the same object
internally as arrays, and the syntax for each can be used interchangably. In order to keep track of objects and
complex types, it makes sense to implement memory management through garbage collection, instead of passing that large
responsibility on to the programmer adapting to use our language.


Another feature that increases our flexibility is allowing functions to be first-class objects. This means that we can
use functions as parameters, return them as values, and assign them to named variables. We can rename a function and
use the original name for something else. We can attach functions to objects and use them as classes. Although passing
functions as first-class objects is a functional idea, here it overlaps with the ideas of dynamic languages so we will
adopt this feature.


Roughly this is a description of the language we intend to create. We haven't written down anything concrete as to the
style yet, but we have a solid understanding of what we want programs to look like and what features they can use to
implement their logic.



#### Part 3: The Setup


A casual reader following along might not be interested in developing their own language, as of this moment. They
might be more interested in the mechanics involved. As a rather extreme exercise, I invite anyone to attempt to create
their own "duck" compatible runtime environment. That being said, I am about to outline the process and tools that I
used to create this language, even though I recognize there are multiple ways to go about the process. 


For example, I chose to write everything in the C programming language. This is really a more difficult task than it
has to be, but I will explain my decision further on. It might be easier for a developer to go about creating a
language with any other environment. A truly dedicated engineer might have already targeted a platform and decided to
start from the get-go with handwritten assembly language. This is not a very portable solution. Another endeavoring
developer might start with a modern language like Java or C#. These are great choices and I would encourage picking up
whatever tools you are familiar with. For reasons that may or may not be obvious, I would recommend fashioning this
dynamic language we are creating in a static host environment. So, a language like Go would be preferred to Scheme. I
know that a number of courses teach programming langauges by implementing a Scheme or Lisp interpreter. What we are
doing is not far off from that except for large differences in style and syntax. And usually these courses require
writing the language interpreter in the language itself, or involve a similar kind of task. The reason I would
recommend staying away from this course is because it begins to lead to deliberately impractical solutions. The
performance you can get from a self-hosted environment is really only half-way there, and any of the benefits of the
solution we are creating tend to get taken away. Forging special frontends for a language can be very useful, and
there are ways to improve productivity and performance by creating a special dialect, but what we are going for is an
entirely new language.


I will note that there is one way that implementing this dynamic language in an interpreted or scripting langauge
might be helpful, and that would be in the case of cross-compiling. I.e. having our 'duck' code reduced to some other
language, like JavaScript or Python, before being executed. That would be fine but then we are dealing with code 
compiling and other complexities which are best addressed in section II of this series.


Reasons supporting my decision to use C are chiefly:

A) That it is portable. A program written in C can be deployed on virtually any operating system. It is easy to
compile for any device and can be executed on almost any microprocessor. Especially when this code is written with the
standards in mind. I would also choose to avoid any new features in the language to maximize compatibility.


B) It is low-level. Without delving into assembly and machine code itself, C represents a close barrier to the target
machine itself. Not knowing what CPU our code will execute on or the exact layout of registers, we want to be aware of
what sort of instructions will be executed and how our program will be laid out in memory.


C) It is not garbage collected. While this point can be debated, the purpose of this project is in ways to create
something new. So, creating a new language with higher level features, it is easier to feel a sense of accomplishment
when we are working on featuers we didn't have from the start. This also helps us to have control over the runtime for
our final interpreter, as we are being dealt the responsibility of memory management ourselves.


D) It's difficult. This hardly qualifies as a reason. I would say that one reason would be 'because it is fast,' but
knowing the internals of some parts of the 'duck' language's stack I know that's hardly the case. Because of the added
complexity of using some better data structures (in terms of code writing), certain elements of table generation are
extremely slow and at times redundant. However, this does not carry over into the runtime (hopefully).


Now this doesn't particularly qualify as a language concern, but as an additional challenge, the project is being
built from the ground-up, in a pulled-up-by-its-own-bootstraps kind of way. That means, as you see the parts of
development around the parser and lexer coming together, two fundamental components of a language, these will all be
hand-fashioned parts and tools. The inner workings of the parser generator will be explained. This completely
overlooks any debates that might be had about tools to use like Bison, Flex, Yacc, or Lex. A number of tools that I
know very little about.


We are also working from our own frontend and our own backend. That means we aren't plugging into LLVM or tying in
with Clang or anything like that. We are using an ANSI compiler with a CMake script. 


If you yourself are developing something in parallel with this project or on your own, I would recommend using all of
these great tools and resources, which would speed-up your journey significantly and you could be done with your
project before you finish reading this guide.


Even considering our target platform and the technical specs we are about to jump through, there will be very little
actual code provided or directly cited. There may be resources attached, but, in general things will be outlined from
an algorithmic approach or described in psuedo-code. There is no reason to go about annotations on a line by line
basis.


#### Part 4: The Lexer
#### Part 5: The Parser
#### Part 6: The Interpreter
#### Part 7: The Library
#### Part 8: The Garbage Collector
#### End Notes
