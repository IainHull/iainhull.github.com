---
layout: post
title: "Idris - First Steps"
subtitle: "Writing my first dependently typed function in Idris."
tags: [idris, noob, types, dependently-typed]
---

I hate debugging code, so I try really hard to produce correct code in the first place. As a result one of my main interests is automatic program verification. Initially for me this meant unit testing, then design by contract, later exploring how the type system can improve correctness. I was luck enough to attend an introduction to [Idris](http://www.idris-lang.org/) with [Edwin Brady](https://twitter.com/edwinbrady) at [Functional Kats Conf](http://functionalkats.com/) in September. Idris is a [dependently typed](https://en.wikipedia.org/wiki/Dependent_type) language, this means that types are first class citizens, they can interact with and depend on values. This enables you to to improve (and even prove) correctness on a whole new level. For example a Vect in Idris is a generic list whose length is part of its type, therefore a Vect of three chars has a different type to a list of four chars. This means if a function parameter requires a Vect of a certain size the compiler guarantees the function is only called with a Vect of that size. No more checking the size at runtime and no more illegal argument exceptions. With one parameter type the compiler prevents a whole class of bugs.

I was really impressed by Idris, and have spent the past few weeks trying to learn it. I started with the book, [Type-Driven Development with Idris](https://www.manning.com/books/type-driven-development-with-idris) available on Manning's early access program. Next I needed an exercise to practice, so chose the [Bank OCR Kata](http://codingdojo.org/cgi-bin/index.pl?KataBankOCR). This involves parsing lines of text and recognising numbers. The requirements state: _"Each entry is 4 lines long, and each line has 27 characters"_, this is a perfect application of dependent types. Therefore the first function I wrote was `maybeVect`, this takes a size and a list and returns a `Maybe` containing a `Vect` of the specified size. If the list is the correct size it returns a `Just` and if the size is wrong it returns `Nothing`.

Here is the code

```idris
maybeVect : (n: Nat) -> List a -> Maybe (Vect n a)
maybeVect Z [] = Just []
maybeVect n [] = Nothing
maybeVect Z xs = Nothing
maybeVect n (x :: xs) = let S(k) = n in
                             map (\ xs' => x :: xs') (maybeVect xs k)
```
The function has four cases:
* the required size is zero and list is empty return an empty `Vect` wrapped in a `Just`.
* the required size is not zero and the list is empty, the size is incorrect so return `Nothing`.
* the required size is zero and the list is not empty, again the size is incorrect so return `Nothing`.
* finally the size is not zero and the list is not empty, here we split the list into its head and tail and use recursion to process the tail, then if the tail is valid we add the head back on.

The key in this final case is letting the compiler track the size of the list in the recursive call. This is done by assigning the number previous to `n` into the value `k`, then using this in the recursive call to `maybeVect`. Now when the head is added to the `Vect` the compiler knows the size of the returned `Vect` will be `n`. The `map` function adds the head to the tail, inside the `Maybe` if it's a `Just` or does nothing if its `Nothing`.

Lets test it out in the repl:

```idris
maybeVect> maybeVect 0 (the (List Char) [])
Just [] : Maybe (Vect 0 Char)

maybeVect> maybeVect 0 ['a']
Nothing : Maybe (Vect 0 Char)

maybeVect> maybeVect 1 (the (List Char) [])
Nothing : Maybe (Vect 1 Char)

maybeVect> maybeVect 1 ['a']
Just ['a'] : Maybe (Vect 1 Char)

maybeVect> maybeVect 2 ['a', 'b']
Just ['a', 'b'] : Maybe (Vect 2 Char)

maybeVect> maybeVect 3 ['a', 'b']
Nothing : Maybe (Vect 3 Char)
```

This is my first Idris function and of course it wasn't plain sailing. Type-driven development, means specifying your types first and letting the compiler guide your implementation. Or as I found, specify your types and let compile error thwart my implementation, so below is how I reached the function above.

The first three cases were easy however I struggled with the last one. I knew I could use the `map` function, but struggled with compile errors so I wrote my own function to prepend an item on to the head of a `Vect` inside a `Maybe`.

```idris
maybePrepend : a -> Maybe (Vect n a) -> Maybe (Vect (S n) a)
maybePrepend x Nothing = Nothing
maybePrepend x (Just xs) = Just (x :: xs)
```

This function although not required simplified my remaining type errors. My first attempt was to use `S(n)` in the pattern matching the size

```idris
maybeVect S(k) (x :: xs) = maybePrepend x (maybeVect xs k)
```

I am not sure why this didn't work (I still have lots to learn), but here is the error.

```
When checking left hand side of maybeVect2:
Type mismatch between
        Maybe (Vect n a) (Type of maybeVect n _)
and
        argTy -> retTy (Is maybeVect n
                                    _ applied to too many arguments?)

Specifically:
        Type mismatch between
                Maybe
        and
                \uv => argTy -> uv
```

Then I read some Idris code and part of the implementation of `Vect` and came up with the following

```idris
maybeVect { n = S(k) } (x :: xs) =  maybePrepend x (maybeVect xs k)
```

Which results in this error

```
MaybeVect.idr:15:12:When checking left hand side of maybeVect:
n is not an implicit argument of maybeVect.maybeVect
Holes: maybeVect.maybeVect
```

Ok so that was a wild guess and once I saw the error I realised I was going in the wrong direction. I looked for other examples of `S(n)` in the `Vect` source code and found the `let` syntax. This resulted in the following line.

```idris
maybeVect n (x :: xs) = let S(k) = n in
                            maybePrepend x (maybeVect k xs)
```

This worked and from here applying `map` was easy.  I am still not sure why my first attempt did not work, but I learned the importance of reducing the moving parts when debugging type-errors in dependently typed languages. Once I defined `maybePrepend` the compiler did guide me in the right direction, I just have to get better at reading that guide.

If you find any of this interesting I encourage you to try Idris yourself. I will continue with my Bank OCR example and let you know how I get on.

The code is available in [this gist](https://gist.github.com/IainHull/91ff8a28826e57316674).
