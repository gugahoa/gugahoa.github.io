---
layout: post
title:  "Defunctionalization"
date:   2019-06-17 14:11:05 -0300
categories: [reasonml]
tags: [defunctionalization]
---
Defunctionalizing is a technique to use high-order
functions in a language that does not support it.
For example, in a language that supports high-order
functions, we could write `fold` as:

```reasonml
let rec fold = (fn, initial, list) =>
  switch (list) {
  | [] => initial
  | [hd, ...tail] => fn(hd, fold(fn, initial, tail))
  };
let sum = list => fold((el, acc) => el + acc, 0, list);
sum([1, 2, 3]);
/* int = 6 */
let add = (n, list) => fold((el, acc) => [el + n, ...acc], [], list);
add(2, [1, 2, 3]);
/* list(int) = [3, 4, 5] */
```

But in a language that do not, we have to abstract
somehow the `fn` argument, we could do that by
creating a type called `arrow(_, _)` which is a GADT
that holds information for our `sum` and `add` helper functions.

```reasonml
type arrow(_, _) =
  | FnPlus: arrow((int, int), int)
  | FnPlusCons(int): arrow((int, list(int)), list(int));
```

Here `FnPlus` is abstracting `(el, acc) => el + acc`, and
`FnPlusCons(int)` is abstracting `(el, acc) => [el + n, ...acc]`.
The first type argument for `FnPlus` is `(int, int)` which
will hold our function arguments, the second type argument is
a `int` that represents the result of the function. The same structure
applies to `FnPlusCons(int)`, but with an additional `int` which will hold
what was the `n` in `let add = (n, list) => fold((el, acc) => [el + n, ...acc], [], list);`

Now we can write an `apply` helper function as:

```reasonml
let apply: type a b. (arrow(a, b), a) => b =
  (fn, value) => {
    switch (fn) {
    | FnPlus => {
      let (el, acc) = value;
      el + acc;
    }
    | FnPlusCons(n) => {
      let (el, acc) = value;
      [el + n, ...acc];
    }
    }
  };
```

And use it in our new `fold` function:

```reasonml
let rec fold: type a b. (arrow((a, b), b), b, list(a)) => b =
  (fn, inital, list) => 
    switch (list) {
    | [] => u
    | [hd, ...tail] => apply(fn, (hd, fold(fn, initial, tail)))
    }
```

Then it's done! We have successfully avoided the use of high order functions.

```reasonml
let sum = list => fold(FnPlus, 0, list);
let add = (n, list) => fold(FnPlusCons(n), [], list);

sum([1, 2, 3]);
/* int = 6 */
add(2, [1, 2, 3]);
/* list(int) = [3, 4, 5] */
```

This concept is a building block towards understanding this [paper](https://www.cl.cam.ac.uk/~jdy22/papers/lightweight-higher-kinded-polymorphism.pdf).

More on this topic can be found here:  
[Defunctionalization at Work](https://www.brics.dk/RS/01/23/BRICS-RS-01-23.pdf)  
[Defunctionalization](https://en.wikipedia.org/wiki/Defunctionalization)  
[Definitional interpreters for higher-orderprogramming languages](https://surface.syr.edu/cgi/viewcontent.cgi?article=1012&context=lcsmith_other)
