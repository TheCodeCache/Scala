# Scala Views - 

`Transformers`:  are nothing but the transformations that we apply on the collections or a group of values in general.  

There are two principal ways to implement `transformers`.  
1. `strict`:  
  that is a new collection with all its elements is constructed as a result of the transformer.  
2. `non-strict` or `lazy`:  
  that is one constructs only a proxy for the result collection, and its elements get constructed only as one demands them.  
```scala
def lazyMap[T, U](coll: Iterable[T], f: T => U) = new Iterable[U] {
  def iterator = coll.iterator map f
}
```
Note: that `lazyMap` constructs a new Iterable without stepping through all elements of the given collection coll.  
The given function `f` is instead applied to the elements of the new collection's iterator as they are demanded.  

Scala `collections` are by default `strict` in all their transformers, except for `Stream`,  
which implements all its transformer methods lazily.  
However, there is a systematic way to turn every collection into a lazy one and vice versa,  
which is based on collection views.  
A `view` is a special kind of collection that represents some base collection, but implements all transformers lazily.  

>If `xs` is some collection, then `xs.view` is the same collection, but with all transformers implemented lazily.  
>and, to get back from a `view` to a `strict collection`, you can use the `force` method.  

```scala
scala> val v = Vector(1 to 10: _*)
v: scala.collection.immutable.Vector[Int] =
   Vector(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
scala> v map (_ + 1) map (_ * 2)
res5: scala.collection.immutable.Vector[Int] =
   Vector(4, 6, 8, 10, 12, 14, 16, 18, 20, 22)
```

> In the last statement, the expression v map (_ + 1) constructs a new vector which is then transformed into a third vector by the second call to map (_ * 2).  
> In many situations, `**constructing the intermediate result from the first call to map is a bit wasteful**`.  
> In the example above, `**it would be faster to do a single map with the composition of the two functions (_ + 1) and (_ * 2)**`.  
> If you have the two functions available in the same place you can do this by hand.  
> But quite often, successive transformations of a data structure are done in different program modules.  
> Fusing those transformations would then undermine `modularity`.  
> A more general way to avoid the intermediate results is by turning the vector first into a view,  
> then applying all transformations to the view, and finally forcing the view to a vector:  

```scala
scala> (v.view map (_ + 1) map (_ * 2)).force
res12: Seq[Int] = Vector(4, 6, 8, 10, 12, 14, 16, 18, 20, 22)
```

There are two reasons to consider using `views`.  
The first is `performance`, as we have seen that by switching a `collection` to a `view` the construction of intermediate results can be avoided  

Usecase:  
1. The problem of finding the first palindrome in a list of words - Views would be better than Collections

# Best Practice - 
1. Apply `views` in `purely functional code` where collection transformations do not have side effects
2. Apply `views` over `mutable collections` where all modifications are done explicitly
3. We must `avoid` to apply a mixture of `views` and `operations` that create new collections while also having side effects



**Reference:**  
1. https://docs.scala-lang.org/overviews/collections/views.html

