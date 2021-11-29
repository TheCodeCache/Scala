# TODO

**Implicit Class:**  

**Implicit Function:**  

**Implicit Parameter:**  

**Implicit Imports:**  
1. import java.lang._  
2. import scala._  
3. import Predef._  

the 2nd scala._ overrides the conflicting classes from the previoud java.lang._  
for ex: scala.StringBuilder overrides java.lang.StringBuilder  

scala.collection.mutable.HashMap is equivalent to collection.mutable.HashMap,  
because scala._ package is already imported,  

Predef._ package contains some of the most common or useful methods,  
these useful methods could have been part of scala._ package itself,  
but the scala._ package was introduced much later than Predef._  
Hence, we still use Predef._ pacakge  

