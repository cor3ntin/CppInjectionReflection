# Code generation in C++

## Reflection

This is not an area I have looked into much, yet, and the current reflection proposals are realy great.
But reflection must be fleshed out first because the injection/meta class proposal depend on it.

I will come back to the keywords issue later on :)


## Constexpr blocks

Constexpr blocks declare compound statements at the namespace, or class level.


They shall be evaluated at compile time, and shall have no side effect beside declaring injection statements.
They exist solely for the purpose of defining a context in which we can create and use injection statements.
Constexptr blocks within a class are evaluated at the end of the class declaration.
Within a class, unless otherwise specified, injected declarations have the same access specifier than the constexpr block they were injected from.




A constexpr block can contain `break` statements to stop its evaluation.
A constexpr block has neither a name, a value nor a type.


## Injection

Injection is the ability to create code fragments and insert them in a pre-existing context.
Most of the injection mecanism is described in [P0172](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2017/p0712r0.pdf) 
However, I propose the following syntaxes:


 - `-> [struct|class|namespace]  { /*...*/  };`  Inject a code fragment in the current context ( the class or namespace the `constexpr` statement lives in ). `struct|class|namespace`  determines the type of the context a fragment can be injected in. In most cases, that type can be inferred and does not need to be specified.

 - `X  -> { /*...*/  };`  inject a fragment in X.  The nature of the fragment is deduced from X. X is either a type or a reflection. If X is a class, the fragment shall not modify the memory layout of X.


 - ` -> { };`  Inject the fragment in the current class/namespace. Direct code injection are not restricted in what they can inject : class members, virtual methods...

Fragments being reflections they can be assigned to a variable, in the following manner:

 - `X = -> class|struct|namespace {  }`
 - `X = -> {  }`  //expression or statement


Diverging from P0712r0, i suggest that injections and code fragment can only exist in a `constexpr` block.
`constexpr` blocks and injection are purposefully designed to be a bit ceremonious to avoid careless injection of anything anywhere.

### Static asserts

In code fragments, static_asserts are only triggered when the fragment is injected somewhere


### ODR

Injections are pretty powerful, but come with certains limitations
 * it is not possible to modify the memory layout of a fully declared class.
 * it is not possible to use injection to violate ODR at the translation unit level
 * it is not possible to overwrite or modify an existing name


## S-Macros

Syntactic macros (for lack of a better name) are a generalization of injection primitives to provide named code injection that can take parameters.
I think there are risks with functions having "code injection" as one of their side effect.
And while functions can take fragments as parameters or return fragments, they should not be allowed to execute injections.

This is a definition of a S-macro ( which can only happen from within a constexpr block. 
`macro_name->( args ) {  }`

They are then used like that `macro_name->(args)`

Note that a macro does not have a "return value". That because a macro doesn't return.

Macro names create an identifier in the parent context of the constexpr-block in which they are created.
They can, like function takes parameters, however the parameters are limited to:
   * Literal types
   * Reflections on : names, types, expressions and identifiers.

A macro expand to either a statement ( including declarations ), an identifier or an expression.
However, a `s-macro` is required to expand to the same type of construct on all its code path.

A S-Macro that expand to a statement can expand to a null statement, but if it expand to an expression or an identifier, it shall do that on all its code path.

A S-Macro however has no `return value` nor is it called.

When a `s-macro` is defined, the compiler ensure that it is not ill-formed, using the same logic that is used for templates.
`typename` can be used to resolve ambiguity.

When used in a cast expression, the compiler can assume that the expression passed as parameter will have a type that matches the type they are casted to.
For example, `X x = expr;`  expr can assumed to be return something of type X, unless the expression parameter is constrained

I suggest using concept to declare `s-macro` parameters


### Wandering off : Partial Template Specialization for concept


The Shorthand notation ( which I'm a big proponent of ), allows to write

`void f(const MyConcept & bar)`  instead of

```
template<MyConcept T>
void algo(T const&);
``` 


For the purpose of what will follow, I suggest that this syntax should be allowed to be partially specialized, such that
```
void f(const MyConcept<bool> & bar)
```

becomes a shorthand for

```
template<typename T>  requires MyConcept<T, bool>
void algo(T const&);

```

This has yet to be explored in much more details


### Ok, back to S-Macros



The concept and class to introduced are

 * `std::meta::expression` : A reflection on an expression that can be constrained on the type of its value
 * `std::meta::type        : A reflection on a type
 * `std::meta::identifier  : An identifier that may not refer to a name


A `s-macro` could also take user defined concepts as parameters. A long as the actual concrete parameter is a reflection or literal type. 


```
constexpr {
    bool debug = /*...*/;
    log->(std::meta::expression<const char*> c, std::meta::expression<>... args) {
            if(debug) {
                -> { printf(->c, (->args)...); };
            }
    }
}

void foo() {
    log("Hello %", "World");  //expand to printf("Hello World")  only and only if debug is true
}

```


### A note about parameters

`s-macro` parameter can not be cv-qualified, nor can they be reference or pointers.
They are not copied, there is no call stack involved, and they are not evaluated until the code in which the s-macro is expanded is evaluated
As the example above show, `s-macro`s answer the issue of lazy/conventional evaluation.


### Macros have a name

`-> log { }`  declares a name `log` in the context it is declared in.
That means that `s-macro` can't have the name of an already declared object, nor can the name of a macro can be used to declare another object.
For obvious reasons, they can not be forward declared and must be defined where they are declared. 


Furthermore, one can use `using`  directives to make copies of aliases.
`s-macro`s can be, like any other c++ construct, be exported and imported from modules.
The same lookup rules applies, for the name of a `s-macro`.


### Order of operations

When a `s-macro` is declared in a constexpr block, we check that the name does not conflict, then the body of the `s-macro` is parsed as if it was a template, the variables declared in the scoping constexpr block are captured
And a valid AST is generated and associated with the name of the `s-macro` which is then injected in the context ( class or namespace ).
If the parameters are constrained ( for example, an expression is constrained on its type ), it shall also be validated at that time.
If an expression is not constrained, we assume it can convert to everything and anything.

When a `s-macro` expansion is encountered

 1. Name lookup for the `s-macro`
 2. Each parameter is parsed,  un(qualified) names are implicitely converted to type reflection, expression or unknown identifier depending on the expectations of the macro.
 3. We check whether the `s-macro` can be expanded where it is called, namely, the parser may expect either an identifier, a statement or an expression (expressions can of course be converted to statements)
 4. We inject the parameter in a copy of the `s-macro` and evaluate that
 5. The S-Macros injection statements are collected and added to the ast in place of the s-macro-expansion-expression


We have therefore ensured that
 1. The `s-macro` is correctly formed
 2. The `s-macro` parameters are correctly formed 
 3. The `s-macro` is correctly used



### Overloading
It's not something that I have think about much but it's certainly something to consider


### What's up with the arrows

`->` is the only token used and required by this proposal. Several people suggested `->` for code injection purposes.
It's already a valid c++ token, which mean there is no need to add any new token or keyword, and `->` represents injection pretty well.

The use of `->` presented here can be distinguished easily for the existing `->`  usages. Namely, pointer-to-member operator, deduction guidelines and trailing return types.   

An alternative suggestion is `<-`. There are arguments for both. 
However, (even though it's not hard problem to solve)  `<-` is not a valid c++ token, but the sequence `<`, `-` can be encountered ( in comparison and template instantiation )


An astute reader would have notice that the arrow in a `macro_expansion->()` expression is completely unnecessary and it is there just to distinguish macros from function.
An even more astute reader will wonder why an annoying advocate defender of the `shorthand form` for concepts, is adding useless syntax.

The fact of the matter is that a function call and a macro expansion are 2 completely different things. The arrow helps the reader be aware that:
 * The transformation happens at compile time
 * The parameters may not be evaluated at all
 * the ast is being modified
 

For the sake of consistency and the above arguments, I would suggest `for->` as a construct to iterate over an iterable reflection.

  
  
### Hold-on, we need a bunch of tokens for reflection

```
namespace std {
    constexpr {
        ->reflex(std::reflectable t) { //about anything
            -> { t; };
        }
    }
}
``` 

Used as `auto meta_string = std::meta::reflex->(std::string);`

The above takes advantage of the fact that macro arguments are implicitly "converted" to their reflection.
The other keywords proposed by p0712r0  such as  `namespace`, `hasname`  `declname` can similarly be implemented as `s-macro` in the `std` namespace, wrapping intrinsics.


### What else can I do with `s-macros`
`s-macro`s can call other `s-macro`s, inject code that call other `s-macro`, and do all of that recursively.
Of course, a `s-macro` expand to code, recursive `s-macro`s expand to... more code !

On the top of my head,  you could use `s-macros` to implement 

 - The `expected` proposal as well as the `try()` construct. [P0779R0](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2017/p0779r0.pdf) was actually one of the inspiration and goals for this proposal.
 - assert and test macros
 - log facilities



## Defining the implementation of user-declared function at compile time. 

One classic thing code generators do is to implement the body of a user defined function.
That function may have arbitrary type and number of arguments.
The solution I propose for that is to use a parameter pack.

```
int foo(int i, double d);
```

can be later defined

```
auto reflex->(foo) (typename args....) {
    //Here we can implement foo
}
```

`args` is a parameter pack that allow to manipulate the function type without knowing their type or arity.
However, it's neither a copy, a cast, or a conversion, but merely a proxy that does not change the signature of the function.
The reflection is used here as to ensure we are defining the implementation of a particular method, and not an overload.
It also tells the compiler to ignore the cv-qualifiers and other attributes.


the following definitions are valid:

```
auto reflex->(foo) (typename args....) {}
auto reflex->(foo) (int i, double d, typename args....) {}
auto reflex->(foo) (typename arg1, typename arg2) {}
```

the following are not

```
auto reflex->(foo) (int i) {}  // parameter count mismatch
auto reflex->(foo) (long i, double d) {} // parameter type mismatch
auto reflex->(foo) (long & i, const double d) {} // parameter cv-qualifiers mismatch
```

Note that despite the use of a parameter pack, this is not a template function, and the function is only "instantiated" once. The arguments declaration transformations are merely a convenience for the developer,
the compiler is fully aware of every parameter time when the declaration is parsed.

In the same fashion the return type is known an immutable even if the definition specifies auto


## Attributes

So far we discussed about reflection and injection, so why talk about attributes, a features seemingly completely unrelated to the matter of hand.
The thing is, when you generate a meta program, you probably want to only apply some transformation to some members, so you need a way to filter them.
The best way to do that is to add a way to query attributes of reflected identifiers.

People will argue that attributes should not modify the semantic of a program, and they are right as things stands. However the only good ( practical) justification is that it would lead to non-standards behaviours.
However, if this proposal makes it to the standard, it will be... well, standard, and therefore all compiler will generate the same code for the same meta program.

People suggested that a new construct unrelated to attributes should be added for that purpose.
However, attributes are certainly sufficient as they can be namespaced, can have values, etc.... a lot of features that are currently not used !

Adding new construct is probably not what we want to do if our collective goal is to simplify the language.

So far I haven't a proposal for what the api could look like.



## typename()

For the convenience of s-macros, code-injection and meta-classes, I suggest that the construct `typename()` should be provided in classes ( and struct, union, enum ) to return the (unqualified?) name of the current class.
An alternative solution would be to allow `decltype(*this)` in static and constexpr context. But that would be weird semantically



## Meta classes

There are several ways to formalize meta classes.
Either the definition is copied to a new type and the transformation happens during the copy.
Or a meta-classe is simply a class that exposes one or several `constexpr` blocks that are automatically executed in the scope of the derived class once it's defined.
I'm not sure which is best.
The later avoid introducing new syntax / semantic.
