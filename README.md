# Cymling Programming Language
(Formerly called RIPPL)

*Design Unfinished: Comments Welcome!*

 - Nearly homoiconic (like a lisp, with it's own JSON-like data definition syntax)
 - Functional (first class functions, immutability assumed/easier/preferred)
 - Type-safe (with type aliases like ML, not Object Oriented)

Cymling is a rarely used word for a pattypan squash.  It's also one of the few short words in the English language that have the letters ML in it ("Cy***ML***ing") which is a nod to the ML programming language.  ML because it showed me that static typing does not need to be Object Oriented.  I could have nodded to Clojure, JSON, and Java too, but ML was the final missing piece for me.

##Design Goals
 - Immutability is the default
 - Applicative Order by default (eager, not like Haskell)
 - Static Type checking
 - Type inference everywhere except function parameters and return types (the only place type safety is proven to improve comprehension).
 - Regular (lisp-like) syntax with very few infix operators
 - No primitives (int, float, etc.).  Built-in and user-defined data behave the same.
 - Types are related using aliases and set theory (like ML), not through object hierarchies.
 - Evaluative (everything is an expression – no return statements)
 - Built-in data types: Record (tuple): `(a b c)`, Map: `{ a=b c=d e=f }`, List: `[a b c]`, Set: `#{a b c}` - like Clojure except the fundamental unit is a Record instead of a linked list.

##Syntax
The syntax is a combination of Clojure (Lisp) and ML, with maybe a sprinkling of Java (or not).

###Comments

In order to leave the division symbol for division (and to nod to lisp), a semicolon is substituted for the otherwise C-style comment syntax.  Single-line comments are preceded by a semicolon `;`. Multi-line comments start with a semi-star `;*` and end with a star-semi `*;`. Cym-doc comments are multi-line with an extra beginning *:

```
; single line comment
;; A more-visible single-line comment

;*
Multi-line comment
Still in a multi-line comment...
Comment ends here: *;

;**
Multi-line CymlingDoc comment
*;
```

###Commas
Commas are whitespace (like Clojure).  Add them when it helps you, but the compiler treats them as spaces.

###Functions vs. Record literals
Lisp puts the the function name to the right of the parenthesis, which causes plain lists to require quoting:
```
(func arg1 arg2)
'(“Marge” 37 16.24)
```

Cymling puts the function to the left and literal records (not lists) will not require quotes:
```
func(arg1 arg2)    ; function application
(“Marge” 37 16.24) ; record/tuple
```

##Operators
Infix operators allow very Natural-Language friendly syntax, but can quickly lead to confusion with precedence and overriding.  So there will be very limited operators in this language, mostly for the type system.

Operators (in precedence order):
 - `.` for function application
 - `=` associates keys with values in records/maps.  `->` denotes a function referece so that the function is effectively delegeted to.

Type operators (in precedence order):
 - `:` introduces a type
 - `|` ("or") for union types, `&` ("and") for intersection types
 
###Dot Operator
Instead of the usual Lisp convention of reading code inside out:
```
(third (second (first arg1) arg2) arg3)
```
We borrow the “dot operator” from C-style languages to enable a “Fluent Interface” style:
```
first(arg1).second(arg2)
           .third(arg3)
```

##Records
Data definition and function application are borrowed primarily from ML's records.  Thus a data definition looks like this (using the built-in defType function):
```
defType(Person (name:String
                age:Int64
                height:Float64))
```

An instance of that definition using names instead of indices:
```
Person(name=“Marge” age=37 height=16.24)
```

The same using indices only (instead of names):
```
Person(“Marge” 37 16.24)
```

Both of the above examples compile to something like Java objects of an appropriate type with getter methods and a constructor, but no setter methods:
```
name():String
age():Int64
height():Float64

;; Record definition with some defaults and inferred types (name:String and Height:Float32)
;; Note that position matters (all Person's will be defined in the order: name, age, height).
defType(Person {name=“Marge” age:Int height=16.24})

;; Instantiation:
let( { marge=Person{age=37}
       fred=Person{name="Fred" age=35}
       sally=Person("Sally" 15 12.2)} ;; Defined using record syntax instead of map syntax (like ML)

     marge.height() ;; returns 16.24:Float32
     fred.age()) ;; returns 35:Int which is also the return for the entire let block.
```

##Interfaces
Interfaces (not objects) can extend the functionality of records in a pseudo-Object-Oriented way.  When creating data that you intend to treat as implementing an interface, simply attach the name of the interface to the data to give it a type when you construct it:
TODO: Pick one:
A. (This cannot be used with union or intersection types)
```
Person{name=“Marge” age=37 height=16.24}
```
B. (This will work with union or intersection types)
```
{name=“Marge” age=37 height=16.24}:Person
{name=“Marge” age=37 height=16.24}:Person|Employee
```

##Functions
Anonymous functions are defined with their arguments followed by the statements to be executed when they are called.  `λ<T>(args:Record body:T):T` is a built-in function used to create them.  A zero-arg sugary short-form is available as well without any parens: `λ<T>body:T`.  This uses the Lambda character (commonly used to mean, "anonymous function," which is unicode U+03BB.  To type this on Ubuntu (there is no need to type leading zeros) `CTRL-SHIFT-u` `3` `b` `b` `RETURN`.  We might use `^`, `#`, or `@` instead of `λ`, but for now we're using what's easiest to read.

```
myFn:Fn0<String> = λ“hello!” ;; or, more formally: λ({} "hello!")

myFn() ;; returns "hello!"

myFn2:Fn0<Unit> = λ({name:String}
                    "Hello $name$. Pleased to meet you!")

myFn2("Kelly") ;; returns "Hello Kelly. Pleased to meet you!"
```

##Other Operators
All other (non-infix) operators have syntax like other functions.  Because they are functions, they can be lazily evaluated:

```
cond((a λ“a”)           ;; eager if, followed by lazy then
     [(λb λ“b”)         ;; list of lazily evaluated if/then clauses.
      (λc λ“c”)]
     λ“not a, b, or c”) ;; lazily-evaluated else clause
```

The signature of cond is:
```
cond:<T> T = λ(if:     (test:Bool     then:λ<T>)
               elseifs:[(test:λ<Bool> then:λ<T>>)]
               else:   λ<T>)
```
The type signature of that is:
```
Fn3<Tup2<Bool,Fn0<T>>,
    List<Tup2<Fn0<Bool>,Fn0<T>>,
    Fn0<T>,
    T>
```

##Type System

Java Generics, p. 16 by Naftalin/Wadler has an example that shows why mutable collections cannot safely be covariant.  Some day I'll copy it to this document.  In any case, immutable collections *can* be covariant because of their nesting properties.  You can have a `ImList<Int>` and add a `Float64` to it and you'll get back the union type `ImList<Int|Float64>` which can then be safely cast to a `ImList<Number>` (`Number` is the most specific common ancestor of both `Int` and `Float64`).  If the List were mutable, you'd have to worry about other pointers to the same list still thinking it was a `List<Int>`, but immutable collections solve that problem.

That paragraph assumes inheritance and, for Java compatibility, it makes sense.  Cymling's view of Java's inheritance in this case is: `defType Number = Int | Float64` where `Int` and `Float64` are both built-in types.

Therefore, the type system needs some indication of what's immutable (grows by nesting like Russian Dolls) and what's not (changes in place).  Since imutability is the default, there should be an `@Mutable` or similar annotation required in order to update data in place.  Probably a second annotation `@MutableNotThreadSafe` should be required for people who really want to live dangerously.  Without such annotations, your class/interface cannot do any mutation.

There may come a time when the type system needs to choose between being mathematically sound and being lenient.  I am not committed to soundness in the strict sense.  I require a type system to prevent-known-bad, but that's weaker than most type systems which have an allow-known-good outlook.  My experience with Java is that sooner or later you have to cast somewhere in your API.  That seems just a little too strict for me, but this paragraph exceeds the limits of my competence, so basically, I don't know.

##Defining Types
To make a person type, you need:
```
defType(Person             ; start defining a type called “Person”
        { name:String      ; The expected methods are name():String
          age:Int          ;                          age():Int
          height:Float })  ;                      and height():Float
```

Note: If we want inheritance, second line *could* include:
```
        extends(Nameable)  ; Extends and implements are the same
```
But I'd prefer to use type aliases:
```
defAlias(Nameable Person)
```

Here is a let block that performs some pretty simple logic on some people (fred is declared, but never used):

```
let( { marge={name=“Marge” age=37 height=16.24}
       fred:Person={name=“Fred” age=39 height=15.5} 
       sarah:Person=(”Sarah” 35 17.0) }
     cond(gt(marge.height() sarah.height())
           “Marge is taller than Sarah”
           “Marge is not taller than Sarah”))
```

Parameterized types use angle-brackets like Java.

```
defType(List<T>             ; Define a type called “List” with type param T
        fns(get(i:Long):T = ...
            cons(item:U):List<T|U> = ...
        ))  ;                  
```


Main Method
```
main({args:List<String>}
     print(“Hello “ args.getOrElse(1 “World”) “!”)
     1)
```

Let's look at if() for a moment.  It takes at least a then() or else() function for lazy evaluation

```
defType(Then<T>
        { apply():T })

defType(Else<T>
        { apply():T })

defType(If<T>
        { condition:Bool
          then:Then<T>=Nil
          else:Else<T>=Nil })

defType(ElsIf<T>
        extends(Else<T> If<T>)
        { condition:Bool
          then:Then<T>=Nil
          else:Else<T>=Nil })
```

##Strings
*Wish List:*
I'd love to have Str8 (pronounced “Straight”) to be native support for UTF8, have all serialization use UTF8, and give Str8 a .to16() method to convert to Java style strings.  This would require a bunch of work that I don't want to do up front, but over time I think it would be a big win.  Also, it would be good to rewrite the regular expressions library to use this and use it quickly.  When people use emoji, we don't want to be counting “code points” as opposed to characters the way you have to in Java.  UTF8 seems to be the new world standard...

##Pattern Matching
Instead of inheritence, we use pattern matching (like ML).
```
match(item
      [(Person{n:name a:age h:height} λ“Person: $n$”)
       (Car(license year)             λ“Car: $license$")
       (default                       λ“Default”)])
```

Note: Enums values may have to be sub-classes of the Enum they belong to for pattern matching to work.

##Types
Any is the progenitor super-type
Nil is a sub-type of most things
Nothing might be a sub-type of Nil indicating that no valid type can be used?

##Union Types
Are available.  String-or-null is:
```
middleName:String|Nil
;; or maybe like Ceylon:
middleName:String?
```

Similar syntax could be used to indicate optional parameters.
```
foo({first:String
     middle:String=“”
     last:String}
     cat(“Hello “ first middle last))
```

Union and intersection types are available through type aliases:
```
defAlias(Suit
         Clubs|Diamonds|Hearts|Spades)

defAlias(FaceCard
         Jack|Queen|King|Ace)

defAlias(QueenOfSpades
         Queen & Spades)

defAlias(Num Int)

;; A Rank could be either a face card or a Num (int):
defType(Rank
        FaceCard|Num)

;; A card is a combination of a suit and a rank:
defType(Card
        { suit:Suit
          rank:Rank
          toString=cat(suit “ “ rank) ;; zero-argument function
)
```

Function within the current namespace:
```
let{printCard:Fn<Card,String> =      ; variable name and optional type
     λ( (card:Card)                  ; function definition.  Arguments form a record
        "$card.suit$ $card.rank$") } ; body
```
Here we extend a type:
```
defType(WithBack
        showBack=λ“*****”)

defAlias(CardWithBack
         Card & WithBack)
```

A poker chip:
```
defAlias(Color
         Red|Green|Blue|Yellow)

defType(Chip
        {color:Color value:Int})

defAlias(PlayItem
         Chip|CardWithBack)
```

Now when you use a PlayItem, you have to destructure it:
```
printCard(item:PlayItem) = 
    case(item (Chip chip println(cat(“Chip: “ chip.color)))
              (CardWithBack card println(cat(“Card: “ card.printCard)))))
```

##Additional Ideas
From Josh Bloch: https://youtu.be/EduWekviwRg

 - Maybe auto-promote variables from a Record to a Map with Assoc, and let the type-system handle that.
 - Type system must handle nulls (Nil).  Any time you start a function by checking basic attributes of arguments is a missed opportunity for the type system to help you out.
 - Make it easier to write immutable, private code.
 - Friend classes/packages?  So you can share certain internals only with certain other classes/packages?

#License

Copyright 2015 Glen K. Peterson

Licensed under the Apache License, Version 2.0 (the "License"); you may not use this file except in compliance with the License. You may obtain a copy of the License at

http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software distributed under the License is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the License for the specific language governing permissions and limitations under the License.

#Reserved syntax
```
. field access
, whitespace
{} map/dictionary (also stores key order from creation)
() tuple/record
[] vector/list
#{} set
<> parameterized type
: introduces a type
@ introduces an annotation
```

## Keywords
```
default The default case for match and cond statements
match for type matching (ML calls this "pattern matching" but Java uses that to mean Regex)
Nil for null or false
t for true
```
