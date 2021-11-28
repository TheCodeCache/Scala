

We can pass a `block of code` to a function as regular parameter, the syntax is as follows:  

```scala
package com.profile.codeblock

object Timer extends scala.App {

  def timer[A](blockOfCode: => A) = {
    val startTime = System.nanoTime
    val result = blockOfCode
    val stopTime = System.nanoTime
    val delta = stopTime - startTime
    println("result::", result)
    (result, delta / 1000000d)
  }

  val (result, time) = timer { println("Hello"); var x = 5; x += 1; x; }
  println("result: ", result)
  println("time: ", time)
}
```

This is how, we can compute the execution time a/c to wall clock.  

**Reference:**  
1. Scala Cookbook

