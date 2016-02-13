# Sum types

There are many ways to build new types in OCaml, and *sum types* is
one of the most important ones. While a type `t = int * string` means that
you need an integer _and_ a string to build values of type `t`, sum
types correspond to situations when you need only one _or_ the
other. A little too abstract? Let's start with an example.

## Logics

OCaml has a boolean type, and all usual operators are avaible:
```ocaml
let a = true;;
let b = false;;
not a;;
a && b;;
a || b;;
```

So far so good. Let's introduce a (modestly) richer logic, where a
fact could be true, false or "maybe true" (this is known as a
[three-valued logic](https://en.wikipedia.org/wiki/Three-valued_logic)). This
means that a variable can take three possible values, making booleans
inadequate as a representation. One possibility would then be to use
the integer type, but we can do a lot better:

```ocaml
type trilean = True | False | Maybe;;
```

After this type definition, `True`, `False` and `Maybe` become valid
identifiers to construct values of type `trilean` (and they are the
only ones to be so). For this reason they are called **constructors**
of the type `trilean`. Note that constructors must start with a
capital letter (while variables must start with a lower case).

To define an operation on this type, we typically use *pattern
matching*. For example we can extend the logical negation in the
following way:

```ocaml
let nott x = match x with
  | True -> False
  | False -> True
  | Maybe -> Maybe
;;
```

The `match` construct can be used to decompose any OCaml expression
and examine cases separately. Each pipe character (`|`) introduces a
new case, that is described by a _pattern_. A pattern is basically an
expression where some parts can be ommited, using a wildcard
(underscore character, `_`). For each case we need to provide the
value of the function, and in all cases, the result should have the
same type (which is the type of the whole `match` expression). Let's
have a look at a more complex example, with the definition of the
conjunction in our new logic:

```ocaml
let andt x y = match (x, y) with
  | (False, _) -> False
  | (_, False) -> False
  | (Maybe, _) -> Maybe
  | (_, Maybe) -> Maybe
  | (True, True) -> True
;;

andt True Maybe;;
```

Here we do pattern matching on a pair of values. This is a general
method: if you want to do case analysis on several values at once,
match on a tuple gathering them. There is a wildcard in the first
case, meaning that whatever is the value of `y`, if `x` is `False`
then return `False`. Note that cases are considered from top to
bottom, and the first case with a compatible pattern will be used
(even if subsequent patterns are compatible too). For instance, the
value `(False, Maybe)` will be handled by the first case and not the
fourth one. Pattern-matching is a very effective way to express a
function (try to rewrite `andt` with `if` expressions only, if you
need to be convinced!). But it's not only more readable: you can have
the compiler check additional things for free. Let's see that on the
disjunction:

```ocaml
let wrong_ort x y = match (x, y) with
  | (True, _) -> True
  | (_, True) -> False
  | (Maybe, _) -> Maybe
  | (_, Maybe) -> Maybe
;;
```

Here the compiler was able to spot an error in this function (were
you?). Not only that, it was able to show me an example of case I
forgot to handle (here `(False, False)`). The compiler issued a
warning, nevertheless accepting the definition. If we evaluate the
function on that case, that leads to an error:

```ocaml
wrong_ort False False;;
```

The compiler can also detect redundant cases:

```ocaml
let wrong_ort2 x y = match (x, y) with
  | (True, _) -> True
  | (_, True) -> False
  | (Maybe, _) -> Maybe
  | (_, Maybe) -> Maybe
  | (False, False) -> False
  | (True, Maybe) -> True
;;
```

There are limitations, of course (and I'm not going to show them) but
pattern matching on sum types is a major feature in ML languages, and
it helps a lot in practice. To leverage that technique, it is crucial
to define types that describe our data very closely, and avoid
representations were some values are conventionally not
permited. Typically here, we would have no help from the compiler if
we had defined our trileans as integers.

## DNA sequences

Pattern matching can be used on arbitrarily complex OCaml
expressions. To see that, let's see how we may represent DNA
sequences. Of course our first idea is to represent them as strings:

```ocaml
type wrong_dna_sequence = string;;
let seq = "ACCCAGT";;
```

The type `wrong_dna_sequence` is a **type alias** for `string`, which
means it is a synonym (it can be used in place of it anywhere). This
representation is not really satisfying because the type
`wrong_dna_sequence` does not express a major property of DNA
sequences, which are sequences over an alphabet of four letters. And
indeed, nothing prevents us from saying that `"Hellow world"` is a DNA
sequence, which means that when writing functions that operate on DNA
sequences, we have to check that we are not given any funny character
instead of a nucleotide. As an example, let's say we want to define
the complement of a nucleotide, we'd write:

```ocaml
let complement x = match x with
  | 'A' -> 'T'
  | 'C' -> 'G'
  | 'G' -> 'C'
  | 'T' -> 'A'
  | _ -> failwith "Invalid character for nucleotide"
;;
```

Having to deal with the last case is annoying (especially if you have
to do this all over the place), but the worse is that our design here
is very brittle: what if the letters arrive in lowercase? Actually we
could deal with that by adding more cases:

```ocaml
let complement x = match x with
  | 'A' | 'a' -> 'T'
  | 'C' | 'c' -> 'G'
  | 'G' | 'g' -> 'C'
  | 'T' | 't' -> 'A'
  | _ -> failwith "Invalid character for nucleotide"
;;
```

but this is starting to be confusing, because we are paying too much
attention to encoding details. While the logic of the function is
pretty straightforward, the function looks rather complex already. In
addition it is a bit misleading: what if the capitalization was
meaningful? This function would lose this information, but nothing in
the type says that. One could easily overlook this aspect and use that
function although it is changing the data in an unacceptable way
(e.g. the capitalization represents conservation). Before moving on,
note on the previous function how we collected several cases with the
same result, e.g. `| 'A' | 'a' -> 'T'`.

Like before we can do a lot better, by defining a type that describes
just what our data is:

```ocaml
type nucleotide = A | C | G | T;;

let complement x = match x with
  | A -> T
  | C -> G
  | G -> C
  | T -> A
;;
```

By using a sum type instead of characters to represent nucleotides,
our `complement` function is now a lot simpler. In addition, note that
the compiler was able to infer a more informative type.

Well now we are ready to represent DNA sequences as list of
nucleotides:

```ocaml
type dna_sequence = nucleotide list;;

let seq = A :: G :: G :: T :: C :: A :: [];;
```

Using pattern matching, we can search for a motif in a sequence. Let's
say we'd like to find occurrences of "A[CG]NNCA", we could simply write:

```ocaml
let rec search_motif xs = match xs with
  | A :: (C | G) :: _ :: _ :: C :: A :: _ -> true
  | _ :: (   _   :: _ :: _ :: _ :: _ :: _ as next_position) ->
    search_motif next_position
  | _ -> false
;;
```

The first case is a translation of our motif to an OCaml pattern on
`nucleotide list`. If we can match on that, then it means we found an
occurrence in the sequence. Note that in addition to wildcards, we can
also use alternatives *inside* the pattern (`... :: (C | G) ::
...`). The second case is considered if the first one failed, and
correspond to any sequence of length at least the length of our
motif. In that case, we shift one step to the right and search again
(with a recursive call). Note how we can capture a specific part of a
value using the `as` keyword. Here we say that the part of the
sequence after the first element should be named `next_position` and
we reuse it in the result. The third and last case is a "catch all"
case, for sequences of length 5 or less.

N.B. I am not saying that this is how one should perform motif search
in practice. This approach is admittedly not expressive nor efficient
enough for most cases, but was chosen for illustrative purposes. Rest
assured that there are countless situations when pattern matching is
both the most convenient and efficient solution!

## Constructors with arguments

As such, sum types look a lot like what other languages call `enum`
types (for enumerations). They are in fact more general, because a
constructor can optionally take one argument (a bit like a
function). Suppose we'd like to represent mutations that can occur at
a site of a DNA sequence. Classically we'd like to represent three
types of events: insertions, deletions and substitutions. In the case
of substitutions, we'd like to describe the nucleotides before and
after mutation. Here's a possible type definition for that:

```ocaml
type mutation =
  | Insertion
  | Deletion
  | Substitution of nucleotide * nucleotide
;;
```

The first two cases are _constant_ constructors like we have seen
already; the third constructor however takes one argument, which is a
pair of nucleotides. Yes, constructors can take at most one argument,
but since this argument can be a tuple, we can stuff multiple
arguments this way.

Here is how we build values of type `mutation`:

```ocaml
let m1 = Insertion;;
let m2 = Substitution (A, G);;
```


A function performing an alignment between two sequences typically
requires a cost function to do its job. That is, it would take an
argument of type `mutation -> int`, and here is an example of such
function:

```ocaml
let cost m = match m with
  | Insertion | Deletion -> 5
  | Substitution (x, y) ->
    if x = y then 0 else 1
;;
```

## Conclusion

Sum types are a core feature of ML. They are very effective at
bringing more clarity and type checking to the code. This means that
instead of writing run-time checks all over the place, you give
appropriate types to your data, which are then verified by OCaml
automatically. The benefit of that is first that you write less of
(essentially boring) code, which lets you less opportunities to make
mistakes; a second and even greater benefit is that type checking is
performed at *compile time* (we say "statically"). In contrast,
the tests written in your code, will only show a problem when running
the program, which could take some time or even not happen at all if
you didn't run the program with a certain input.

In this view, the main challenge is to be able to encode correctness
properties of your code with the type system of OCaml (e.g. "There are
only four possible nucleotides"). Hopefully, OCaml's type system is
particularly rich and there are many clever techniques that can be
used to express complex properties. But that's yet another story.
