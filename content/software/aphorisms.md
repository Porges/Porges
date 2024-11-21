---
title: Aphorisms
tags: 
- programming
date: 2024-08-26
---
Something that has worked well for me several times (twice so far) is coming up with a catchy aphorism and then getting other people to write about it.

I need to try this again.

### implementations, not representations

Turned into the article “[Pass implementations, not representations](https://www.nichesoftware.co.nz/2017/10/14/representation-vs-implementation.html)” (2017) by Bevan Arps. 

### parse, don’t validate

[This motto](https://web.archive.org/web/20220717204533/https://twitter.com/porges/status/1182355121457385472) was turned into the wonderful “[Parse, don’t validate](https://lexi-lambda.github.io/blog/2019/11/05/parse-don-t-validate/)” (2019) by Alexis King. It’s much better than anything I could (or can) have written on the subject.[^1]

[^1]: The original thing I was working on writing back in 2017 (“a type for every concept”) I never actually got around to publishing at the time. But there’s a hint of this in the [early draft](https://github.com/Porges/Porges/commit/637feafd48b57e48eae96af8743f532b36de9d0e#:~:text=Also%2C%20prefer%20normalization%20to%20simple%20validation.).

> [!note]
> 
> For historical reference, note that the “CouchDB thing” which my original tweet was referring to  was [CVE-2017-12635](https://www.cvedetails.com/cve/CVE-2017-12635/). In this case, the built-in JavaScript JSON parser was used to _validate_ the contents of the JSON file before the (Erlang) `jiffy` parser was used to actually consume it for processing. Differences in how these two parsers worked allowed users to use duplicate JSON keys to circumvent the validation and grant themselves administrative roles.

