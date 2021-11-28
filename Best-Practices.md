# Scala Coding Practices:
**At the application level:**  
- At the big-picture, application-design level, follow the 80/20 rule, and try to write
80% of your application as pure functions, with a thin layer of other code on top of
those functions for things like I/O.  
- Learn “Expression-Oriented Programming” (Recipe 20.3).  
- Use the Actor classes to implement concurrency (Chapter 13).  
- Move behavior from classes into more granular traits. This is best described in the
Scala Stackable Trait pattern.  

**At the coding level:**  
- Learn how to write pure functions. At the very least, they simplify testing.  
- Learn how to pass functions around as variables (Recipes 9.2 to 9.4).  
- Learn how to use the Scala collections API. Know the most common classes and
methods (10 and 11).  
- Prefer immutable code. Use vals and immutable collections first (Recipe 20.2).  
- Drop the null keyword from your vocabulary. Use the Option/Some/None and Try/
Success/Failure classes instead (Recipe 20.6).  
- Use TDD and/or BDD testing tools like ScalaTest and specs2  

**Outside the code:**  
- Learn how to use SBT. It’s the de-facto Scala build tool (Chapter 18).  
- Keep a REPL session open while you’re coding (or use the Scala Worksheet), and
constantly try small experiments (Recipes 14.1 to 14.4, and many examples
throughout the book).  

# Scala Coding Style: 
1. https://docs.scala-lang.org/style/  
2. http://twitter.github.io/effectivescala/

**Reference:**   
1. Scala Cookbook
2. https://s3-ap-southeast-1.amazonaws.com/tv-prod/documents%2Fnull-Scala+Cookbook.pdf

