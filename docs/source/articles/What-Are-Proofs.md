## What are proofs?

Since proofs are, for most devs, the most unusual aspect of Formality, I've put this section on the docs as a simple explanation on what they are useful for. I think the best way to describe it is by using the analogy of "specifications as types". Let me elaborate. There is a full spectrum of type systems, right? JS and Python are untyped: the input of a function can be anything, nothing is guaranteed. In C, Java, Solidity, you can be more precise: *"this function accepts an `int` and returns an `int`"*. That "specifies" what the function does to an extent, but is very limited. Formality is at the top of the ladder: its types are so precise that you can **specify a complete algorithm at the type language**! Let me give you an example:

```javascript
import Base@0

Spec : Type
  {a : Bool} -> [b : Bool ~ Not(a == b)]
```

This code specifies *"a function that receives a Bool `a`, and returns a bool `b`,* ***such that*** `a != b`*"*. Can you see how there is only one `Bool -> Bool` function that satisfies that specification? And the cool thing is that the compiler can verify if a function satisfies it mechanically, without room for error. So, this works:

```javascript
// A function that negates a boolean
negate : {case a : Bool} -> [b : Bool ~ Not(a == b)]
| true  => [false ~ true_not_false] // if a is true, return false, and prove that `a != false`
| false => [true  ~ false_not_true] // if a is false, return true, and prove that `a != true"

// Proves that the "negate" function satisfies our `Spec`
main : Spec
  negate
```

But if you change any of the returned bools, it won't work anymore. It is literally impossible to make anything other than a boolean negation pass! In other words, a term of type `Spec` **proves** he specification represented by it. That's what theorem proving is; it is nothing but a fancy way to say *"type-checking in a language that has very precise types"*. And Formality types can be arbitrarily precise. For example, this `Spec`:

```javascript
Spec : Type
  {len : Ind} ->                       // Given a length `len`
  ! { ~A : Type                        // And a type `A`
    , idx : Fin(len)                   // And an index, `idx`, up to `len`
    , vec : Vec(A, len)                // And a vector, `vec`, with `len` elements of type `A`
    } -> [x : A ~ At(A,x,len,idx,vec)] // Returns the element `x` that is at index `idx` of that `vec`
```

specifies an array accessor that can't have an out-of-bounds error, and that can't return the wrong element! If OpenSSL proved it, we wouldn't have Heartbleed. And the cool thing is that those proofs happen statically, they have zero runtime costs!

In the context of smart-contracts, we could have specs like "this contract's balance satisfies certain invariant", completely preventing things like TheDAO being drained. Of course, proofs can be huge and ugly, but that's ok, developers are paid to work hard and write good software. The point is to have a small list of simple specifications that users can read and be confident the smart-contract behaves as desired, without having to trust its developers.

From [this Reddit thread](https://www.reddit.com/r/ethereum/comments/d45vpq/im_hyper_bullish_on_ethereum/f08waxj/?context=1).
