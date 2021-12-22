# object
it's singleton and has no constructor at all, however, we can have an apply() method
```scala
object Abc{
def apply(arg: String): Int = 12
}
println(Abc("junk"))
```
output:
```
12
```
we can not use `new` operator with scala `object` definitions like above in order to create another `object`,  
that's why they are designed as singleton.  

another ex:
```scala
object Car {
  val numberOfWheels = 4

  def run(): Unit = {
    val currentDateAndTime: Date = new Date(System.currentTimeMillis())
    println(s"I am a new car running on $currentDateAndTime!")
  }
}
```

# case object
A case object inherits all the features of objects, and extends them further:  

1. A default implementation of serialization
2. Pattern matching
3. Enumeration
4. A default implementation of toString

All of the above features are not supported on `object` definitions.  

Ex:
```scala
case object Bicycle {
  val numberOfWheels = 2

  def run(): Unit = {
    val currentDateAndTime: Date = new Date(System.currentTimeMillis())
  }
}
```

# class

# case class

# companion class

# companion object

# trait

# sealed class

# sealed trait

# abstract class

# Mixins


