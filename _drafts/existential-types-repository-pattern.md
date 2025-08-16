---
layout: post
title: "A repository for entities with existential types for keys"
tags: csharp functional-programming
---

# Introduction

I would like to implement a repository pattern in C# that can tell the
user of the repository what data type they need to provide to the `Find` method
to retrieve an item by its ID without making that data type into a generic
argument in the repository interface.

{% highlight csharp %}

interface IRepository<T> where T : BaseEntity
{
    T Find(T.TKey id);
}

{% endhighlight %}

This made up syntax, `T.TKey`, means: The data type "alias" defined in `T` called
`TKey`. Inside the `T` entity, it could look like `TKey = System.Guid`, for
example. This, in my opinion, makes a lot of sense as we know for  what `T`
we've got an `IRepository`, so we should be able to also tell what the data type
used to uniquely identify each entity is, and reference it accordingly in our
entity finder method.

This functionality already exists in other programming languages, like Rust.
Rust allows the definition of "associated types" in traits. Traits are (without
looking too closely at all) equivalent to C# interfaces. 

In Rust, we can define a `BaseEntity` trait that requires implementors to define
what their ID type is going to be as well as its getters and setters, like so:

{% highlight rust %}

trait BaseEntity {
    type TKey;
    fn get_id(&self) -> &Self::TKey;
    fn set_id(&mut self, key: Self::TKey);
}

{% endhighlight %}

When a struct wants to implement `BaseEntity`, perhaps because our repository
requires types that can be stored to implement `BaseEntity`, its `impl` block
must define the concrete type aliased to `TKey`.

{% highlight rust %}

struct Repository<T: BaseEntity> {
    store: Vec<T>
}

impl<T: BaseEntity> Repository<T> {
    pub fn new() -> Self {
        Self {
            store: vec![]
        }
    }

    pub fn find(&self, key: &T::TKey) -> Option<&T> {
        // Placeholder
        None
    }
}

impl BaseEntity for User {
    type TKey = u32;

    fn get_id(&self) -> &Self::TKey {
        &self.id
    }

    fn set_id(&mut self, key: Self::TKey) {
        self.id = key;
    }
}

// Now we can instantiate a repository of user like so:
let user_repository = Repository::<User>::new();

let user = user_repository.find(&10); // OK
let user = user_repository.find("10"); // Error: mismatched types, expected
                                       // &u32, found &str.

{% endhighlight %}

With this setup, Rust's compiler knows that when we call `find` on
`user_repository`, the parameter passed to the method must be of type `&u32`.
This is the functionality I want to implement in C#.

What we're talking about here is more formally known as "existential types",
which can be roughly understood as "for any type that implements `BaseEntity`,
there exists a concrete `TKey` type, _fixed for that implementor_, that
satisfies the trait's requirements".

Fortunately for me, I'm not the only person who would like to have this
possibility available as you can see in this C# language proposal:
[8711][csharplang-8711]. Even more fortunately, the proposal includes how the
syntactic sugar of existential types would translate to what the CLR sees, which
can guide our implementation of this repository pattern.

# Creating BaseEntity in C#

When designing our `BaseEntity`, we want to make it generic. We want each base
entity to have the possibility of saying `User` has a primary key of type `guid`
or `Comment` has a primary key type of `int`, so our `BaseEntity` might look
something like this at first:

{% highlight csharp %}

abstract class BaseEntity<TKey>
{
    public TKey? Id { get; set; }
}

{% endhighlight %}

However, if you recall from the beginning of the article, our `IRepository`
interface established a constraint that all the types a repository may be
instantiated with _must_ inherit from a non-generic `BaseEntity`.

When thinking about this, for me it felt vaguely intuitive that at this point
`BaseEntity<T>` must implement a non-generic interface so our repository can
make that its generic type constraint.

{% highlight csharp %}

abstract class BaseEntity<TKey> : ISomeInterface;

interface IRepository<T> where T : ISomeInterface;

{% endhighlight %}

Now we can define a `User` class, for example, that inherits from
`BaseEntity<Guid>` and which meets the criteria to be stored in our repository.
Still, we don't yet know how we might be able to define, much less implement, a
`Find` method in our repository that uses the ID type that `User` has defined,
since from within `IRepository` we don't know what `T` `User` has defined for
`BaseEntity<T>`.

Thanks to Brian Berns' lovely posts on functional programming in C#, I have a
rough idea of where to go next from here. Check out his articles
[here][brian-berns].

# What is ISomeInterface?

Since we said that all of our base entities implement a common interface which
enables us to treat them all equally regardless of the actual `TKey` or concrete
class that inherits from them, we might wonder: what's the point? What kind of
shared functionality do all base entities need to allow us to look them up
inside our repository's `Find` method?

`ISomeInterface` is a wrapper on all of our base entities that declares a method
which receives an object that will perform an operation on said base entity.
This allows us to create an object which contains a value and performs an
operation on any base entity checking if the value it contains matches the
entity's ID. This is what's called a visitor pattern which is defined as
follows:

> "Represent an operation to be performed on the elements of an object
> structure. Visitor lets you define a new operation without changing the
> classes of the elements on which it operates."[^1]

The visitor pattern requires that objects able to be visited implement an accept
method which receives an instance of a visitor object. So the interface for the
objects we want to visit might be defined like so:

{% highlight csharp %}

interface IEntityWrapper
{
    void Accept(IEntityVisitor visitor);
}

{% endhighlight %}

In the above example, you can see the second part to the visitor pattern. The
visitor interface, which must be implemented by all classes that can visit
objects. This is what our definition might look like:

{% highlight csharp %}

interface IEntityVisitor
{
    void Visit<TKey>(BaseEntity<TKey> entity);
}

{% endhighlight %}

Since our `BaseEntity<TKey>` implements `IEntityWrapper`, let's define the body
of `Accept` inside our base class:

{% highlight csharp %}

abstract class BaseEntity<TKey> : IEntityWrapper
{
    public TKey? Id { get; set; }

    public void Accept(IEntityVisitor visitor)
    {
        visitor.Visit(this);
    }
}

{% endhighlight %}

# A visitor that produces values

Our previous implementation of the visitor pattern, while aligned with the
definition, doesn't meet our needs yet, as we can't determine whether the
visited object is the one we're looking for because the return type of the
visitor's `Visit` method is void. The visitor needs to give the caller
information on whether the visited object is valid according to the internal
logic of the visitor's operation.

To solve this, let's define an interface for a visitor which returns a value
after visiting an entity:

{% highlight csharp %}

interface IEntityVisitor<TRet>
{
    TRet Visit<TKey>(BaseEntity<TKey> entity);
}

{% endhighlight %}

And our new visitor type also needs to be accepted by the entity wrapper
interface:

{% highlight csharp %}

interface IEntityWrapper
{
    // Previous methods...

    TRet Accept<TRet>(IEntityVisitor<TRet> visitor);
}

abstract class BaseEntity<TKey> : IEntityWrapper
{
    // Previous methods...
    
    public TRet Accept<TRet>(IEntityVisitor<TRet> visitor)
    {
        return visitor.Visit(this);
    }
}

{% endhighlight %}

With this, a class implementing `IEntityVisitor<TRet>` can visit any
`BaseEntity<TKey>`, regardless of its `TKey` type, perform operations on it, and
return a `TRet`. And in turn, visited entities which implement `IEntityWrapper`
will return the value produced by the visitor after accepting said visitor.

Now that we've got a visitor interface that can produce values, we need to
consider how to implement a visitor which contains a value that will be compared
with the visited entity's `Id` property and return whether it matches.

# A match by key visitor

As we previously mentioned, a visitor needs to be constructed with a fixed
value. For this we need to abstract away the type of the value the visitor
contains, similarly to how we abstract away the key type of our base entity.

In line with our `IEntityWrapper` interface, a solution for this is to wrap our
test key value in an interface, `IKeyWrapper`.

{% highlight csharp %}

interface IKeyWrapper
{
    bool TryUnwrap<TWanted>([NotNullWhen(true)] out TWanted? key);
}

{% endhighlight %}

This interface requires implementors to try to convert the underlying value to
the requested `TWanted` type, put it in the `out` parameter and return whether
the conversion succeeded. Its implementation might look like this:

{% highlight csharp %}

class KeyWrapper<TKey> : IKeyWrapper
{
    private readonly TKey _key;
    
    public KeyWrapper(TKey key)
    {
        _key = key;
    }

    public bool TryUnwrap<TWanted>([NotNullWhen(true)] out TWanted? key)
    {
        if (_key is TWanted k)
        {
            key = k;
            return true;
        }

        key = default;
        return false;
    }
}

{% endhighlight %}

Now we can create a visitor class that implements `IEntityVisitor<bool>` and
whose constructor receives an `IKeyWrapper` instance which it can use when
visiting entities.

{% highlight csharp %}

class MatchByKeyVisitor : IEntityVisitor<bool>
{
    private readonly IKeyWrapper _key;

    public MatchByKeyVisitor(IKeyWrapper key)
    {
        _key = key;
    }

    public bool Visit<TKey>(BaseEntity<TKey> entity)
    {
        return _key.TryUnwrap(out TKey? key) &&
               EqualityComparer<TKey>.Default.Equals(entity.Id, key);
    }
}

{% endhighlight %}

I think it's important to note how this technique works. Since our visit method
is generic over the entity's key type, `TKey`, we can ask the wrapped key to
try and unwrap to the same type. If it fails because our wrapped key and the
entity's key are of different types, we return false early. But if it succeeds
we can use the equality comparer for `TKey` with the `key` output and the
entity's ID.

[csharplang-8711]: https://github.com/dotnet/csharplang/discussions/8711
[brian-berns]: https://dev.to/shimmer/functional-programming-in-c-3h6e

[^1]: Erich Gamma et al., _Design Patterns: Elements of Reusable Object-Oriented
    Software_, Addison Wesley, 1994, p. 331


