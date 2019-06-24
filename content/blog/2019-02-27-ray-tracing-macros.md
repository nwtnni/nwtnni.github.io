+++
title = "Ray Tracing: Macros"
date = 2019-02-27
description = "Implementing vector operations with Rust macros."
+++

Last semester, I learned about ray tracers in our [computer graphics course][1].
Inspired by the instructor-provided Java framework and other open-source
ray tracers like [pt][2] and [pbrt][3], I spent the past week working through
Peter Shirley's very approachable [Ray Tracing in One Weekend][4], which walks 
you through the implementation of a basic C++ ray tracer. Here I describe some
interesting lessons from translating to Rust, and [my ray tracer can be found here][5].

One of the first tasks for any graphics software is to either find or write
a linear algebra library: vector and matrix computations are everywhere.
The most mature option is probably [nalegbra][6], but since we only need the
Vec3\<f32\> type, I figured I'd follow Shirley and implement it myself.

## Background

Rust traits are like interfaces, describing common behavior across
different data types. Of particular note are the traits defined in
std::ops, like std::ops::Mul, which integrate directly with Rust's
operator syntax. For example, here's the definition of the Add trait:

```rust
pub trait Add<RHS = Self> {
    type Output;
    fn add(self, rhs: RHS) -> Self::Output;
}
```

Normally, you would have to use a method call:

```rust
let five = 2.add(3);
```

But because of syntactic sugar, this is equivalent to:

```rust
let five = 2 + 3;
```

Rust doesn't allow you to overload as many operators as C++, or
to define your own like OCaml, but this is plenty for now. For
more reading, you can also check out the book section on [advanced traits][7],
which actually also uses the Add trait as a case study.

## Motivation

While method chaining is nice, we would like to concisely do things
like add two vectors together, or multiply a vector by a scalar, and
this requires operator overloading. For example:

```rust
let a = v1.add(v2)
          .sub(v3)
          .mul(5.0);

let b = (v1 + v2 - v3) / 5.0;
```

However, implementing all of the traits by hand is boilerplate-heavy
and error-prone, because we end up duplicating a lot of logic. Case
in point:

```rust
impl Add<Vec3> for Vec3 {
    type Output = Vec3;
    fn add(self, rhs: Vec3) -> Self::Output {
        Vec3([
            self.x() + rhs.x(),
            self.y() + rhs.y(),
            self.z() + rhs.z()
        ])
    }
}

impl Add<&Vec3> for Vec3 {
    type Output = Vec3;
    fn add(self, rhs: &Vec3) -> Self::Output {
        Vec3([
            self.x() + rhs.x(),
            self.y() + rhs.y(),
            self.z() + rhs.z()
        ])
    }
}

impl Add<Vec3> for &Vec3 {
    type Output = Vec3;
    fn add(self, rhs: Vec3) -> Self::Output {
        Vec3([
            self.x() + rhs.x(),
            self.y() + rhs.y(),
            self.z() + rhs.z()
        ])
    }
}

impl Add<&Vec3> for &Vec3 {
    type Output = Vec3;
    fn add(self, rhs: Vec3) -> Self::Output {
        Vec3([
            self.x() + rhs.x(),
            self.y() + rhs.y(),
            self.z() + rhs.z()
        ])
    }
}
```

Without all four implementations, we can't operate on Vec3's and &Vec3's identically.
See [this playground link][8] for a demonstration: we would have to reference or dereference
everything appropriately before using an operator, which is pretty annoying for a simple Copy
type like Vec3. Here, good means that the line works, whereas bad means we get a
compile-time type error.

```rust
use std::ops::Add;

#[derive(Copy, Clone, Debug)]
struct Wrapper(usize);

impl Add<Wrapper> for &Wrapper {
    type Output = Wrapper;
    fn add(self, rhs: Wrapper) -> Self::Output {
        Wrapper(self.0 + rhs.0)
    }
}

fn main() {

    // Good    
    let a = Wrapper(0) + Wrapper(1);
    
    // Good
    let b = Wrapper(0).add(Wrapper(1));
    
    // Good: automatic coercion of self for method call
    let c = (&Wrapper(0)).add(Wrapper(1));
    
    // Bad
    let d = Wrapper(0).add(&Wrapper(1));
    
    // Bad
    let e = Wrapper(0) + &Wrapper(1);
    
    // Bad
    let f = &Wrapper(0) + Wrapper(1);
    
    // Bad
    let g = &Wrapper(0) + &Wrapper(1);
    
}
```

This is definitely one area where being explicit about references and
values creates more headache than it's worth, so I'd rather just implement the trait for
all common combinations.

Note that you don't have the same issue for self and method calls: if a method takes self
by reference, and you call the method from a value, it works just fine. But operators aren't
as smart, it seems.

## Macros

Function-like macros are a powerful tool for generating boilerplate. They expand at compile-time
into valid Rust code, and aren't just limited to expressions: you can generate just about
anything, from trait implementations to full modules.

Here, we're interested in generating trait implementations for Vec3.
The overall template is the same; there are only a few parameters
(denoted below by the underscore) that change between each implementation.

```rust
impl _<_> for _ {
    type Output = Vec3;
    fn _(self, rhs: _) -> Self::Output {
        Vec3::from(match (self.0, rhs) { _ => _ })
    }
}
```

We can take those wildcards and lift them into macro parameters:

```rust
macro_rules! op {
    (
        $op:ident,
        $fn:ident,
        $lhs:ty,
        $rhs:ty,
        $input:pat => $output:expr
    ) => {
        impl $op<$rhs> for $lhs {
            type Output = Vec3;
            fn $fn(self, rhs: $rhs) -> Self::Output {
                Vec3::from(match (self.0, rhs) {
                    $input => $output
                })
            }
        }
    }
}
```

And now we can implement addition with a single macro invocation:

```rust
op!(
    Add, add,
    Vec3, Vec3,
    ([x1, y1, z1], Vec3([x2, y2, z2])) =>
    [x1 + x2, y1 + y2, z1 + z2]
);
```

But why stop there? Our macro is actually general enough to handle all of the other operations as well:

```rust
op!(
    Sub, sub,
    Vec3, Vec3,
    ([x1, y1, z1], Vec3([x2, y2, z2])) =>
    [x1 - x2, y1 - y2, z1 - z2]
);
op!(
    Mul, mul,
    Vec3, Vec3,
    ([x1, y1, z1], Vec3([x2, y2, z2])) =>
    [x1 * x2, y1 * y2, z1 * z2]
);
op!(
    Div, div,
    Vec3, Vec3,
    ([x1, y1, z1], Vec3([x2, y2, z2])) =>
    [x1 / x2, y1 / y2, z1 / z2]
);
op!(
    Mul, mul,
    Vec3, f32,
    ([x, y, z], t) =>
    [x * t, y * t, z * t]
);
op!(
    Div, div,
    Vec3, f32,
    ([x, y, z], t) =>
    [x / t, y / t, z / t]
);
```

Our macro allowed us to focus on the interesting logic--the actual component-wise operations--instead
of copy-pasting a lot of boilerplate. But you'll notice that we haven't fixed the problem we mentioned
earlier, where you can't add a Vec3 to a &Vec3. So what's the fix?

```rust
macro_rules! impl_all {
    (
        $op:ident,
        $fn:ident,
        $lhs:ty,
        $rhs:ty,
        $input:pat => $output:expr
    ) => {
        op!($op, $fn, $lhs, $rhs, $input => $output);
        op!($op, $fn, $lhs, &$rhs, $input => $output);
        op!($op, $fn, &$lhs, $rhs, $input => $output);
        op!($op, $fn, &$lhs, &$rhs, $input => $output);
    }
}
```

We can wrap another macro around impl\_op that implements the operation for all combinations
of reference and value types! This gives us more flexible binary operators, but at the cost of
more code being generated, which can cause larger binaries and longer compile times.

## Conclusion

Using macros can greatly reduce the amount of boilerplate you have to write by 
hand, and Rust's metaprogramming capabilities open up a lot of interesting
possibilities for code generation. Procedural macros are even more powerful, with
perhaps the best known example being [serde\_derive][9].

[1]: http://www.cs.cornell.edu/courses/cs4620/2018fa/
[2]: https://github.com/fogleman/pt
[3]: https://www.pbrt.org/
[4]: https://github.com/petershirley/raytracinginoneweekend
[5]: https://github.com/nwtnni/photon
[6]: https://www.nalgebra.org/
[7]: https://doc.rust-lang.org/book/ch19-03-advanced-traits.html
[8]: https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=325d1dd6ca296832500bd2ad864707da
[9]: https://serde.rs/derive.html
