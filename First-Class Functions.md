
As with other types of data such as `String` and `Int`,  
functions have types based on the types of their input arguments types and their return value type.  
A function can be  
stored in a value or variable,  
passed to a function,  
returned from a function,  
and used with data structures, to support  `map()`, `reduce()`, `fold()`, `filter()` etc. higher-order functions.  

**Function Types and Values:**  
The type of a function is a simple grouping of its input types and return value type,  
Syntax:  A function type -  
**( [type, ...] ) => type**  

For ex:  
```scala
scala> def double(x: Int): Int = x * 2  
double: (x: Int)Int  

scala> double(5)  
res0: Int = 10  

scala> val myDouble: (Int) => Int = double  
myDouble: `Int => Int = <function1>`  

scala> myDouble(5)  
res1: Int = 10  

scala> val myDoubleCopy = myDouble  
myDoubleCopy: `Int => Int = <function1>`  

scala> myDoubleCopy(5)  
res2: Int = 10  

`Int => Int` indicates a function type accepting one integer type data that returns integer type value.  

scala> val myDouble = double _  
myDouble: Int => Int = <function1>  

scala> val amount = myDouble(20)  
amount: Int = 40  
```

**Function Literals:**  
This can be thought as a Lambda (java/python) or Anonymous functions.  
It's a working function that lacks a name, and assign it to a new function value.  

For ex:
```scala
scala> val doubler = (x: Int) => x * 2
doubler: Int => Int = <function1>

scala> val doubled = doubler(22)
doubled: Int = 44
```
The function literal in this example is the syntax `(x: Int) => x * 2`  
Function literals can be stored  
in function values and variables, or defined as part of a higher-order function invocation.  
We can express a function literal in any place that accepts a function type  
function0, function1, function2, .. are also literals

**Higher Order Functions:**  
A `higer-order` function is a function that has a value with a function type as an input parameter or return value.  

Two of the most famous higher-order functions - `map()` and `reduce()`  
map() takes a function paramter and uses it to convert one or more items to a new value and/or type.  
reduce() takes a function parameter and uses it to reduce a collection of multiple items down to a single item.  

**Placeholder Syntax**  
It's a shortened form of function literals, replacing named parameters  
with wildcard operators `(_)`. It can be used when (a) the explicit type of the function is  
specified outside the literal and (b) the parameters are used no more than once  

placeholder syntax reduces the amount of extra code required to call these methods.  
Ex:
```scala
scala> val doubler: Int => Int = _ * 2
doubler: Int => Int = <function1>

scala> def combination(x: Int, y: Int, f: (Int,Int) => Int) = f(x,y)
combination: (x: Int, y: Int, f: (Int, Int) => Int)Int

scala> combination(23, 12, _ * _)
res13: Int = 276

scala> def tripleOp(a: Int, b: Int, c: Int, f: (Int, Int, Int) => Int) = f(a,b,c)
tripleOp: (a: Int, b: Int, c: Int, f: (Int, Int, Int) => Int)Int

scala> tripleOp(23, 92, 14, _ * _ + _)
res14: Int = 2130

scala> def tripleOp[A,B](a: A, b: A, c: A, f: (A, A, A) => B) = f(a,b,c)
tripleOp: [A, B](a: A, b: A, c: A, f: (A, A, A) => B)B

scala> tripleOp[Int,Int](23, 92, 14, _ * _ + _)
res15: Int = 2130

scala> tripleOp[Int,Double](23, 92, 14, 1.0 * _ / _ / _)
res16: Double = 0.017857142857142856

scala> tripleOp[Int,Boolean](93, 92, 14, _ > _ + _)
res17: Boolean = false

```

**Partially Applied Functions and Currying:**
Invoking functions, both regular and higher-order, typically requires specifying all of  
the function’s parameters in the invocation (the exception being those functions with  
default parameter values). `What if you wanted to reuse a function invocation and retain  
some of the parameters to avoid typing them in again?`  

```scala
scala> def factorOf(x: Int, y: Int) = y % x == 0
factorOf: (x: Int, y: Int)Boolean

scala> val f = factorOf _
f: (Int, Int) => Boolean = <function2>

scala> val x = f(7, 20)
x: Boolean = false

// If you want to retain some of the parameters, you can partially apply the function by
// using the wildcard operator to take the place of one of the parameters. The wildcard
// operator here requires an explicit type, because it is used to generate a function value
// with a declared input type:

scala> val multipleOf3 = factorOf(3, _: Int)
multipleOf3: Int => Boolean = <function1>

scala> val y = multipleOf3(78)
y: Boolean = true

```
The new function value, multipleOf3, is a partially applied function, because it contains  
some but not all of the parameters for the factorOf() function.  

A cleaner way to partially apply functions is to use functions with multiple parameter  
lists. Instead of breaking up a parameter list into applied and unapplied parameters,  
apply the parameters for one list while leaving another list unapplied. This is a technique  
known as currying the function:  

```scala
scala> def factorOf(x: Int)(y: Int) = y % x == 0
factorOf: (x: Int)(y: Int)Boolean

scala> val isEven = factorOf(2) _
isEven: Int => Boolean = <function1>

scala> val z = isEven(32)
z: Boolean = true

```

**Partial Functions:**  
All of the functions we have studied so far are known as total functions, because they  
properly support every possible value that meets the type of the input parameters.  
A simple function like `def double(x: Int) = x*2` can be considered a total function;  
there is no input x that the double() function could not process.  

There are some functions that do not support every possible value that meets the input types  
For ex:  
1. a function that returns the square root of the input number  
would certainly not work if the input number was negative  
2. a function that divides by a given number isn’t applicable if that number is zero  

such functions are called partial functions, because they can only partially apply to their input data.  


"What Is the Difference Between Partial and Partially Applied Functions?  
  A partial function, as opposed to a total function, only accepts a partial amount of all possible input values.   
  A partially applied function is a regular function that has been partial‐ly invoked, and remains to be fully invoked (if ever) in the future.   

For ex:
```scala
scala> val statusHandler: Int => String = {
 | case 200 => "Okay"
 | case 400 => "Your Error"
 | case 500 => "Our error"
 | }
statusHandler: Int => String = <function1>

scala> statusHandler(200)
res20: String = Okay

scala> statusHandler(400)
res21: String = Your Error

scala> statusHandler(401)
scala.MatchError: 401 (of class java.lang.Integer)
```

usecase: partial functions more useful when working with collections and pattern matching  
For example, we can "collect" every item in a collection that is accepted by a given partial function  

**Invoking Higher Order Functions with Function Literal Blocks:**  
```scala
scala> def safeStringOp(s: String, f: String => String) = {
 | if (s != null) f(s) else s
 | }
safeStringOp: (s: String, f: String => String)String
scala> val uuid = java.util.UUID.randomUUID.toString
uuid: String = bfe1ddda-92f6-4c7a-8bfc-f946bdac7bc9
scala> val timedUUID = safeStringOp(uuid, { s =>
 | val now = System.currentTimeMillis
 | val timed = s.take(24) + now
 | timed.toUpperCase
 | })
timedUUID: String = BFE1DDDA-92F6-4C7A-8BFC-1394546043987

scala> def timer[A](f: => A): A = {
 | def now = System.currentTimeMillis
 | val start = now; val a = f; val end = now
 | println(s"Executed in ${end - start} ms")
 | a
 | }
timer: [A](f: => A)A
scala> val veryRandomAmount = timer {
 | util.Random.setSeed(System.currentTimeMillis)
 | for (i <- 1 to 100000) util.Random.nextDouble
 | util.Random.nextDouble
 | }
Executed in 13 ms
veryRandomAmount: Double = 0.5070558765221892
```
**Reference:**  
1. Learning Scala Text Book  

