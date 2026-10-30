---
title: "How monads can be used outside of functional programming languages"
date: 2025-02-05
---

Monads are a famously difficult concept in functional programming, though once they 'click', creating and using them can be rather trivial. Probably the most famous monad is the *nullable* type, also known as the `Maybe` monad in Haskell or the `Option` type in Rust, where a value can be represented as 'something' or 'nothing'.

```hs
-- The Maybe monadic type declaration in Haskell
data Maybe a = None | Just a
```

Monads play a very important role in functional programming languages, often described generally as:
> a way to structure computations as a sequence of steps, where each step not only produces a value but also some extra information about the computation, such as a potential failure, non-determinism, or side effect.

-- (from [Wikipedia](https://en.wikipedia.org/wiki/Monad_(functional_programming)))

In terms of non-functional languages, we *already* have systems to deal with potential failures (exceptions), non-determinism, and side effects, while also having a clear sequence of computational steps - being the definition of [*imperative languages*](https://en.wikipedia.org/wiki/Imperative_programming).

## The Exception monad

In this section, we'll look at an imperative language that does have exceptions (C#) and an imperative language that does not have exceptions (Rust) and compare the control flow between them to see if we can factor out anything in common.

Rust does not have exceptions. It is one of the most loved features of the language and a personal favourite of mine. Instead, it relies on the `Result<T, E>` type whose variants are either a success result or a failure result, i.e. `Ok` and `Err`.

For example, we will look at a classic divide-by-zero problem and model is using results.

*For brevity, we'll assume we're using `anyhow::Result` to avoid explicitly typing out the error type*

```rust
fn safe_div(p: f32, q: f32) -> Result<f32> {
    match (p, q) {
        (_, 0) => Err(Error::msg("Attempted to divide by zero.")),
        _ => Ok(p / q)
    }
}
```

Then, in a `main` function, we'll attempt several divisions to try and get an answer.

```rust
fn main() -> Result<f32> {
    let p: f32 = 10;
    let q: f32 = 2;

    let res = safe_div(p, q);

    // we have to check here that the result succeeded before continuing.
    if res.is_err() {
        return res;
    }
    let ans = res.unwrap();     // can safely unwrap here

    let res = safe_div(p, ans);
    if res.is_err() {
        return res;
    }
    let ans = res.unwrap();
    // ...

    // if at this point, no failures have occurred.
    println!("The answer is: {}", ans);
    Ok (ans)
}
```

As you can see, it can quickly get verbose constantly checking if a result succeeded or failed. This is why the `?` operator was created.

### The ? operator

I'll call this operator the 'short-circuit operator', as it will cause a function to return early if it encounters a failure result, or it will unwrap if the expression was successful. We can rewrite the above main function to be much simpler as a result:

```rust
fn main() -> Result<f32> {
    let p: f32 = 10;
    let q: f32 = 2;

    // if the expression failed, the failure is returned.
    let ans = safe_div(p, q)?;
    let ans = safe_div(p, ans)?;
    // ...

    // if at this point, no failures have occurred.
    println!("The answer is: {}", ans);
    Ok (ans)
}
```

As you can see, the short-circuit operator reduces a lot of boilerplate across our programs.

### C# - exceptions

In C# (and many other imperative languages), exceptions are primarily used to handle errors. They are a control-flow technique that can similarly short-circuit functions and yield control back up the call-stack until the error is handled explicitly.

The same example as above:

```c#
float Divide(float p, float q) {
    if (q == 0.0) {
        throw new Exception("Attempted to divide by zero.");
    }
    return p / q;
}
```

And the caller code:

```c#
float Main() {
    var p = 10.0;
    var q = 2.0;

    float ans;
    ans = Divide(p, q);
    ans = Divide(p, ans);
    // ...

    // if at this point, no failures have occurred.
    Console.WriteLine($"The answer is {ans}");
    return ans;
}
```

Notice that this looks almost identical to the control-flow of Rust with the short-circuit operator. The only issue here is that it's not clear from this code that divide can fail.

In C# and other languages that don't have checked exceptions (see not Java), exceptions are not included in function contracts, meaning there's no way to know that the function can fail just by looking at it. Of course, well documented code will tell you if the code can throw exceptions, but not everyone documents their code well.

However, this isn't meaning to say that exceptions are bad or inferior to results. Exceptions typically don't need to be considered in your application code, since they should be exceptional. There is an unfortunate anti-pattern where exceptions are being used as general control-flow (i.e. glorified goto statements), and this is the main argument against checked exceptions.

```java
float divide(float p, float q) throws DivideByZeroException {
    if (q == 0.0f) {
        throw new DivideByZeroException();
    }
    return p / q;
}
```

an example of Java's checked exceptions

Now, this post isn't for or against the use of exceptions in code, so we'll get back on track.


## Something or nothing - the Maybe monad

This will be a brief section, as it was covered earlier and it's the simplest example.

In many manual memory-management languages such as C/C++, `nullptr` was used to denote whether a pointer pointed to some meaningful memory or if it points to nothing. This practice was carried on to automatic memory-managed languages such as Java and C# so that any object could possibly be `null`.

The inclusion and use of null is very simple and can cause fewer headaches in the short-term, but poses significant headaches in large and critical codebases where null dereferencing can lead to errors, vulnerabilities, and system crashes.

In recent years, C# has introduced null-safety, a very popular pattern that guarantees certain references will not be null at compile-time. To be pedantic, C# is not entirely null-safe, since it's an optional feature and can be overridden, but for new projects that opt-in to null-safety, it's one of the best things you can do for your own sanity.

It does this through the `?` decorator.

The ? decorator after a type T will tell the compiler that this type is actually `Nullable<T>`. As an example:

```c#
void Main(string[] args) {
    string? arg = args.FirstOrDefault();
    // arg is either some string or nothing
}
```

Other languages that have no nulls often rely on ADTs (Abstract Data Types) to represent something or nothing.

```rust
enum Option<T> {
    None,
    Some(T)
}
```

These are functionally equivalent, that is to say that the Nullable<T> type in C# is a monad. It even has its own short-circuiting features via the ?. and the ?? operators, e.g.

```c#
var name = person?.Name ?? "default name"
```

## Conclusion

To avoid dragging this post out for too long, I'll cut the post off here. There are many other examples of monads in imperative languages, but ultimately, they just aren't that important. They're there, but no one really cares because in a non-functional context, there are so many other control-flow patterns available that obsolete the desire for categorising things as monads.

One final example that I'm particularly interested in is the *Future monad* which models asynchronous actions. Thankfully, this has already been covered in much greater detail by a much more qualified person: [Bartosz Milewski - C++17 - I See a Monad in Your Future!](https://bartoszmilewski.com/2014/02/26/c17-i-see-a-monad-in-your-future/). Make sure to check it out!