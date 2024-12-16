+++
title = 'Kamin Interpreters - Extending with Vectors and Matrices'
date = 2024-12-08T17:01:08+01:00
draft = false
genres = ['interpreters', 'programming languages']
tags = ['scala']
+++
Time for the first test run—can the Basic foundation support layering a new programming language on top? To test this, 
I have implemented [APL](https://en.wikipedia.org/wiki/APL_(programming_language)) (A Programming Language), 
a programming language focused on working with one-dimensional (vectors) and multi-dimensional (matrices) arrays. 
In the Kamin version, we only work with 1 and 2 dimensions, but the concept can, of course, be generalized.

## APL - Vectors and Matrices
Until now, we have worked exclusively with integers. All evaluations result in producing an integer, which is also 
stored in the Environment. The entire implementation is tightly coupled to integers. This needs to be generalized so 
that we can handle arbitrary values—in this case, vectors and matrices as well.

We therefore introduce the concept of **Value**:

```scala 3
trait Value:
  def isTrue: Boolean
```
And implement it in the form of an integer value:
```scala 3
case class IntegerValue(value: Int) extends Value:
  override def isTrue: Boolean = value != 0

  override def toString: String = value.toString
```
This means we had to change the evaluation to:
```scala 3
trait ExpressionEvaluator[T <: ExpressionNode]:
  extension (t: T) def evaluateExpression(using environment: Environment)
                                         (using functionDefinitionTable: FunctionDefinitionTable)
                                         (using reader: Reader): Either[String, Value]

```
And the environment handling to:
```scala 3
sealed trait Environment:
  def get(key: String): Option[Value]
  def set(key: String, value: Value): Unit
  def openScope(keys: Seq[String]): Unit
  def closeScope(): Unit
```
But it also allows us to introduce the concepts of vector and matrix as shown below:
```scala 3
case class VectorValue private(value: Vector[Int]) extends Value:
  override def isTrue: Boolean = this.value.exists(_ != 0)
  override def toString: String = value.mkString(" ") + "\n"

case class MatrixValue private(value: Vector[Vector[Int]]) extends Value:
  override def isTrue: Boolean = value.exists(_.exists(_ != 0))
  override def toString: String = value.map(_.mkString(" ")).mkString("\n")
```

That was easy!
### Arithmetic and Relations

And yet, not entirely. How should we, for example, handle concepts such as addition, given that addition works 
differently depending on whether we're dealing with integers, vectors, or matrices? Not to mention the combinations.

Let’s start by looking at how addition works in the Basic version:
```scala 3
given Evaluator[AdditionExpressionNode] with
  extension (t: AdditionExpressionNode) override def evaluate(using environment: Environment)(using functionDefinitionTable: FunctionDefinitionTable): Either[String, Int] =

    evaluateParameters(Seq(t.operand1, t.operand2), environment, functionDefinitionTable) match
      case Left(error) => Left(error)
      case Right(params) => Right(params.head + params(1))
```
It looks fairly straightforward, but how do we now include addition for two vectors? We have several options:

#### Extending Addition
We could extend addition to also handle vectors by matching on the values we are processing.
```scala 3
given Evaluator[AdditionExpressionNode] with
  extension (t: AdditionExpressionNode) override def evaluate(using environment: Environment)(using functionDefinitionTable: FunctionDefinitionTable): Either[String, Int] =

    evaluateParameters(Seq(t.operand1, t.operand2), environment, functionDefinitionTable) match
      case Left(error) => Left(error)
      case Right(params) =>
        params match
          case List(v1: IntegerValue, v2: IntegerValue) => Right(v1.value + v2.value)
```
This would mean that every time we add a new value where addition makes sense, we’d have to go into the core 
implementation (the handling of addition) and add the logic for this value as well. I’d like to avoid this 
because we aim to layer additional languages on top of the existing Basic, not modify the Basic implementation itself.

### Addition in Value
We could extend each `Value` to handle addition. 
```scala 3
case class IntegerValue(value: Int) extends Value:
  override def isTrue: Boolean = value != 0
  override def addition(value1: Value, value2: Value): Value =
    (value1, value2) match
      case (v1: IntegerValue, v2: IntegerValue) => IntegerValue(v1.value + v2.value)
  override def toString: String = value.toString
```
This makes sense if we are only performing addition between the same types of values. However, because we also want 
addition to work for combinations, such as adding a vector to an integer, this approach requires us to modify 
`IntegerValue` to handle addition with any new `Value`. We face the same problem as before—we are not building on 
top of Basic but modifying Basic every time we add a new language.

### Addition as Context
A third option is to use Scala’s type programming. We could build the evaluation of addition by letting Scala locate a 
definition for addition between the two involved types and apply it. 
```scala 3
def arithmeticOperation[T1 <: Value, T2 <: Value](
                                                   v1: T1,
                                                   v2: T2,
                                                   op: Arithmetic[T1, T2] => (T1, T2) => Either[String, Value],
                                                   opName: String
                                                 )(using dispatcher: ArithmeticDispatcher): Either[String, Value] =
  dispatcher.dispatch(v1, v2) match
    case Some(arithmetic: Arithmetic[T1, T2] @unchecked) => op(arithmetic)(v1, v2)
    case None => Left(s"$opName is not supported for ${v1.getClass} and ${v2.getClass}")

given ExpressionEvaluator[AdditionExpressionNode] with
  extension (t: AdditionExpressionNode) override def evaluateExpression(using environment: Environment)
                                                                       (using functionDefinitionTable: FunctionDefinitionTable)
                                                                       (using reader: Reader)
                                                                       (using booleanDefinition: BooleanDefinition): Either[String, Value] =

    evaluateParameters(Seq(t.operand1, t.operand2), environment, functionDefinitionTable, reader, booleanDefinition) match
      case Left(error) => Left(error)
      case Right(params) => arithmeticOperation(params.head, params(1), _.addition, "Addition")
```
If no such definition exists, addition would return an error.

The realization of addition is achieved by implementing an `Arithmetic` trait that defines the relevant arithmetic functions.
```scala 3
trait Arithmetic[T1 <: Value, T2 <: Value]:
  def addition(operand1: T1, operand2: T2): Either[String, Value]
  def subtraction(operand1: T1, operand2: T2): Either[String, Value]
  def multiplication(operand1: T1, operand2: T2): Either[String, Value]
  def division(operand1: T1, operand2: T2): Either[String, Value]
```
This trait is then implemented for the combinations of types where it makes sense. For example, the Basic foundation includes an implementation of this trait for two `IntegerValues`:
```scala 3
given Arithmetic[IntegerValue, IntegerValue] with
  override def addition(operand1: IntegerValue, operand2: IntegerValue): Either[String, Value] =
    Right(IntegerValue(operand1.value + operand2.value))

  override def subtraction(operand1: IntegerValue, operand2: IntegerValue): Either[String, Value] =
    Right(IntegerValue(operand1.value - operand2.value))

  override def multiplication(operand1: IntegerValue, operand2: IntegerValue): Either[String, Value] =
    Right(IntegerValue(operand1.value * operand2.value))

  override def division(operand1: IntegerValue, operand2: IntegerValue): Either[String, Value] =
    if operand2.value != 0 then
      Right(IntegerValue(operand1.value / operand2.value))
    else
      cannotDivideWithZero
```
APL extends the implementation to include vectors (and matrices) as well as the combination of a vector and an integer.

In the same way, relational operators (`=`, `<`, and `>`) are handled through a `Relational` trait.

#### How do we locate the implementation?
The `Dispatcher` trait and corresponding object handles the actual dispatchment
```scala 3
trait Dispatcher[Op[_ <: Value, _ <: Value]]:
  def dispatch(v1: Value, v2: Value): Option[Op[Value, Value]]
  def orElse(other: Dispatcher[Op]): Dispatcher[Op] =
    (v1: Value, v2: Value) =>
      this.dispatch(v1, v2).orElse(other.dispatch(v1, v2))

object Dispatcher:
  def create[T1 <: Value, T2 <: Value, Op[_ <: Value, _ <: Value]](using
                                                                   tt1: TypeTest[Value, T1],
                                                                   tt2: TypeTest[Value, T2],
                                                                   operation: Op[T1, T2]
                                                                  ): Dispatcher[Op] =
    new Dispatcher[Op]:
      def dispatch(v1: Value, v2: Value): Option[Op[Value, Value]] =
        for
          t1 <- tt1.unapply(v1)
          t2 <- tt2.unapply(v2)
        yield operation.asInstanceOf[Op[Value, Value]]
```
This trait provides two key methods:

1. **Dispatch**: A method that performs the dispatch by searching through potential implementations.
2. **orElse**: A method to register an implementation of the relevant trait.

#### The Weak Spot
This registration step is arguably the weakest point. It requires modifying the central registry whenever new `Arithmetic` and `Relational` implementations are added. 
```scala 3
given ArithmeticDispatcher =
  Dispatcher.create[IntegerValue, IntegerValue, Arithmetic]
    .orElse(Dispatcher.create[IntegerValue, MatrixValue, Arithmetic])
    .orElse(Dispatcher.create[MatrixValue, IntegerValue, Arithmetic])
    .orElse(Dispatcher.create[MatrixValue, MatrixValue, Arithmetic])
    .orElse(Dispatcher.create[VectorValue, IntegerValue, Arithmetic])
    .orElse(Dispatcher.create[IntegerValue, VectorValue, Arithmetic])
    .orElse(Dispatcher.create[VectorValue, VectorValue, Arithmetic])
```
Ideally, I would like to move this registration out so that individual language modules can handle it. However, this doesn’t seem feasible if I want to leverage Scala’s native context management.

I could adopt a more traditional IoC (Inversion of Control) approach, and I would probably do so if the goal of this project wasn’t to explore Scala’s capabilities.

For now, we’ll stick with this approach. To make the process clearer, I’ve separated the configuration from the rest of the infrastructure. This way, only the configuration needs to be touched (apologies in advance for this workaround).
## Evaluation of Expressions

The evaluation of expressions is handled similarly to Basic by providing an `Evaluator` implementation for each of the 
new nodes in the Abstract Syntax Tree. Each implementation is placed in the language it applies to, while the `Evaluator` 
trait resides in the infrastructure—exactly as intended.

However, just as with Basic, evaluation is also needed at the trait level. In the Basic version, we solved this by 
matching on the individual AST types. While we could continue with this approach, it would lead to further infiltration 
of individual programming languages into the infrastructure. Now, in addition to registering `Arithmetic` 
implementations for the relevant `Value` types (annoying but manageable), we would also need to register evaluators for 
AST nodes in a massive match structure. This would become unmanageable.

### A New Approach
In this case, we’ve opted for an IoC (Inversion of Control) and repository approach, defining a `TypeRegistry` 
that we use.
```scala 3
class TypeRegistry[K, V]:
  private val registry = mutable.Map[K, V]()

  def register(key: K, value: V): Unit =
    registry(key) = value

  def get(key: K): Option[V] = registry.get(key)

  def clear(): Unit = registry.clear()

  def unregister(key: K): Unit = registry.remove(key)
```
On this registry, we’ve defined an `ExpressionEvaluatorRegistry`:
```scala 3
object ExpressionEvaluatorRegistry:
  private val registry = new TypeRegistry[Class[? <: ExpressionNode], ExpressionEvaluator[? <: ExpressionNode]]()

  def register[T <: ExpressionNode](cls: Class[T], evaluator: ExpressionEvaluator[T]): Unit =
    registry.register(cls, evaluator.asInstanceOf[ExpressionEvaluator[? <: ExpressionNode]])

  def get[T <: ExpressionNode](cls: Class[T]): Option[ExpressionEvaluator[T]] =
    registry.get(cls).asInstanceOf[Option[ExpressionEvaluator[T]]]

  def clear(): Unit = registry.clear()
```
This replaces the central match structure:
```scala 3
given ExpressionEvaluator[ExpressionNode] with
  extension (t: ExpressionNode) override def evaluateExpression(using environment: Environment)
                                                               (using functionDefinitionTable: FunctionDefinitionTable)
                                                               (using reader: Reader)
                                                               (using booleanDefinition: BooleanDefinition): Either[String, Value]=

  ExpressionEvaluatorRegistry.get(t.getClass) match
    case Some(evaluator) => evaluator.asInstanceOf[ExpressionEvaluator[t.type]].evaluateExpression(t)(using environment)(using functionDefinitionTable)(using reader)(using booleanDefinition)
    case None            => Left(s"No evaluator registered for ${t.getClass.getName}")

```
The remaining task is to adapt this registry for use in each language.

This approach allows us to keep infrastructure and programming languages separate. 
It also ensures that not all language constructs need to be evaluable in every language (which makes sense), and even allows us to modify the evaluation of shared constructs from one language to another.
## Organization
The final step is to organize the code into meaningful modules. Generally, the core infrastructure package is contained within the `kamin` package. APL is placed in `kamin.apl`, and Basic in `kamin.basic`.

Additionally, there is a configuration file in each of the packages, containing the configurations specific to each language: Basic and APL.

## APL Examples

Before I conclude this blog, let's have a bit of fun with APL.

```
-> apl
->(define fac (n) (*/ (indx n)))
fac
->(fac 5)
120
```
This used two APL operators:
* */ - reduction using multiplication - all values in a vector (or matrix) are multiplied. This operator also exists as +/, -/ and // and max/, or/ and and/
* indx - index generation - returns an vector with the values from 1 up til the argument

```
->(define avg (v) (/ (+/ v) (shape v)))
avg
->(avg '(4 5 6))
5
```
Again are we using an reduction operator (this time +/), but also the shape operator returning the shape of an integer, vector or matrix

```
->(set m (restruct '(4 4) '(1 1 0 0 0)))
1 1 0 0
0 1 1 0
0 0 1 1
0 0 0 1
```

The last example shows how to restruct i vector. Only vectors can be defined using ' and to use them as matrix, the restruct operator is used. I will form the vector into matrix form, eventually repeating or cutting off values if needed.
## First Generalization
We have achieved the goal of building an infrastructure that, at almost every level, can be adapted to individual programming languages. This allows new languages to be built *on top of* the infrastructure rather than being integrated *into* it. The only remaining detail is the handling of `Arithmetic`/`Relational` implementations for the relevant `Value` types, which we will accept for now.

The next project will be extending the system to handle **LIPS**. By now, that should be straightforward.
