+++
title = 'Kamin Interpreters - LISP and meta programming'
date = 2024-12-16T17:02:31+01:00
draft = false
genres = ['interpreters', 'programming languages']
tags = ['scala']
+++
The time has come to take the next step—LISP. LISP is primarily composed of symbols and lists, which provides a variety 
of opportunities to experiment with meta-programming. Programming the programmer, for example, by building a LISP 
interpreter directly in LISP (or at least partly). That will be today’s goal.
## LISP

Now we must be able to implement LISP on top of the infrastructure without major changes, right? Well, 
we're almost there.

There is, however, one small change required before I start working on the LISP interpreter and a set of LISP examples.

### True or False?
During the major cleanup we did for APL, where we decoupled the infrastructure from the values being worked with, 
we missed one thing (there are probably more oversights to address, we'll see). In Basic and APL, true and false are 
handled as 0 and 1, respectively. In LISP, we use different values for the same purpose. 
Here, the symbol `T` represents true, and the empty list `()` represents false. We’ll revisit both values.

To support this, we introduce another trait into the evaluation process:
```scala 3
trait BooleanDefinition:
  def trueValue: Value
  def falseValue: Value

trait ExpressionEvaluator[T <: ExpressionNode]:
extension (t: T) def evaluateExpression(using environment: Environment)
                                       (using functionDefinitionTable: FunctionDefinitionTable)
                                       (using reader: Reader)
                                       (using booleanDefinition: BooleanDefinition): Either[String, Value]

```
This trait will then be used in the evaluation of the relational operators:
```scala 3
given Relational[IntegerValue, IntegerValue] with
  override def equal(operand1: IntegerValue, operand2: IntegerValue)(using booleanDefinition: BooleanDefinition): Either[String, Value] =
    Right(if operand1.value == operand2.value then booleanDefinition.trueValue else booleanDefinition.falseValue)

  override def greaterThan(operand1: IntegerValue, operand2: IntegerValue)(using booleanDefinition: BooleanDefinition): Either[String, Value] =
    Right(if operand1.value > operand2.value then booleanDefinition.trueValue else booleanDefinition.falseValue)

  override def lessThan(operand1: IntegerValue, operand2: IntegerValue)(using booleanDefinition: BooleanDefinition): Either[String, Value] =
    Right(if operand1.value < operand2.value then booleanDefinition.trueValue else booleanDefinition.falseValue)
```

Additionally, the false value is used in the evaluation of constructs like `Begin`:
```scala 3
given ExpressionEvaluator[BeginExpressionNode] with
  extension (t: BeginExpressionNode) override def evaluateExpression(using environment: Environment)
                                                                    (using functionDefinitionTable: FunctionDefinitionTable)
                                                                    (using reader: Reader)
                                                                    (using booleanDefinition: BooleanDefinition): Either[String, Value] =
    t.expressions.foldLeft[Either[String, Value]](Right(booleanDefinition.falseValue)) { (acc, expr) =>
      acc match
        case Left(error) => Left(error)
        case Right(_) => expr.evaluateExpression
    }
```
and `While`:
```scala 3
given ExpressionEvaluator[WhileExpressionNode] with
  extension (t: WhileExpressionNode) override def evaluateExpression(using environment: Environment)
                                                                    (using functionDefinitionTable: FunctionDefinitionTable)
                                                                    (using reader: Reader)
                                                                    (using booleanDefinition: BooleanDefinition): Either[String, Value] =
    @tailrec
    def evaluateLoop(): Either[String, Value] =
      t.testExpression.evaluateExpression match
        case Left(error) => Left(error)
        case Right(test) if !test.isTrue => Right(booleanDefinition.falseValue)
        case Right(_) =>
          t.bodyExpression.evaluateExpression match
            case Left(error) => Left(error)
            case Right(_) => evaluateLoop() // Recur to continue the loop

    evaluateLoop()
```
where the result of both is the false value.

And that’s it. We’re not there yet, but almost.

### LISP a la Kamin

Kamin uses a reduced version of LISP that is based on Basic, and employs three different types of values:

- Integers like `3`
- Symbols like `dkk`
- Lists as a combination tool: `(3 dkk)`

All of this can be used in **Quoted Expressions**, e.g., `'(3 DKK)`, which are expressions that should be treated as-is and not evaluated. These are evaluated by using two values.

#### Lists

Lists are collections of `Values`:
```scala 3
case class ListValue(value: List[Value]) extends Value:
  override def isTrue: Boolean = this != ListValue.nil
  override def toString: String = value.mkString("(", " ", ")")

object ListValue:
  def nil: ListValue = ListValue(List.empty)
```
**False**: The boolean value `false` is represented as the empty list: `()`.

There are several operators defined on lists.

##### Cons

The `cons` operator creates a list from two elements, where the last element is required to be a list:
```
->(cons 'a '(b c))
(a b c)
```
To create an initial list the last element must be the empty list:
```
->(cons 'a '())
(a)
```
If the first element already is a list, a list of lists is created (this can be continued recursively if needed):
```
->(cons '(a b c) '(e f g))
((a b c) e f g)
```
##### Car

The `car` operator returns the first element of a list:
```
->(car '(a b c d))
a
```

If the list is empty, it returns an error:
->(car '())
Invalid parameters for car. Requires a non-empty list

##### Cdr

The `cdr` operator returns the rest of the list:
```
->(cdr '(a b c d))
(b c d)
```

If the list is empty, it returns an error:
```
->(cdr '())
Invalid parameters for cdr. Requires a non-empty list
```

### Symbols

The second type of value is **symbols**:
```
->'a
a
```

Symbols represent arbitrary elements with symbolic meanings. And can be arbitrarily complex:
```
->'(+ 2 3)
(+ 2 3)
```
An example of a symbol is the boolean value `true`, represented as `T`:
```
->(= 1 1)
T
```
We will use symbols continuously in LISP.
### Predicates
Finally, LISP extends Basic with a set of predicates:

- **list?**: Tests whether the value is a non-empty list.
- **number?**: Tests whether the value is a number.
- **null?**: Tests whether the value is an empty list.
- **symbol?**: Tests whether the value is a symbol.

```
->(list? '( 1 2 3))
T
->(list? 1)
()
->(list? '())
()
->(number? 5)
T
->(number? '(5))
()
->(null? '())
T
->(null? '(5))
()
->(null? 5)
()
->(symbol? 'a)
T
->(symbol? '(a))
()
```
### LISP Evaluator
To conclude, let’s create a small LISP evaluator written in LISP itself.

We want this evaluator to handle expressions like:
```
->(eval '(+ 3 (* 4 5)))
23
```
We define the evaluation function:
```
->lisp
->(define eval (exp) (if (number? exp) exp (apply-op (car exp) (eval (cadr exp)) (eval (caddr exp)))))
eval
```
We also define the `apply` function:
```
->(define apply-op (f x y) (if (= f '+) (+ x y) (if (= f '-) (- x y) (if (= f '*) (* x y) (if (= f '/) (/ x y) 'error!)))))
apply-op
```
Finally we need some helper functions:
```
->(define cadr (l) (car (cdr l)))
cadr
->(define caddr (l) (car (cdr (cdr l))))
caddr
```

And now, we’re ready to calculate:
```
->(eval '(+ (* 4 (/ 10 2)) (- 7 3)))
24
```

Personally, I'm stunned how the flexibility of LISP can be utilized.
### Conclusion

That wraps up LISP. This section was particularly fun to create for two reasons:

1. **The Infrastructure**  
   The infrastructure is beginning to take shape as intended. Apart from a small change to handle boolean values, we were able to build LISP directly on top of the infrastructure.

2. **LISP and Symbolic Programming**  
   LISP and symbolic programming are inherently fun. I would have loved to work on a macro interpreter as an extension (as described in one of Kamin’s exercises), but that will have to wait.

### Next Steps
In the next couple of sections, we’ll dive into functional programming, viewed through Kamin’s lens.

