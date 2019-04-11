# Introducing the Copy Constructor – Talk outline

NOTE: Most of the technical aspects are copy pasted from the DIP as I wrote the DIP having a potential presentation in mind

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
them.

### Considering qualified postblits

I. `const`/`immutable` postblits. A solution for const and immutable postblits would be to type check them as normal constructors, where the first assignment of a member is considered an initialization and subsequent assignments count as modifications. This is problematic because after the blitting phase, the destination object is no longer in its initial state and subsequent assignments to its fields will be regarded as modifications, making it impossible to construct nontrivial immutable/const objects in the postblit. In addition, it is possible for multiple postblits to modify the same field. Consider:

```d
struct A
{
    immutable int a;
    this(this)
    {
        this.a += 2;  // modifying immutable, an error or OK?
    }
}

struct B
{
    A a;
    this(this)
    {
        this.a.a += 2;  // modifying immutable, an error or OK?
    }
}

void main()
{
    B b = B(A(7));
    B c = b;
}
````
When B c = b; is encountered, the following actions are taken:

1. b's fields are blitted to c
2. A's postblit is called
3. B's postblit is called

After step 1, the object `c` has the exact contents as `b`, but it is neither initialized (the postblits still need to run) nor uninitialized (the field `B.a` does not have its initial value). From a type checking perspective this is a problem because the assignment inside `A`'s postblit is breaking immutability. This makes it impossible to postblit objects that have `immutable`/`const` fields. To alleviate this problem one could consider that after the blitting phase the object is in a raw state, therefore uninitialized; this way, the first assignment of `B.a.a` is considered an initialization. However, after this step the field `B.a.a` is considered initialized, therefore how is the assignment inside `B`'s postblit supposed to be type checked? Is it a violation of immutability, or should it be legal? Indeed, it is breaking immutability because it is changing an immutable value. However as this is part of initialization (recall that `c` is initialized only after all the postblits are ran) it should be legal, thus weakening the immutability concept and creating a different strategy from that implemented by normal constructors.

II. `shared` postblits. Shared postblits cannot guarantee atomicity while blitting the fields because that part is done automatically and it does not involve any synchronization techniques. The following example demonstrates the problem:

```D
shared struct A
{
    long a, b, c;
    this(this) { ... }
}

A a = A(1, 2, 3);

void fun()
{
    A b = a;
    /* do other work */
}
```

Let's consider the above code is run in a multithreaded environment
When `A b = a;` is executed, the following actions are carried:

* `a`'s fields are copied to `b` bitwise ("blitted")
* `this(this)` is called

In the blitting phase, no synchronization mechanism is employed, which means
that while copying is in progress, another thread may modify `a`'s data, resulting
in the corruption of `b`. There four possible approaches to fixing this issue:

1. Make `shared` objects larger than two words noncopyable. This solution cannot be
taken into account because it imposes a major arbitrary limitation: almost all
`struct`s would become noncopyable.

2. Allow incorrect copying and expect that the user will
do the necessary synchronization from the outside. Example:

```D
shared struct A
{
    Mutex m;
    long a, b, c;
    this(this) { ... }
}

A a = A(1, 2, 3);

void fun()
{
    A b;
    a.m.acquire();
    b = a;
    a.m.release();
    /* do other work */
}
```

Although this solution solves the synchronization problem, it does so in a manner
that requires unencapsulated attention at each copy site. A distinct performance problem is
created by releasing the mutex after the postblit is
run, which increases the amount of work in the critical section. The release should at best occur immediately following
the blitting phase (the first line of the postblit) because the copy is
thread-local, but this results in non-scoped locking: the mutex is released
in a different scope than that in which it was acquired. Also, the mutex is
automatically (and wrongfully) copied.

3. Introduce a preblit function that would be called before blitting the fields.
The purpose of the preblit is to offer the possibility of preparing data
before the blitting phase; acquiring the mutex on the `struct` to
copy is one of the operations that the preblit would be responsible for. Later
on, the mutex would be released in the postblit. This approach has the benefit of
minimizing the critical section in a manner that offers encapsulation, but
suffers the disadvantage of adding even more complexity by introducing a new
concept which requires type checking disparate sections of code (need to type check
across preblit, blit, and postblit).

4. Use an implicit global mutex to synchronize the blitting of fields. This approach
has the advantage that the compiler will do all the work and synchronize all blitting
phases (even if the threads don't actually touch each other's data) at a cost to
performance. Python implements a global interpreter lock which is criticized for its
nonscalable high contention; there are ongoing discussions of removing it from the Python
implementation [5].

As discussed above, the postblit is difficult to type check without unreasonable restrictions and
cannot be synchronized without undue costs. After documenting all these issues, I couldn't find a
reasonable solution to all of there problems and neither did Andrei or Walter, so we came to the
conclusion that our best option would to start on a clean slate and implement the copy constructor.

Pros:

1. it is a known concept, C++ uses it and everyone is happy
2. we have a robust implementation of constructors (that works perfectly with qualifiers) and we
can leverage that in our implementation.

Cons:

We lose the sweet automatic blit offered by the postblit. But this is not actually a con, since
D's biggest strength is metaprogramming we can trivially substitute that via a generic function:

```d
foreach (i, ref field; src.tupleof)
    this.tupleof[i] = field;
```

This brings us to the copy constructor:

### Syntax

Inside a `struct` definition, a declaration is a copy constructor declaration if it is a constructor declaration that takes the first parameter as a nondefaulted reference to the same type as the `struct` being defined. Additional parameters may follow if and only if all have default values. Declaring a copy constructor in this manner has the advantage that no parser
modifications are required, thus leaving the language grammar unchanged. Example:

```d
import std.stdio;

struct A
{
    this(ref return scope A rhs) { writeln("x"); }                     // copy constructor
    this(ref return scope A rhs, int b = 7) immutable { writeln(b); }  // copy constructor with default parameter
}

void main()
{
    A a;
    A b = a;            // calls copy constructor implicitly - prints "x"
    A c = A(b);         // calls constructor explicitly
    immutable A d = a;  // calls copy constructor implicittly - prints 7
}
```

The copy constructor may also be called explicitly (as shown above in the line introducing `c`) because it is also a constructor within the preexisting language semantics.

The argument to the copy constructor is passed by reference in order to avoid infinite recursion.

### Semantics

This section discusses all the aspects of the semantic analysis and interaction between the copy
constructor and other language features.

#### Copy constructor and postblit cohabitation

In order to ensure a smooth transition from postblit to copy constructor, this DIP proposes the
following strategy: if a `struct` defines a postblit (either user-defined or generated), all
copy constructor definitions will be ignored for that particular `struct` and the postblit
will be preferred. Existing code bases that do not use the postblit may start using the
copy constructor, whereas codebases that currently rely on the postblit may
start writing new code using the copy constructor and remove the postblit
from their code. This DIP recommends deprecation of postblit but does not prescribe a specific deprecation schedule.

A semantically close transition path from postblit to copy constructor may be achieved as follows:

```d
// Existing code
struct A
{
    this(this) { ... }
    ...
}
// Replacement code
struct A
{
    this(ref return scope inout A rhs) inout { ... }
}
```

#### Copy constructor usage

A call to the copy constructor is implicitly inserted by the compiler whenever a `struct` variable is initialized
as a copy of another variable of the same unqualified type:

1. When a variable is explicitly initialized:

```d
struct A
{
    this(ref return scope A rhs) {}
}

void main()
{
    A a;
    A b = a; // copy constructor gets called
    b = a;   // assignment, not initialization
}
```

2. When a parameter is passed by value to a function:

```d
struct A
{
    this(ref return scope A another) {}
}

void fun(A a) {}

void main()
{
    A a;
    fun(a);    // copy constructor gets called
}
```

3. When a parameter is returned by value from a function and NRVO cannot be performed:

```d
struct A
{
    this(ref return scope A another)
    {
        writeln("cpCtor");
    }
}

A fun()
{
    A a;
    return a;
}

A a;
A gun()
{
    return a;
}

void main()
{
    A a = fun();      // NRVO - no copy constructor call
    A b = gun();      // NRVO cannot be performed, copy constructor is called
}
```

The parameter of the copy constructor is passed by reference, so initializations will be
lowered to copy constructor calls only if the source is an lvalue. Although this can be
worked around by declaring temporary lvalues which can be forwarded to the copy constructor,
binding rvalues to lvalues is beyond the scope of this DIP.

Note that when a function returns a `struct` instance that defines a copy constructor and NRVO cannot be
performed, the copy constructor is called at the callee site before returning. If NRVO
may be performed, then the copy is elided:

```d
struct A
{
    this(ref return scope A rhs) {}
}

A a;
A fun()
{
    return a;      // lowered to return tmp.copyCtor(a)
    // return A(); // rvalue, no copyCtor called
}

void main()
{
    A b = fun();    // b is constructed in place with the return value of fun
}
```

Copy constructor overloads can be explicitly disabled:

```d
struct A
{
    @disable this(ref return scope A rhs);
    this(ref return scope immutable A rhs) {}
}

void main()
{
    A b;
    A a = b;     // error: disabled copy construction

    immutable A ia;
    A c = ia;  // ok

}
```

In order to disable copy construction, all copy constructor overloads must be disabled.
In the above example, only copies from mutable to mutable are disabled; the overload for
immutable to mutable copies is still callable.

#### Overloading

The copy constructor can be overloaded with different qualifiers applied to the parameter
(copying from a qualified source) or to the copy constructor itself (copying to a qualified
destination):

```d
struct A
{
    this(ref return scope A another) {}                        // 1 - mutable source, mutable destination
    this(ref return scope immutable A another) {}              // 2 - immutable source, mutable destination
    this(ref return scope A another) immutable {}              // 3 - mutable source, immutable destination
    this(ref return scope immutable A another) immutable {}    // 4 - immutable source, immutable destination
}

void main()
{
    A a;
    immutable A ia;

    A a2 = a;      // calls 1
    A a3 = ia;     // calls 2
    immutable A a4 = a;     // calls 3
    immutable A a5 = ia;    // calls 4
}
```
The proposed model enables the user to define the copy from an object of any qualified type
to an object of any qualified type: any combination of two among mutable, `const`, `immutable`, `shared`,
`const shared`.

The `inout` qualifier may be applied to the copy constructor parameter in order to specify that mutable, `const`, or `immutable` types are
treated the same:

```d
struct A
{
    this(ref return scope inout A rhs) immutable {}
}

void main()
{
    A r1;
    const(A) r2;
    immutable(A) r3;

    // All call the same copy constructor because `inout` acts like a wildcard
    immutable(A) a = r1;
    immutable(A) b = r2;
    immutable(A) c = r3;
}
```

In the case of partial matching, existing overloading and implicit conversion
rules apply to the argument.

#### Copy constructor call vs. blitting

When a copy constructor is not defined for a `struct`, initializations are handled
by copying the contents from the memory location of the right-hand side expression
to the memory location of the left-hand side expression (i.e. "blitting"). Example:

```d
struct A
{
    int[] a;
}

void main()
{
    A a = A([7]);
    A b = a;                 // mempcy(&b, &a)
    immutable A c = A([12]);
    immutable A d = c;       // memcpy(&d, &c)
}
```

When a copy constructor is defined for a `struct`, all implicit blitting is disabled for
that `struct`. Example:

```d
struct A
{
    int[] a;
    this(ref return scope A rhs) {}
}

void fun(immutable A) {}

void main()
{
    immutable A a;
    fun(a);          // error: copy constructor cannot be called with types (immutable) immutable
}
```

#### Generating copy constructors

A copy constructor is generated implicitly by the compiler for a `struct S` if all of the following conditions are met:

1. `S` does not explicitly declare any copy constructors;
2. `S` defines at least one direct member that has a copy constructor, and that member is not overlapped (by means of `union`) with any other member.

If the restrictions above are met, the following copy constructor is generated:

```d
this(ref return scope inout(S) src) inout
{
    foreach (i, ref inout field; src.tupleof)
        this.tupleof[i] = field;
}
```

## Breaking Changes and Deprecations

1. The parameter of the copy constructor is passed by a mutable reference to the
source object. This means that a call to the copy constructor may legally modify
the source object:

```d
struct A
{
    int[] a;
    this(ref return scope A another)
    {
        another.a[2] = 3;
    }
}

void main()
{
    A a, b;
    a = b;    // b.a[2] is modified
}
```

This is surprising and potentially error-prone behavior because changing the source of a copy is not customary and may surprise the user of a type. (For that reason, C++ coding standards adopt the convention of taking the source by means of reference to `const`; copy constructors that use non-`const` right-hand side are allowed but discouraged.) In D, `const` and `immutable` are more restrictive than in C++, so forcing `const` on the copy constructor's right-hand side would make simple copying task unduly difficult. Consider:

```d
class Window
{
    ...
}
struct Widget
{
    private Window display;
    ...
    this(ref return scope const Widget rhs)
    {
        display = rhs.display; // Error! Cannot initialize a Window from a const(Window)
    }
}
```

Such sharing of resources across objects is a common occurrence, which would be impeded by forcing `const` on the right-hand side of a copy. (An inferior workaround would be to selectively cast `const` away inside the copy constructor, which is obviously undesirable.) For that reason this DIP proposes allowing mutable copy sources.

2. This copy constructor changes the semantics of constructors that receive a parameter by reference of type
`typeof(this)`. The consequence is that existing constructors might be called implicitly in some
situations:

```d
struct C
{
    this(ref return scope C another)    // normal constructor before DIP, copy constructor after
    {
        import std.stdio : writeln;
        writeln("Yo");
    }
}

void fun(C c) {}

void main()
{
    C c;
    fun(c);
}
```

This will print `"Yo"` subsequent to the implementation of this DIP, whereas under the current semantics it prints nothing. A case can be
made that a constructor with the above definition could not be correctly used as anything else than a copy
constructor, in which case this DIP actually fixes the code.

After all this information, I will talk a bit about future work:

-> Smarter generation of copy constructors.
-> Generating opAssign from copy constructors.

I don't know yet how long the presentation will take as I don't have the slides now, but depending on how much
time I will have left I can go into a lot of details on the above mentioned topics.

After that, I will wrap it up somehow.
