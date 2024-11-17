+++
title = 'Kamin Interpreters - Basic evaluation'
date = 2024-11-16T21:16:53+01:00
draft = false
genres = ['interpreters', 'programming languages']
tags = ['scala']
+++
We have now handled the lexer and parser for the Basic language in a way that anticipates modifications to support 
other languages discussed in Kamin's book, *"Programming Languages: An Interpreter-Based Approach."* It's now time to 
implement the second part of **Eval** and then **Print** in the Read-Eval-Print loop, completing the interpreter for 
Basic. My expectation is that this post will not be as extensive as the previous one — which is a relief!
## Revisiting Basic
As previously mentioned, Basic consists of two main constructs:
1. **Function Definitions**, such as:  
   `(define double (x) (+ x x))`
2. **Expressions That Can Be Evaluated**, such as:  
   `(double 5)`

These two constructs communicate through a **symbol table** called the Function Definition Table:
- **Function Definitions** insert function definitions into the table.
- **Expressions** read from this table.

## Function Definitions
The handling of **Eval** and **Print** for function definitions is straightforward. As mentioned, the function 
definition is registered in the table and then printed, as illustrated below:
```
BasicParser.parse(PeekingIterator[Token](BasicLexer.tokens(input))) match
  case Left(e) => error(e)
  case Right(f: FunctionDefinitionNode) =>
    println(registerFunction(f))
```
A function definition would proceed approximately as follows:
```
-> (define +1 (x) (+ 1 x))
+1
```
## Expressions
The evaluation of expressions is more complex for several reasons:
1. **Environment**:  
   An environment is used to store variables as they are defined or modified.
2. **Expression Nodes**:  
   Expressions are evaluated through a shared trait, `ExpressionNode`, which represents the structure of expressions.
### Environment
During evaluation, variables can be created or modified using `set`. Additionally, parameters with values may be passed 
when a defined function is called. To manage this, we use an **Environment**:
```
sealed trait Environment:
  def get(key: String): Option[Int]
  def set(key: String, value: Int): Unit
  def openScope(keys: Seq[String]): Unit
  def closeScope(): Unit
```
#### Basic Environment in Basic
In the Basic language, the environment is relatively simple, inspired by Pascal. It consists of two layers:
- **Global Environment**: Always present, forming the base layer for evaluation.
- **Local Environment**: Temporarily created when a function is evaluated. It contains the function’s parameters and their corresponding values.
#### Environment Behavior
- **Global Variables**: The global environment is the fallback for variable lookups. However, global variables cannot be updated from within a function.
- **Local Variables**: Variables created or modified via `set` are handled in the local environment if it exists and the variables exist in it. If no local environment is present or the variable does not exist in the local environment, they are managed in the global environment.

In other words: the local scope only contains parameters, while variables are in the global scope. 
This is something to keep in mind.
#### Function Call and Environment
When a function is called:
1. A **new local environment** is created for the called function, shadowing the existing local environment.
2. Parameters and their values are stored in this new environment.
3. Upon returning from the function:
    - The current local environment is destroyed.
    - The previous local environment for the calling function is restored.
### Expression Nodes
Expression nodes are evaluated using extensions, much like extensions of classes and objects. However, this time, the 
focus is on **extending traits**. Here's why this approach is useful:
1. **Seam for Adding Functionality**:  
   By extending traits, we can introduce new functionality without altering existing code.
2. **Avoiding Large AST Classes**:  
   Building massive AST classes with mixed functionality can be cumbersome. With extensions, we modularize the implementation, allowing targeted functionality like type-checking to be added as separate extensions.
3. **Handling Multiple Trait Implementations**:  
   Extensions enable custom handling for each implementation of a trait. This ensures each evaluation is tailored to the specific realization of the `ExpressionNode` trait.

```
given Evaluator[IntegerExpressionNode] with
  extension (t: IntegerExpressionNode) override def evaluate(using environment: Environment)(using functionDefinitionTable: FunctionDefinitionTable): Either[String, Int] =
    Right(t.integerValue)

given Evaluator[VariableExpressionNode] with
  extension (t: VariableExpressionNode) override def evaluate(using environment: Environment)(using functionDefinitionTable: FunctionDefinitionTable): Either[String, Int] =
    environment.get(t.variableExpression) match
      case Some(value) => Right(value)
      case None => Left(unrecognizedName(t.variableExpression))
```
#### Why Extend Traits?
Extending traits is slightly more complex than extending classes or objects because:
- A trait can have multiple implementations.
- Each implementation may require a unique handling strategy.

However, this complexity enables highly focused evaluations for each implementation. 
This aligns with the **Single Responsibility Principle**, ensuring that:
- Evaluations are easier to maintain.
- Each evaluation can be tested independently.

Yet another fantastic feature with Scala.
#### Evaluation at the Trait Level
Sometimes, evaluation might need to occur at the trait level itself rather than on specific implementations. 
For instance, someone might call an evaluation method on the trait directly instead of on an object implementing the trait.
To handle this:
- A **default evaluation method** at the trait level can be provided.
- This method delegates to the appropriate implementation, as illustrated below.

```
given Evaluator[ExpressionNode] with
  extension (t: ExpressionNode) override def evaluate(using environment: Environment)(using functionDefinitionTable: FunctionDefinitionTable): Either[String, Int]=
    t match
      case n:IntegerExpressionNode =>
        summon[Evaluator[IntegerExpressionNode]].evaluate(n)(using environment)(using functionDefinitionTable)
      case n: VariableExpressionNode =>
        summon[Evaluator[VariableExpressionNode]].evaluate(n)(using environment)(using functionDefinitionTable)
....
```
With the groundwork laid for parsing and evaluating expressions using the `ExpressionNode` trait and its extensions, 
we can now extend the **Read-Eval-Print Loop** to handle expressions effectively.
```
BasicParser.parse(PeekingIterator[Token](BasicLexer.tokens(input))) match
  case Left(e) => error(e)
  case Right(f: FunctionDefinitionNode) =>
    println(registerFunction(f))
  case Right(e: ExpressionNode) =>
    evaluateExpression(e).foreach(println)
    
def evaluateExpression(e: ExpressionNode): Option[Int] =
    e.evaluate match
      case Left(e) =>
        error(e)
        None
      case Right(value) => Some(value)
```
An example is:
```
-> (+1 4)
5
```
## Examples
### Sigma

The sigma function calculates the sum from `m` to `n`:
```
sigma(m, n) = m + (m + 1) + (m + 2) + .... + n
```
This can be implemented in Basic as follows:
```
(define sigma (m n) 
   (if (> m n) 0 (+ m (sigma (+ m 1) n))))
```
A few runs:
```
-> (sigma 1 5)
15
-> (sigma 7 6)
0
```
### Fibonacci
The Fibonacci sequence is defined as follows:
```
(fib 0) = 0
(fib 1) = 1
(fib m) = (fib m-1) + (fib m-2), m > 1
```
This can be implemented in Basic as follows:
```
(define fib (m) 
   (if (< m 1) 0 
      (if (= m 1) 1 
         (+ (fib (- m 1)) (fib (- m 2))))))
```
A few runs:
```
-> (fib 0)
0
-> (fib 1)
1
-> (fib 2)
1
-> (fib 3)
2
-> (fib 4)
3
-> (fib 5)
5
-> (fib 6)
8
```
## Extending the Language
By default, Basic does not provide a way to take input from the user during evaluation. Kamin himself suggests an 
exercise to extend Basic with a `read` operation, which takes a number as input and returns it as the result.
### Steps to Extend Basic with Input Support
1. **New Token:** A `ReadToken` was introduced to represent the new `read` operation.
2. **Lexer Update:** The `BasicLexer` was updated to recognize `read` and tokenize it as a `ReadToken`.
3. **New AST Node:** A `ReadExpressionNode` was added to represent the `read` operation in the abstract syntax tree.
4. **New Parser:** A `ReadExpressionNodeParser` was implemented to parse the `ReadToken` and construct a `ReadExpressionNode`.
5. **Parser Extension:** The `BasicExpressionNodeParser` was updated to extend `ReadExpressionNodeParser`, integrating the new parsing logic seamlessly.
6. **New Evaluator:** An `Evaluator[ReadExpressionNode]` was created to handle the evaluation of `ReadExpressionNode`, which involves reading input from the user.
### Result
And that's it! The built-in **seams** appear to work effectively, making it straightforward to add new functionality.
### Testing the New Functionality
Let's test the new `read` operation:
```
-> (+ 1 (read))
4
5
-> (read)
mikkel
Error: mikkel not a valid integer!
```

## Conclusion
This post concludes the development of the Basic interpreter. We now have an implementation that can serve as a 
foundation for further extensions. Along the way, we had the opportunity to leverage some of Scala's truly elegant 
features. These features make it possible to introduce **"seams"**—points where extending and building upon the 
implementation becomes straightforward and seamless.

My hope is that for the upcoming languages, the focus will primarily be on the 
language itself and less on the mechanics of the interpreter, except where relevant extensions are required, 
utilizing the "seams". Whether this holds true when reality strikes is another question. But I believe it will be an 
exciting journey.

As always, please check the [GitHub repository](https://github.com/mikkela/scala.git) for the code.
