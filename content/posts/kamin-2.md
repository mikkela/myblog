+++
title = 'Kamin Interpreters - Basic lexing and parsing'
date = 2024-11-16T13:44:28+01:00
draft = false
genres = ['interpreters', 'programming languages']
tags = ['scala']
+++
After briefly introducing my project and the first language, **Basic**, it’s time to delve into something more 
concrete—the implementation of the interpreter for Basic. I’ll explore this in some depth because the Basic interpreter 
will serve as the foundation for interpreters for other languages, which will build on and extend it to handle the 
specific features that make those languages interesting (at least within the context of this blog).

## Overview of the Interpreter

Before diving in, we need to discuss how our interpreters should work overall. As mentioned, we’ll start with Basic, which, as discussed in the previous post, is structured around two types of input:

- **Function definitions**, such as: `(define double (x) (+ x x))`
- **Expressions that can be evaluated**, such as: `(double 5)`

### The Read-Eval-Print Loop

The interpretation of Basic is built around a **Read-Eval-Print Loop**. This loop processes code entered by the user as 
text (**Read**). The text is broken down into individual components (tokens), which are then parsed to construct an 
**Abstract Syntax Tree (AST)** that can be evaluated (**Eval**). The result of this evaluation is returned to the user 
as output (**Print**), after which the user can start again.

The entire process is deliberately kept **lo-fi** to avoid getting lost in details, allowing us to focus on the 
specific programming languages and the unique features they bring to the interpreter.

## What’s Next?

In this blog, we’ll examine the first steps of the Read-Eval-Print loop, including:

1. The Read-Eval-Print loop shell
2. The lexer (tokenization),
2. The parser (understanding the sequence of tokens),
3. And the resulting AST.

In the next post, we’ll complete the loop by looking at:

- The evaluation of the AST,
- And the subsequent printing of the result.

Before moving on, let’s take another look at Basic, now from a more formal perspective.

## Basic Syntax

The syntax used, as previously mentioned, is inspired by Lisp, employing parentheses to enclose individual parts. 
This results in a verbose syntax, but one that is easy to parse and understand. The full syntax is defined below:
```
input           --> expression | fundef
fundef          --> ( define function arglist expression )
arglist         --> ( variable* )
expression      --> value
                --> variable
                --> ( if expression expression expression )
                --> ( while expression expression )
                --> ( set variable expression )
                --> ( begin expression+ )
                --> ( optr expression* )
optr            --> function | value-op
value           --> integer
value-op        --> + | - | * | / | = | < | > | print
function        --> name
variable        --> name
integer         --> sequence of digits, possibly preceeded by minus sign
name            --> any sequence of characters not an integer, and not containing a blank or any of the following characters: ( ) ; 
```
In the description above expression* means zero or more expressions, expression+ means one or more expressions. 
; is used for comments - from ; to end of line is the comment
## Read-Eval-Print Loop
The Read-Eval-Print loop is implemented using a third-party library (**JLine**). Generally, I try to avoid using 
third-party libraries to achieve as direct an implementation in Scala as possible. However, in this case, I decided 
that the benefits were significant enough, especially since this functionality isn’t a central part of the interpreter. 
The library handles features like flags (allowing me to specify which language I’m using) and input history, 
among other things.

Given this, the first version of the implementation looks as follows:
```
var continue = true
while continue do
    var input = lineReader.readLine("->").removeComment()

    if (input == "exit") continue = false
    else
      while !isBalanced(input) do
        input = input + " " + lineReader.readLine(">").removeComment()

      println("Entered: " + input)
```
The code processes the Read-Eval-Print loop via an outer loop. An inner loop ensures that users can press return in 
the middle of their input, delaying evaluation until there are enough closing parentheses to complete the input.

Additionally, any comments are removed from the input before processing. The complete input string is constructed by 
concatenating the entered input strings with added spaces. This approach also means that return cannot be pressed 
arbitrarily; it is only allowed when a logical part of the program, as defined by the grammar above, has been completed.

The return key effectively splits the input into two parts, each aware of its role within the larger context of the 
input being processed.

The balancing check is created as a function local in the file
```
private def isBalanced(input: String): Boolean =
  var stack = List[Char]()

  breakable {
    for (char <- input)
      char match
        case '(' => stack = char :: stack // Push opening parenthesis to stack
        case ')' if stack.isEmpty || stack.head != '(' => break() // Unmatched closing parenthesis
        case ')' => stack = stack.tail // Pop the matching opening parenthesis
        case _ => // Ignore other characters
  }
  stack.isEmpty // If stack is empty, all parentheses were balanced
```
The code leverages Scala's ability to define functions outside of classes. A good example of this is the `main` 
function, which is also defined outside a class. This approach makes it easier to define functionality that doesn’t 
necessarily belong to a specific concept but is simply there to provide assistance.
The removal of comments is placed as a string extensions:
```
implicit class StringExtensions(val s: String) extends AnyVal {
  def removeComment(): String = s.takeWhile(_ != ';')
}
```
One of the advantages in Scala is that I can extend an existing class (in this case, String) with new functionality 
that wasn’t considered when the class was originally created. This serves as an example of a ["seam"](https://martinfowler.com/bliki/LegacySeam.html#:~:text=Michael%20Feathers%20coined%20the%20term,without%20editing%20in%20that%20place”)
—a way to modify behavior without making changes to the original class. Or more accurately, in this context it’s used 
as a way to **add functionality** without altering the existing class.

Although "seam" might not be the perfect term (since we’re adding, not modifying), keep the concept in mind. 
This idea will be a key driver when building the interpreters. 
Ideally, we should be able to add support for a new language to the interpreter without breaking the existing ones.

## Token and Lexing
The first step in processing a program is to divide the program text into parts, called **tokens**, each of which 
represents something we can later attempt to understand. This process is called **lexing**, and before diving into it, 
let’s take a look at how we represent the product of lexing—tokens.

### Token Types in Basic

In Basic, we have four types of tokens:

1. **Tokens from specific text recognition**  
   These are tokens that result from recognizing specific text. Examples include keywords like `if` and `while`, as well as symbols like `)` or `+`. For this type of token, the content is not important (it will always be the same); only the identification matters.

2. **Integers (positive and negative)**  
   These tokens represent integers. We need to recognize not only that they are integers but also capture the value of the integer itself.

3. **Names**  
   These are strings that meet the criteria for variable names, function names, etc. As with integers, we are interested in both the fact that it is a name and the specific string representing the name.

4. **Illegal Tokens**  
   These are strings that, in one way or another, fail to comply with the rules of lexing. Again, we are interested in the specific string and the fact that it is an illegal token.

### Representing Tokens in Scala

Normally, I would represent this with two parts: an **enum** to specify the type and a **value** (literal) to hold 
the string itself. That was also how I implemented the first version.

But then I realized one of the cool features of Scala...
```
trait Token:
  def literal: String

case class NameToken(literal: String) extends Token
case class IntegerValueToken(literal: String) extends Token
case class IllegalToken(literal: String) extends Token

object LeftParenthesisToken extends Token:
  override def literal: String = "("
```
The shared trait `Token` allows us to work with all tokens in a unified way. It is implemented by three different 
**case classes**, each representing the last three types of tokens in our list. In addition to their type, these case 
classes also include the specific value, as described earlier.
The first token type is represented by one of Scala’s most brilliant constructs, and one I appreciate more and more: 
the **singleton object**. There is exactly one object, one single value in the entire world, that represents the 
left parenthesis token `')'`. 
Every time I need to match against a left parenthesis, I use this object. It achieves what I might otherwise have done 
with an enum type but provides a cleaner and more satisfying approach. Whether there are any limitations or issues with 
this approach will become evident further down the road.

### Lexing the text
The lexer is responsible for dividing the received program (provided as text) into relevant **tokens**, which will 
form the foundation for parsing. It achieves this by scanning through the text and, whenever a potential token is 
identified, matching it against one of the known tokens.

This process requires splitting the text into smaller pieces, which is done using **separators**. 
Separators divide the text into distinct parts so that an input like `(if (<> x 1) 1 0)` is not treated as a single 
token but instead as a sequence of tokens.
```
def toToken(text: String): Token =
  text match
    case _ if separators.exists(_.literal == text) => separators.find(_.literal == text).get
    case _ if keywords.exists(_.literal == text) => keywords.find(_.literal == text).get
    case _ if isInteger(text) => IntegerValueToken(text)
    case _ if isName(text) => NameToken(text)
    case _ => IllegalToken(text)
```
When a text fragment is identified, either through whitespace or a separator, it is matched against a set of keywords 
or one of the known types (integer, name, or illegal). But what exactly are **separators** and **keywords**?
These are another example of a built-in **"seam"**, designed to make it (hopefully) possible to use the same lexer 
for all languages by configuring the lexer as shown below:
```
object BasicLexer extends Lexer(
  Seq(LeftParenthesisToken, RightParenthesisToken),
  Seq(EqualToken, LessThanToken, GreaterThanToken, PlusToken, MinusToken, AsteriskToken, SlashToken, PrintToken,
    DefineToken, IfToken, WhileToken, SetToken, BeginToken)
)

class Lexer(separators: => Seq[Token], keywords: => Seq[Token]):
.......
```
Hopefully, this approach will work as we move on to the next languages.
## Parsing the Tokens
The next step (and the final one for this post) is to parse the stream of tokens from the lexer and, based on this 
stream, create an **Abstract Syntax Tree (AST)** that we can later evaluate.
### What is an AST?
An AST is essentially a tree structure where each node represents a construct in the language. Each node contains 
all the relevant information for the construct it represents. For the details, I refer to the code, but let me provide 
an example of a node representing an `if` construct:
```
trait Node
trait InputNode extends Node
sealed trait ExpressionNode extends InputNode

case class IfExpressionNode( testExpression: ExpressionNode,
                             consequenceExpression: ExpressionNode,
                             alternativeExpression: ExpressionNode) extends ExpressionNode
```
The AST is built using a hierarchy of **marker traits**, which allows us to work with the relevant nodes without 
needing to know their specific implementations. This approach is already utilized in the `if` node, and it will be 
further leveraged in the next post when we move on to evaluation.
Since we expect to use pattern matching based on the available expressions, this trait is marked as **sealed**. 
This allows the Scala compiler to verify that the pattern matching is exhaustive, ensuring that all possible cases are 
handled.
### Leveraging Scala's Trait Composition for Parsing
To handle parsing, we utilize yet another fantastic feature in Scala: **trait composition**.

Let’s consider the following scenario:
- We have a common trait `Base` and two traits, `Extended1` and `Extended2`, both extending `Base`.
- `Base` has a method `m` that it implements, and both `Extended1` and `Extended2` provide their own implementations of `m`.
- A class `C` extends both `Extended1` and `Extended2` (and thus, implicitly, `Base`).
When the method `m` is called on `C`, the implementation from the **last listed trait** (e.g., `Extended2`) is used. 
If this implementation calls `super`, the method resolution moves to the **next trait in the sequence** 
(in this case, `Extended1`), and eventually reaches `Base`.
#### Practical Implications for Parsing
What does this mean in practice? It allows us to split the parsing of different constructs into **separate parsers**, 
each with a shared base parser. This structure makes the parsing process modular and easier to manage, as 
illustrated below.
```
trait Parser[ResultType <: InputNode, ParserContextType <: ParserContext]:
  def parse(tokens: PeekingIterator[Token])(using context: ParserContextType): Either[String, ResultType] =
    val peeking = tokens.peek(1)
    if peeking.isEmpty then
      invalidEndOfProgram
    else
      invalidToken(peeking.head)
      
trait IntegerValueExpressionNodeParser extends Parser[ExpressionNode, BasicLanguageFamilyParserContext]:
  override def parse(tokens: PeekingIterator[Token])(using context: BasicLanguageFamilyParserContext): Either[String, ExpressionNode] =
    checkTokensForPresence(tokens, _.isIntegerValueToken) match
      case Right(Seq(value)) =>
        tokens.consumeTokens(1)
        Right(IntegerExpressionNode(value.literal.toInt))
      case _ => super.parse(tokens)


trait VariableExpressionNodeParser extends Parser[ExpressionNode, BasicLanguageFamilyParserContext]:
  override def parse(tokens: PeekingIterator[Token])(using context: BasicLanguageFamilyParserContext): Either[String, ExpressionNode] =
    checkTokensForPresence(tokens, _.isNameToken) match
      case Right(Seq(value)) =>
        tokens.consumeTokens(1)
        Right(VariableExpressionNode(value.literal))
      case _ => super.parse(tokens)
```
The `PeekingIterator` is a wrapper around the token stream, allowing parsers to look ahead and determine whether 
the upcoming tokens can be handled by the current parser.

When `BasicParser`, the parser for Basic, extends both `IntegerParser` and `VariableParser`, a call to `BasicParser` 
to parse will proceed as follows:
1. **IntegerParser** is tried first.  
   If it can handle the next token, it returns the result.
2. If **IntegerParser** cannot handle the token, it calls `super`, which moves the parsing to **VariableParser**.  
   If this parser can handle the token, it provides the result.
3. If **VariableParser** also fails, the process continues to the base **Parser**, which is responsible for returning an error message of some kind.
#### Benefits of This Approach
1. **Focused Parsers**:  
   Each parser becomes specialized, making it easier to test and maintain. This adheres to the **Single Responsibility Principle**.
2. **Another Seam for Flexibility**:  
   This structure creates another **"seam"**—you can add or remove language constructs by choosing whether a parser extends or excludes a specific parser trait.

Below is an illustration of how this works in `BasicParser`:
```
object BasicFunDefNodeParser extends FunctionDefinitionNodeParser

object BasicExpressionNodeParser extends IntegerValueExpressionNodeParser
  with VariableExpressionNodeParser
  with IfExpressionNodeParser
  with WhileExpressionNodeParser
  with SetExpressionNodeParser
  with BeginExpressionNodeParser
  with AdditionExpressionNodeParser
  with SubtractionExpressionNodeParser
  with MultiplicationExpressionNodeParser
  with DivisionExpressionNodeParser
  with EqualityExpressionNodeParser
  with LessThanExpressionNodeParser
  with GreaterThanExpressionNodeParser
  with PrintExpressionNodeParser
  with FunctionCallExpressionNodeParser
  
object BasicParser extends Parser[FunctionDefinitionNode | ExpressionNode, BasicLanguageFamilyParserContext]:
  override def parse(tokens: PeekingIterator[Token])(using context: BasicLanguageFamilyParserContext): Either[String, FunctionDefinitionNode | ExpressionNode] =
    BasicFunDefNodeParser.parse(tokens) match
      case Right(value) => Right(value)
      case Left(_) => BasicExpressionNodeParser.parse(tokens)
```
I think this is a brilliant feature of Scala.

## Wrapping Up
This concludes the first part of interpreting the Basic language. In the next post, I will dive into the details of 
**evaluation**.
The code can be found on [GitHub](https://github.com/mikkela/scala.git).


