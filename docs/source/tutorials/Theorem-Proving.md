## Theorem Proving

**TODO**: review and update with correct error messages (after the `Hole` update). Also talk about `log(x)`.

### Simple proofs

Let's prove a theorem about the boolean `not`:

```haskell
import Base@0

main : {b : Bool} -> not(not(b)) == b
  ?
```

Evaluate it using `fm -t <file>/main` and the type checker complains:

```shell
[ERROR]
Hole found.
- With goal... not(not(b)) == b
- Inside of... {b} => ?
- With context:
- b : Bool

{b : Bool} -> not(not(b)) == b
```

Let's pattern-match on `b`.

```haskell
import Base@0

main : {b : Bool} -> not(not(b)) == b
  case/Bool b
  | true  => ?
  | false => ?
  : not(not(b)) == b
```

Not helpful:

```shell
[ERROR]
Hole found.
- With goal... not(not(b)) == b
- Inside of... (%b)(~{self} => not(not(b)) == b, ?)
- With context:
- b : Bool

[ERROR]
Hole found.
- With goal... not(not(b)) == b
- Inside of... (%b)(~{self} => not(not(b)) == b, ?, ?)
- With context:
- b : Bool

{b : Bool} -> not(not(b)) == b
```

Using `self` on the motive:

```haskell
import Base@0

main : {b : Bool} -> not(not(b)) == b
  case/Bool b
  | true  => ?
  | false => ?
  : not(not(self)) == self
```

Progress, `b`s is specialized `true` on the expected type of the `true` branch:

```shell
[ERROR]
Hole found.
- With goal... not(not(true)) == true
- Inside of... (%b)(~{self} => not(not(self)) == self, ?)
- With context:
- b : Bool

[ERROR]
Hole found.
- With goal... not(not(false)) == false
- Inside of... (%b)(~{self} => not(not(self)) == self, ?, ?)
- With context:
- b : Bool

{b : Bool} -> not(not(b)) == b
```

If we reduce both sides, we get the same expression: `{true, false} => true`. In this case, we can use `refl`:

```haskell
import Base@0

main : {b : Bool} -> not(not(b)) == b
  case/Bool b
  | true  => refl(~true)
  | false => ?
  : not(not(self)) == self
```

Progress, compiler now complains about the `false` branch:

```shell
[ERROR]
Hole found.
- With goal... not(not(false)) == false
- Inside of... (%b)(~{self} => not(not(self)) == self, refl(~true), ?)
- With context:
- b : Bool

{b : Bool} -> not(not(b)) == b
```

We can do the same:

```haskell
import Base@0

main : {b : Bool} -> not(not(b)) == b
  case/Bool b
  | true  => refl(~true)
  | false => refl(~false)
  : not(not(self)) == self
```

No type error. Our proof is complete! Note that, if we used `case`'d args, Formality would fill the `self` on the motive for us. The proof becomes just:

```haskell
import Base@0

main : {case b : Bool} -> not(not(b)) == b
| true  => refl(~true)
| false => refl(~false)
```

### Inductive proofs

Let's prove a similar theorem, but for negation on arbitrary-length bit-strings instead of plain booleans:

```haskell
import Base@0

// Imported from Base@0
// T Bits
// | b0 {pred : Bits}
// | b1 {pred : Bits}
// | be

#bnot*n : !{case halt bits : Bits} -> Bits
| b0 => b1(bnot(bits.pred))
| b1 => b0(bnot(bits.pred))
| be => be
```

Start with the theorem we want to prove:

```haskell
#main*n : !{bits : Bits} -> <bnot(n)>(<bnot(n)>(bits)) == bits
  ?
  halt: ?
```


Remember that `halt:` is mandatory on recursive definition to provide the base-case. TODO: explain why the `-#`s on the type. 

The type checker complains:

```haskell
Type mismatch.
- Found type... Hole
- Instead of... bnot(step(n), bnot(step(n), bits)) == bits
- When checking ?
- On expression {bits} => ?
- With the following context:
- n    : Ind
- n    : Ind
- main : {bits : Bits} -> bnot(n, bnot(n, bits)) == bits
- bits : Bits
```

Notice that:

1. It asks for `P(step(n))` instead of `P(n)`.

2. We have, on context, `main`, which gives us `P(n)`.

That's because the body of a recursive function is actually the step case of inductive proof, so all we need to do is, assuming `P(n)`, prove `P(step(n))`!

Let's match against `bits`, using `self` on the motive:

```haskell
#main*n : !{bits : Bits} -> <bnot(n)>(<bnot(n)>(bits)) == bits
  case/Bits bits
  | b0 => ?
  | b1 => ?
  | be => ?
  : <bnot(step(n))>(<bnot(step(n))>(self)) == self
  halt: ?
```

Now the complaint becomes:

```shell
Type mismatch.
- Found type... Hole
- Instead of... bnot(step(n), bnot(step(n), b0(pred))) == b0(pred)
- When checking ?
- On expression {bits} => ?
- With the following context:
- n    : Ind
- n    : Ind
- main : {bits : Bits} -> bnot(n, bnot(n, bits)) == bits
- bits : Bits
- pred : Bits
```

This is better because now it expects `b0(pred)` instead of just `bits` . This allows the left-side of the equation to be reduced to:

```haskell
b0(bnot(n, bnot(n, pred))) == b0(pred)
```

This is perfect because we can use the inductive hypothesis to get this same equation, without the `b0`s. As in, we need to go...

```haskell
from :    bnot(n, bnot(n, bs))  == b0(bs)
to   : b0(bnot(n, bnot(n, bs))) ==    bs
```

All we need is to add `b0` on both sides. We can do it with `cong`, from the base libraries (`Base@0`):

```haskell
#main*n : !{bits : Bits} -> <bnot(n)>(<bnot(n)>(bits)) == bits
  case/Bits bits
  | b0 => cong(~Bits, ~Bits, ~(<bnot(n)>)((<bnot(n)>)(pred)), ~pred, ~b0, ~main(pred))
  | b1 => ?
  | be => ?
  : <bnot(step(n))>(<bnot(step(n))>(self)) == self
  halt: ?
```

TODO: should we have a built-in syntax to simplify `cong`?

Now the checker complains about the `b1` case:

```shell
Type mismatch.
- Found type... Hole
- Instead of... bnot(step(n), bnot(step(n), b1(pred))) == b1(pred)
- When checking ?
- On expression {pred} => ?
- With the following context:
- n    : Ind
- n    : Ind
- main : {bits : Bits} -> bnot(n, bnot(n, bits)) == bits
- bits : Bits
- pred : Bits
```

We can easily complete this proof now:

```haskell
#main*n : !{bits : Bits} -> <bnot(n)>(<bnot(n)>(bits)) == bits
  case/Bits bits
  | b0 => cong(~Bits, ~Bits, ~<bnot(n)>(<bnot(n)>(pred)), ~pred, ~b0, ~main(pred))
  | b1 => cong(~Bits, ~Bits, ~<bnot(n)>(<bnot(n)>(pred)), ~pred, ~b1, ~main(pred))
  | be => refl(~be)
  : <bnot(step(n))>(<bnot(step(n))>(self)) == self
  halt: refl(~bits)
```

As usual, it could be simplified with case'd arguments:

```haskell
#main*n : !{case bits : Bits} -> <bnot(n)>(<bnot(n)>(bits)) == bits
| b0 => cong(~Bits, ~Bits, ~<bnot(n)>(<bnot(n)>(bits.pred)), ~bits.pred, ~b0, ~main(bits.pred))
| b1 => cong(~Bits, ~Bits, ~<bnot(n)>(<bnot(n)>(bits.pred)), ~bits.pred, ~b1, ~main(bits.pred))
| be => refl(~be)
halt: refl(~bits)
```

Note that the big difference here, with relation to Agda/Coq proofs, is that, in their cases, since recursive functions are defined by structural recursion, inductive proofs are also defined by recursion on the structure. For example, if we wanted to prove this theorem in Agda, we'd just match the bit-string, prove the base case by reflexivity, and prove the recursive case by calling `main` recursively on `pred`.

In Formality, it **looks** like the proof is the same, but there is a subtle, yet important, difference: under the hoods, we're not actually recursing on the `Bits` structure. Instead, we're folding over `n : Ind`, a datatype capturing the inductive hypothesis on natural numbers. As such, in order to prove that `bnot(n, bnot(n, bits)) == bits` hold for any `n`, we first must prove that it is true for `n == 0`, i.e., when `bits` has a maximum recursion depth of `0` (i.e., is "out-of-gas"), which is true by reflexivity since the function returns `bits` itself. We then prove that assuming this is true for a maximum recursion depth of `n`, then it is also true for `bits(step(n))`. This is the `step` case, which coincides with Agda's proof.

Proving that kind of inductive theorem on Formality is a little more verbose than in Agda, since you have to wrap the whole proof inside `Ind`, and always tell the compiler what to do when the function "runs out of gas". In exchange, since termination is guaranteed by EAL, there is no "structural recursion" checker, so you're allowed to be more flexible in your recursive definitions.
