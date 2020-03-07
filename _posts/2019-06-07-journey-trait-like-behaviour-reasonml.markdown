---
layout: post
title:  "Journey to Trait-like behaviour in ReasonML/OCaml"
date:   2019-06-07 14:11:05 -0300
categories: [reasonml, ocaml]
tags: [functor, rust]
---
## Motivation
I have been using ReasonML as a replacement for JavaScript on the front-end development for my startup for quite some time now. But with a powerful type-system, it makes me miss Rust type-system features.
Right now, I don't know enough ReasonML/OCaml to truly let loose the way I mentally model the problems at hand, which have been spoiled by my time with Rust.

This note is a documentation of my journey in trying to bridge the gap between my Rust knowledge and ReasonML/OCaml knowledge.

## Where I want to get
In Rust, you can have things such as:
```rust
#[derive(Deserialize)]
struct MyStruct {
  foo: String
}

fn decode_from_json<T: Deserialize>(json: &str) -> T {
  // ...
}

let my_struct: MyStruct = decode_from_json("{\"foo\":\"bar\"}");
```

I want to create a function that given a return type, correctly decodes the json to that type.

## Prep work
Since I was trying to reproduce the `serde::Deserialize` trait, I started looking into what the community had already built and gathered around.  
From this I found two libraries worth mentioning: `bs-json` and `ppx_decco`, with `ppx_decco` being my personal favorite.  

`bs-json` is a great library for manually parsing json, seems like a great building block for a more higher level library such as `ppx_decco` (I haven't checked if `bs-json` is used by `ppx_decco`)

`ppx_decco` description is as follow: "Bucklescript PPX which generates JSON serializers and deserializers for user-defined types."  
This library was quite to my liking as it generates the serializers and deserializers for you, has great expressibility to map the JSON to your type, and the generated functions have a simple API.  
But why doesn't it work for me? (It does, but I have a itch I want to scratch, so let's pretend it doesn't)  
Because the API I'm aiming for is not something like `my_type_decode(Js.Json.t) => my_type` it's `decode_json(Js.Json.t) => 'a`

## Implementation
After looking into libraries from the ecosystem and not finding anything quite like the itch I want to scratch, I started playing around with some implementations that could get me close to where I want, and start researching from there.

#### Module Function (Functors)
This was the first thing that came to mind, as it was mentioned in the ReasonML
documentation. Although in the doc, it says that you can't pass Modules to functions, which made it quite hard to arrive at the second implementation that is exactly that.

Functors works by receiving a Module and returning another Module.

So how could we use Functors to implement something Trait-like?

A Functor is really good for making generic code relative to modules, such as a generic implementation of `HashMap` that could go be like:
```reasonml
module type Eq = {
  type t;
  let eq: t => t => bool;
};

module type Hash = {
  type t;
  let hash: t => string;
};

module type EqHash = {
  include Eq;
  include Hash with type t := t;
};

module MakeHash = (M: EqHash) => {
  /* ... implementation details */
};

module Int = {
  type t = int;
  
  let eq = (a, b) => a == b;
  let hash = a => string_of_int(a);
};

module IntHashMap = MakeHash(Int);
```

But this seems to not quite be what we want. The Functor looks more like `impl Trait for Type` instead of `trait Trait`. Altough those `module type Eq` and `module type Hash` looks a lot like what we want.


#### Modules as first-class arguments in functions

As the API I'm aiming for is a function, let's try to do it with a function! To achieve it, we have to create a module signature:

```reasonml
module type Decodable = {
  type t;
  let decode: string => t;
};
```

Then, as we code our modules, if one should be Decodable, we only need to adhere to the Decodable type signature, such as:

```reasonml
module Unit = {
  type t = unit;
  let decode = _ => ();

  let to_string = _ => "()";
};
```

This is enough to write a function such as `decode_json((module Unit), "anything")`

```reasonml
let decode_json = (type a, decode, json) => {
  module Decode = (val decode: Decodable with type t = a);
  Decode.decode(json)
};

/* Example */
let _ = decode_json((module Unit), "anything");
```

This function looks a lot like the one from Rust, except that it receives the Module as argument. Could we make it not receive the Module as argument, instead having OCaml infering it? Turns out we can't (Or at least, I couldn't after researching). We have to pass the Module as argument.  

This boils down to how in ReasonML/OCaml you can have different implementations of a "type-class" for the same type, whereas in Rust you can't, so Rust can infer the type that implements the "type-class" (In this case, Trait).

## Conclusion
After all we couldn't get to our `decode_json(Js.Json.t) => 'a`, but we got quite close there with `decode_json((module Decodable with type t = 'a), Js.Json.t) => 'a`

I'm quite happy with the result, as it got me more used to the flexibility of the type-system, helped me better understand how to model my problem to fit with this type-system and made me realize that the difference between Rust's type-system and OCaml's type-system is in the trade-offs, and both are equally powerful.

Thanks a lot to Pedro Castilho, who helped me throughout this journey! Without him, it would have taken me a lot more time to learn all this. Most of what you have seen here has been me trying to do something, bouncing the idea at him and him kindly explaining to me how we could arrive at the final result.


Edit: Thanks to Yawar Amin for replying with a great resource about modular
implicits, which would enable the compiler to infer the correct module to pass
as argument to the function. [First-Class Modules and Modular Implicits in
OCaml](https://tycon.github.io/modular-implicits.html)

Edit 2: Thanks to Cem for pointing out another great (de)serialization library:
[Automatic, well-typed JSON serialization in Reason/OCaml with Milk](https://jaredforsyth.com/posts/announcing-milk/)
