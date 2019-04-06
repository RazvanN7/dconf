# Introducing the Copy Constructor – Talk outline

Last year, in this period, I was trying to fix some issues regarding the postblit and came to the conclusion
that it has some insurmountable problems, which I will later discuss. The alternative to the postblit is the
copy constructor, which I designed and implemented with the help of Andrei and Walter. Given that the concept
is very important for the language and the fact that there still are a lot of people that love the postblit idea,
I thought that this is a great opportunity to get everybody on the same page and set the grounds for further
discussions on improvements regarding the copy constructor.

## Introduction

Now, what is a copy constructor and why is it useful? I'm sure that most of the people present here are familiar with
this concept, but for those that aren't: it is a C++ specific notion as other languages do not offer builtin
support for it and the whole point of it is to offer user control over what happens when a copy of an object is being made.
In a world without copy constructors, whenever an object is initialized from another one of the same type, the compiler
does a bit-copy between the source and the destination. That's what happens in all other languages, except C++. This
mechanism has the drawback that it always creates shallow copies; for example, if an object contains a pointer to an
array, copying that object will result in 2 different objects having a view over the same the array; while there are
scenarios in which this behavior is the desired one, there are also situations in which deep copy is required. This is
where the copy constructor steps in: it is an explicitly user defined constructor that is automatically called whenever
a copy is being made. The advantage is that you do not have to call it manually so it makes certain programming patterns
(like reference counting) more expressive and easier to implement.

If the copy constructor is so cool, how come other languages do not implement it? Well, most of the popular languages
are object oriented programming languages that work with reference type classes, so most of the time you are passing
references, not making copies; if you want a copy then you can create your own function/method and call it.

Now let's see what is D's take on this matter. In D we have the notion of `postblit`. This is the glorified version of a
copy constructor, where the compiler does the copying of the fields for you and all you have to do is adjust the fields
that need to be updated. This is an awesome feature as it makes all the boilerplate code that simply does
`this.field1 = rhs.field2` dissapear. For example, if you had a reference counted `struct` that has 90 fields + an `uint`
that tracks the number of references for an instance, with the traditional copy constructor, you would have to write
at least 91 lines (90 for field initialization + 1 the refcount update); with the `postblit` you need to write a single
line (the refcount update), as all the fields are copied automatically. This is awesome, everybody loves this new invention...
.... BUT! Even though the postblit is very seductive, it has its hidden claws which bite you when you least expect it.
To better understand the nature of the problem, we need to take a closer look at the postblit and its functionality.

## Overview of `this(this)`

### Interaction with qualifiers

The postblit would have been fine in a setting that did not require any qualifiers. Moreover, the spec explicitly states
that the postblit cannot be qualififed or meaningfully overloaded, however the following is not rejected:

```d
struct A { this(this) const {} }
struct B { this(this) immutable {} }
struct C { this(this) shared {} }
```

The semantics of the postblit in the presence of qualifiers is not defined, and experimentation reveals that the behavior is a
patchwork of happenstance semantics: `const` postblits are not able to modify any fields in the destination, immutable postblits
never get called (resulting in compilation errors), shared postblits cannot guarantee atomicity while blitting the fields. In
this situation we are faced with a dilemma: either we disallow the qualification of postblits or we try to correctly support
them. We cannot 
