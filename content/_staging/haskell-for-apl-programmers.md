---
title: A Guide to Haskell for APL programmers
draft: true
---

Welcome! This is a tutorial for those of you in the APL community who are interested in learning about Haskell, a language which has been gaining mindshare amongst the hobbyist programmer community. For those of you in the Haskell community you may be able to apply this information an inverse manner[^1] and perhaps gain some more knowledge about the wonderful world of array programming languages.[^2]

[^1]: For which, see [[#Inverse functions]].
[^2]: But it is important to note that the “A” stands for “A” and not “Array”.

## Why learn Haskell?

Maybe you’ve been writing APL for a few years [since it was released in 1978](https://dl.acm.org/doi/10.1145/3386319), and you might like its design. You might even be aware that Iverson recently won the [Turing Award](https://web.archive.org/web/20091223064709/http://awards.acm.org/citation.cfm?id=9147499&srt=all&aw=140&ao=AMTURING) for his work on the language. However, despite APL itself being [almost perfect](https://www.jsoftware.com/papers/perlis78.htm), there are still reasons to try other languages.

For one thing, the functional style has also [become popular](https://archive.vector.org.uk/art10007770) for writing APL programs, so we should explore what is perhaps the functional language to end all functional languages: indeed, Haskell was devised by [a cabal of Portlanders](https://www.haskell.org/onlinereport/preface-jfp.html) in order to unify an ecosystem of “more than a dozen non-strict, purely functional programming languages, all similar in expressive power and semantic underpinnings”.

As we all know, APL is “the language of the future” — rare words of praise from Dijkstra — so let us compare it to this programming technique from the past (Haskell predates Windows 3.1 and the World Wide Web!)

## Shared features

Many of the core abilities of Haskell and APL are similar. In this section we will examine some of their shared features.

### Adding numbers

The primary operation of a computer being a tool of _business_,[^3] we will always want to compute how much the line has gone up. For this, we will need to add numbers to other numbers. Thankfully this operation is simple in Haskell, and looks much the same there…

[^3]: If I may be forgiven, this is a [small joke](https://aplwiki.com/wiki/IBM) on my part!

```haskell
1.0 + 2.7 -- = 3.7
```

… as it does in APL:

```apl
1 + 2.7 ⍝ = 3.7
```

Haskell does, however, maintain adherence to the legacy <span style="font-variant-caps: small-caps">BedMas</span>[^4] operator ambiguity resolution scheme and so one must be careful when mixing operators:

[^4]: Marketed in some jurisdictions under license as PEMDAS or BiDMaS/BoDMaS. The variant from Dublin is apparently preferred due to its low mineral content.

![[Pasted image 20241206152641.png|YouTube Thumbnail image: Trick Facebook question EXPLAINED!]]

```haskell
6 / 2*(1+2) -- = 9.0
```

Whereas APL gives the predictable:

```apl
6 ÷ 2×(1+2) ⍝ = 1
```

 It is also important to note that in Haskell, different types of numbers must be combined in different ways, and it does not yet implement the _[Iverson unification](https://en.wikipedia.org/wiki/Iverson_bracket)_. If we want to multiply normal Boolean numbers we must use a different linguistic subset:

```haskell
All True <> All False -- All { getAll = False }
```

Compare that to:

```apl
1×0 ⍝ = 0
```

Because of this strict differentiation, Haskell also does not have the _Compress–Replicate merger_ and, for example, instead of `filter (\x -> x * x) [1, 2, 3]` one must write `[1, 2, 3] >>= join replicate`. But such is life.

Of course, if we want to add many numbers, we really want to _fold_ them.

### Implementing folds

Let us first remind ourselves of some common APL definitions:

```apl
sum ← +/
max ← ⌈/
```

Implementing folds in Haskell is similar, albeit more error-prone due to the need to explicitly specify the inherent monoidal identity:

```haskell
sumOf = foldr (+) 0
maxOf = foldr max minBound
```

> [!Note]
> You will see that the Haskell compiler, unlike most APL development environments, is often unsure of a symbol’s _type_, and we must help the compiler by gently handling the `+` operator to it with parentheses.

The wordiness of Haskell may at first be off-putting. We will soon encounter some proper symbolic names which should put any early fears to rest — this is not [FLOW-MATIC](https://en.wikipedia.org/wiki/FLOW-MATIC)!

### Composition of morphisms

One area in which APL has historically struggled is when composing morphisms in the category of vector spaces; this operation has always been relatively wordy compared to most of the other functionality provided by the language:

```apl
x +.× y
```

Haskell does gain some advantage here where (compared to the other operations at least) we do have a specialized operator for performing this operation. To be clear, it is actually no shorter than the APL version but the forward-thinking involved in reserving this important operator does show the suitability of Haskell as a target for high-performance categorically-informed numeric array programming:

```haskell
y <<< x
```

Dare I claim that both languages are eminently suited for ~~machine learning~~ AI?

## Contrivances

### Inverse functions

Unfortunately, despite being a “functional” language, Haskell has only poor support for function inversion. While we can easily define:

```apl
to_f ← (32∘+) ∘ (1.8∘×)
to_c ← to_f ⍣ ¯1
```

In Haskell this requires some high-powered machinery: