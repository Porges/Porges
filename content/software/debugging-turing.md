---
date: 2009-08-10
title: "Debugging Turing: an excursion with Scheme"
---
So, I thought it would be a fun idea for my first ever Lisp/Scheme program to implement Alan Turing’s original _a_-machines from his paper, _[On Computable Numbers, with an Application to the Entscheidungsproblem](https://web.archive.org/web/20090827145936/http://plms.oxfordjournals.org/cgi/reprint/s2-42/1/230)_ (paper available to public). Fun? Oh, I hadn’t any idea…

### Preamble; choice of implementation

I decided to go with the latest and greatest version of Scheme: [R⁶RS](http://www.r6rs.org/). There are currently two implementations available under Ubuntu Linux: [Ikarus](http://ikarus-scheme.org/) and [Ypsilon](http://www.littlewingpinball.net/mediawiki/index.php/Ypsilon). I installed both so I wouldn’t be swayed by any tempting extensions to the standard. 🙂 Despite this, I ended up using Ikarus for most testing, as it ran quite a lot faster, although Ypsilon gave _much_ better stack traces.

I also used the `streams` library for dealing with infinite lists ([SRFI 41](http://srfi.schemers.org/srfi-41/srfi-41.html)), which is included with both implementations.

> [!Note]
>  I’ll be providing all the code so that you _should_ be able to copy-paste it into a new file and run it.

```scheme
(import (rnrs)
    (rnrs r5rs (6)) ; provides 'delay' & 'force'
    (streams))
```

### The Machine

Turing sought to capture the essence of computation. For this purpose he constructed an idealized machine, which can read and write symbols to an infinitely long piece of tape. We are going to model these idealized machines and see what they can do.

First, we want some types to represent the _a_-machines (the _a_ is for automatic). Each machine has a set of states (_m_-configurations) it can be in, each of which contains a mapping from a set of **symbols** to a list of **actions** and a new **_m_-configuration**. I decided that the state would be my basic unit of construction, but twiddled the meaning of _m_-configuration a tiny bit so that instead of having each configuration contain a mapping from symbols, I would instead have a list of configurations, each one with a list of which symbols activate it. I ended up with the following:

```scheme
; type for a configuration
(define-record-type m-cfg
            (fields symbols operations next))
```

This defines a new record type called `m-cfg` with the fields `symbols`, `operations` and `next`. The `define-record-type` form defines a constructor (`make-m-cfg`) and accessors for each field (`m-cfg-*`). Rather than have a ‘machine’ type, I decided that this would just be left implicit; if we know what the current state is then we can follow the links to `next` whenever we want to.

### The Tape

Now, each machine operates upon an infinitely long ‘tape’. To model this, I use two streams, which are infinite lists. One is the infinite length of the tape to the left of the machine, and the other is the infinite length of the tape to the right of the machine. I decided that the first item in the right list would be the current item that the machine is reading.

```scheme
; type for a tape (modeled as two stacks)
(define-record-type tape
            (fields left right min max index))
```

You will note that there are also the fields `min`, `max`, and `index`. These are solely used to track how much of the tape the machine has “visited”. Without this information, we would not know how much of the tape to show when we want to look at it; and since it is infinitely long, this could be a problem! 🙂

### Representing operations

The `operations` that a machine can perform consist of:

- Move right
- Move left
- Print symbol
- Erase symbol
- Halt

I implemented these as an enumeration, just in case, but I didn’t actually end up utilizing any of the enumeration features.

```scheme
; the operations available
(define-enumeration op
  (right left erase print halt)
  op-set)
```

There is a small difficulty with this representation: the ‘print’ operation needs to be able to take an argument. I decided that operations would always be passed around as lists, and that only ‘print’ would have a second element in the list: its argument. Here is a little shorthand to make this representation easier:

```scheme
; these need to be lists, because print takes an argument
(define L (list (op left)))
(define R (list (op right)))
(define P (lambda (c) (list (op print) c)))
(define E (list (op erase)))
(define H (list (op halt)))
```

Thus, we can represent a list of operations like this: `(list R R (P #\A) L E H)`—that’s right, right, print ‘A’ (Scheme’s syntax for characters is a little weird), left, erase, halt.

### Moving around on the tape

Here is a simple tape; it is completely empty. Note that I use the symbol `'empty` to represent empty places on the tape. `stream-constant` makes an infinite stream of the value(s) supplied.

```scheme
(define empty-tape
  (make-tape (stream-constant 'empty) (stream-constant 'empty) 0 0 0))
```

Next we need code to actually implement the operations described above. It is fairly straightforward, but we also have to keep track of the index and max/min points on the tape:

```scheme
; Tape manipulation:
(define (current-symbol tape)
  (stream-car (tape-right tape)))
 
(define (move-item from to)
  (values (stream-cdr from)
      (stream-cons (stream-car from) to)))
 
(define (move-right tape)
  (let-values (
      [(right left) (move-item (tape-right tape) (tape-left tape))]
      [(min max) (update-min-max (tape-min tape) (tape-max tape) (+ 1 (tape-index tape)))])
    (make-tape left right min max (+ 1 (tape-index tape)))))
 
(define (move-left tape)
  (let-values (
      [(left right) (move-item (tape-left tape) (tape-right tape))]
      [(min max) (update-min-max (tape-min tape) (tape-max tape) (- (tape-index tape) 1))])
    (make-tape left right min max (- (tape-index tape) 1))))
 
(define (print tape symbol)
  (make-tape
    (tape-left tape)
    (stream-cons symbol (stream-cdr (tape-right tape)))
    (tape-min tape)
    (tape-max tape)
    (tape-index tape)))
 
(define (erase tape)
  (print tape 'empty))
 
(define (update-min-max min max i)
  (cond 
    [(< i min) (values i max)]
    [(> i max) (values min i)]
    [else      (values min max)]))
```

Notice that erase is actually redundant because we can just print `'empty`. We also want a “dispatcher” of sorts that takes a value representing an operation and performs that operation. This is where passing the argument around with the ‘print’ came in useful:

```scheme
; performs an operation on a tape, returns new tape
(define (perform-op tape oper)
  (case (car oper)
    [(op left) (move-left tape)]
    [(op right) (move-right tape)]
    [(op print) (print tape (cadr oper))]
    [(op erase) (erase tape)]
    [(op halt) #f])) ; ungraceful halt!
```

### Running the machine

Now that we have the operations implemented, we can almost run a state against a tape. First we need to figure out just which of the configurations of the state to run. This procedure receives a **list** of configurations (but they are all part of the same ‘state’) and a symbol, and picks the first configuration that has a matching symbol. Note that the configurations can also have the symbol `'any`, which matches anything.

```scheme
; finds the correct rule to follow for this symbol
(define (find-cfg machine symbol)
  (find (lambda (cfg) (or
                (find [lambda (s) (eqv? symbol s)]
                  (m-cfg-symbols cfg))
                (find [lambda (s) (eqv? 'any s)]
                  (m-cfg-symbols cfg))))
        machine))
```

Now that we have a way to find out which rule to perform, and how to perform it, we can run it against a tape. This procedure advances the machine to the next state, performing all the operations needed. It returns the new state and a new tape.

```scheme
; runs a machine forward one step
(define (run-machine tape machine)
  (let ([cfg (find-cfg (force machine) (current-symbol tape))])
    (list (fold-left perform-op tape (m-cfg-operations cfg))
          (m-cfg-next cfg))))
```

### Displaying the tape

Being able to run the machine against a tape isn’t much good if we can’t see the result, so here’s a procedure to print out what it looks like. This is where we need the indexes we kept track of on the tape, so we know when to stop printing.

```scheme
(define (print-tape tape)
  ; move as far left as possible
  (let ([leftTape (go-far-left tape)])
    ; now go right
    (do ([t leftTape])
      ((> (tape-index t) (tape-max tape)) t)
      (when (eqv? (tape-index tape) (tape-index t))
              (display "["))
      (cond
          [(eqv? (current-symbol t) 'empty) (display ".")]
          [else (display (current-symbol t))])
      (when (eqv? (tape-index tape) (tape-index t))
              (display "]"))
      (set! t (move-right t)))))
 
; moves to the far left of the tape (as far as has been travelled)
(define (go-far-left tape)
  (do ([t tape])
    ((= (tape-index t) (tape-min tape)) t)
     (set! t (move-left t))))
```

I print out a ‘.’ for each blank, and surround the current symbol with square brackets.

### Turing Machines!

Now we can test it! Here is my definition of Turing’s first published machine:

```scheme
; Turing's first published machine!
(define m1 (letrec 
  ([b (delay (list
       (make-m-cfg (list 'any) (list (P #\0) R) c)))]
   [c (delay (list
       (make-m-cfg (list 'any) (list R) e)))]
   [e (delay (list
       (make-m-cfg (list 'any) (list (P #\1) R) f)))]
   [f (delay (list
       (make-m-cfg (list 'any) (list R) b)))])
  b))
```

Now you see why we needed to import ‘delay’… since Scheme is a strictly-evaluated language, we can’t just `letrec` each state in terms of the others, so I wrap each one up in `delay`, then `force` it in one place; just before we use it in the `run-machine` procedure.

Other than that it is fairly straightforward, each state has a list of _m_-configurations, each of which has a list of what symbols it accepts, the actions to take, and the next state to move to. After we letrec, we have the initial state defined as the ‘machine’—in this case, `b`.

```scheme
We can run this machine like so:

(define (go t m)
 (do ([tm (list t m)])
  ((eqv? tm #f) (car tm))
  (print-tape (car tm))
  (newline)
  (set! tm (run-machine (car tm) (cadr tm)))))
 
(go empty-tape m1)
```

This gives us the following output:

```
[.]
0[.]
0.[.]
0.1[.]
0.1.[.]
0.1.0[.]
0.1.0.[.]
0.1.0.1[.]
0.1.0.1.[.]
0.1.0.1.0[.]
...
```

This looks correct: P0, R, R, P1, R, R, etc.

### Machine redux

The next machine Turing gives is the same as the first one, only in a different form:

```scheme
; the same machine, only smaller
(define m2 (letrec
         ([b (delay (list
           (make-m-cfg (list 'empty) (list (P #\0)) b)
           (make-m-cfg (list #\0) (list R R (P #\1)) b)
           (make-m-cfg (list #\1) (list R R (P #\0)) b)))])
         b))
```

It has only one state, but changes depending on what the current symbol is. This produces the same output as the first machine, but in fewer steps:

```
[.]
[0]
0.[1]
0.1.[0]
0.1.0.[1]
0.1.0.1.[0]
0.1.0.1.0.[1]
0.1.0.1.0.1.[0]
0.1.0.1.0.1.0.[1]
0.1.0.1.0.1.0.1.[0]
...
```

You may be wondering why Turing leaves blanks between each printed symbol. He used the convention that only the ‘even’ squares (termed _F_-squares) would be the output. The ‘odd’ squares (termed _E_-squares) would be used as a scratch-pad.

### The Third Machine

The third machine is a little more interesting. Whereas the first two printed out `01010101...`, this prints `01011011101111011111...`:

```scheme
; Turing's third machine
(define m3 (letrec
         ([b (delay (list
                   (make-m-cfg (list 'any) (list (P #\ǝ) R (P #\ǝ) R (P #\0) R R (P #\0) L L) o)))]
          [o (delay (list
                  (make-m-cfg (list #\1) (list R (P #\x) L L L) o)
                  (make-m-cfg (list #\0) (list) q)))]
          [q (delay (list
                  (make-m-cfg (list #\0 #\1) (list R R) q)
                  (make-m-cfg (list 'empty) (list (P #\1) L) p)))]
          [p (delay (list
                  (make-m-cfg (list #\x) (list E R) q)
                  (make-m-cfg (list #\ǝ) (list R) f)
                  (make-m-cfg (list 'empty) (list L L) p)))]
          [f (delay (list
                  (make-m-cfg (list 'empty) (list (P #\0) L L) o)
                  (make-m-cfg (list 'any) (list R R) f)))])
         b))
```


With the output (here I’ve used underlining to indicate the current symbol):

<pre><u>.</u>
ǝǝ<u>0</u>.0
ǝǝ<u>0</u>.0
ǝǝ0.<u>0</u>
ǝǝ0.0.<u>.</u>
ǝǝ0.0<u>.</u>1
ǝǝ0<u>.</u>0.1
ǝ<u>ǝ</u>0.0.1
ǝǝ<u>0</u>.0.1
ǝǝ0.<u>0</u>.1
ǝǝ0.0.<u>1</u>
ǝǝ0.0.1.<u>.</u>
ǝǝ0.0.<u>1</u>.0
ǝǝ0.<u>0</u>.1x0
ǝǝ0.<u>0</u>.1x0
ǝǝ0.0.<u>1</u>x0
ǝǝ0.0.1x<u>0</u>
ǝǝ0.0.1x0.<u>.</u>
ǝǝ0.0.1x0<u>.</u>1
ǝǝ0.0.1<u>x</u>0.1
ǝǝ0.0.1.<u>0</u>.1
ǝǝ0.0.1.0.<u>1</u>
ǝǝ0.0.1.0.1.<u>.</u>
ǝǝ0.0.1.0.1<u>.</u>1
ǝǝ0.0.1.0<u>.</u>1.1
ǝǝ0.0.1<u>.</u>0.1.1
ǝǝ0.0<u>.</u>1.0.1.1
ǝǝ0<u>.</u>0.1.0.1.1
ǝ<u>ǝ</u>0.0.1.0.1.1
ǝǝ<u>0</u>.0.1.0.1.1
ǝǝ0.<u>0</u>.1.0.1.1
ǝǝ0.0.<u>1</u>.0.1.1
ǝǝ0.0.1.<u>0</u>.1.1
ǝǝ0.0.1.0.<u>1</u>.1
ǝǝ0.0.1.0.1.<u>1</u>
ǝǝ0.0.1.0.1.1.<u>.</u>
ǝǝ0.0.1.0.1.<u>1</u>.0
ǝǝ0.0.1.0.<u>1</u>.1x0
ǝǝ0.0.1.<u>0</u>.1x1x0
ǝǝ0.0.1.<u>0</u>.1x1x0
ǝǝ0.0.1.0.<u>1</u>x1x0
ǝǝ0.0.1.0.1x<u>1</u>x0
ǝǝ0.0.1.0.1x1x<u>0</u>
ǝǝ0.0.1.0.1x1x0.<u>.</u>
ǝǝ0.0.1.0.1x1x0<u>.</u>1
ǝǝ0.0.1.0.1x1<u>x</u>0.1
ǝǝ0.0.1.0.1x1.<u>0</u>.1
ǝǝ0.0.1.0.1x1.0.<u>1</u>
ǝǝ0.0.1.0.1x1.0.1.<u>.</u>
ǝǝ0.0.1.0.1x1.0.1<u>.</u>1
ǝǝ0.0.1.0.1x1.0<u>.</u>1.1
ǝǝ0.0.1.0.1x1<u>.</u>0.1.1
ǝǝ0.0.1.0.1<u>x</u>1.0.1.1
ǝǝ0.0.1.0.1.<u>1</u>.0.1.1
ǝǝ0.0.1.0.1.1.<u>0</u>.1.1
ǝǝ0.0.1.0.1.1.0.<u>1</u>.1
ǝǝ0.0.1.0.1.1.0.1.<u>1</u>
ǝǝ0.0.1.0.1.1.0.1.1.<u>.</u>
ǝǝ0.0.1.0.1.1.0.1.1<u>.</u>1
ǝǝ0.0.1.0.1.1.0.1<u>.</u>1.1
ǝǝ0.0.1.0.1.1.0<u>.</u>1.1.1
ǝǝ0.0.1.0.1.1<u>.</u>0.1.1.1
ǝǝ0.0.1.0.1<u>.</u>1.0.1.1.1
ǝǝ0.0.1.0<u>.</u>1.1.0.1.1.1
ǝǝ0.0.1<u>.</u>0.1.1.0.1.1.1
ǝǝ0.0<u>.</u>1.0.1.1.0.1.1.1
ǝǝ0<u>.</u>0.1.0.1.1.0.1.1.1
ǝ<u>ǝ</u>0.0.1.0.1.1.0.1.1.1
ǝǝ<u>0</u>.0.1.0.1.1.0.1.1.1
ǝǝ0.<u>0</u>.1.0.1.1.0.1.1.1
ǝǝ0.0.<u>1</u>.0.1.1.0.1.1.1
ǝǝ0.0.1.<u>0</u>.1.1.0.1.1.1
ǝǝ0.0.1.0.<u>1</u>.1.0.1.1.1
ǝǝ0.0.1.0.1.<u>1</u>.0.1.1.1
ǝǝ0.0.1.0.1.1.<u>0</u>.1.1.1
ǝǝ0.0.1.0.1.1.0.<u>1</u>.1.1
ǝǝ0.0.1.0.1.1.0.1.<u>1</u>.1
ǝǝ0.0.1.0.1.1.0.1.1.<u>1</u>
ǝǝ0.0.1.0.1.1.0.1.1.1.<u>.</u>
ǝǝ0.0.1.0.1.1.0.1.1.<u>1</u>.0
ǝǝ0.0.1.0.1.1.0.1.<u>1</u>.1x0
ǝǝ0.0.1.0.1.1.0.<u>1</u>.1x1x0
ǝǝ0.0.1.0.1.1.<u>0</u>.1x1x1x0
ǝǝ0.0.1.0.1.1.<u>0</u>.1x1x1x0
ǝǝ0.0.1.0.1.1.0.<u>1</u>x1x1x0
ǝǝ0.0.1.0.1.1.0.1x<u>1</u>x1x0
ǝǝ0.0.1.0.1.1.0.1x1x<u>1</u>x0
ǝǝ0.0.1.0.1.1.0.1x1x1x<u>0</u>
ǝǝ0.0.1.0.1.1.0.1x1x1x0.<u>.</u>
ǝǝ0.0.1.0.1.1.0.1x1x1x0<u>.</u>1
ǝǝ0.0.1.0.1.1.0.1x1x1<u>x</u>0.1
ǝǝ0.0.1.0.1.1.0.1x1x1.<u>0</u>.1
ǝǝ0.0.1.0.1.1.0.1x1x1.0.<u>1</u>
ǝǝ0.0.1.0.1.1.0.1x1x1.0.1.<u>.</u>
ǝǝ0.0.1.0.1.1.0.1x1x1.0.1<u>.</u>1
ǝǝ0.0.1.0.1.1.0.1x1x1.0<u>.</u>1.1
ǝǝ0.0.1.0.1.1.0.1x1x1<u>.</u>0.1.1
ǝǝ0.0.1.0.1.1.0.1x1<u>x</u>1.0.1.1
ǝǝ0.0.1.0.1.1.0.1x1.<u>1</u>.0.1.1
ǝǝ0.0.1.0.1.1.0.1x1.1.<u>0</u>.1.1
ǝǝ0.0.1.0.1.1.0.1x1.1.0.<u>1</u>.1
ǝǝ0.0.1.0.1.1.0.1x1.1.0.1.<u>1</u>
ǝǝ0.0.1.0.1.1.0.1x1.1.0.1.1.<u>.</u>
ǝǝ0.0.1.0.1.1.0.1x1.1.0.1.1<u>.</u>1
ǝǝ0.0.1.0.1.1.0.1x1.1.0.1<u>.</u>1.1
ǝǝ0.0.1.0.1.1.0.1x1.1.0<u>.</u>1.1.1
ǝǝ0.0.1.0.1.1.0.1x1.1<u>.</u>0.1.1.1
ǝǝ0.0.1.0.1.1.0.1x1<u>.</u>1.0.1.1.1
ǝǝ0.0.1.0.1.1.0.1<u>x</u>1.1.0.1.1.1
ǝǝ0.0.1.0.1.1.0.1.<u>1</u>.1.0.1.1.1
ǝǝ0.0.1.0.1.1.0.1.1.<u>1</u>.0.1.1.1
ǝǝ0.0.1.0.1.1.0.1.1.1.<u>0</u>.1.1.1
ǝǝ0.0.1.0.1.1.0.1.1.1.0.<u>1</u>.1.1
ǝǝ0.0.1.0.1.1.0.1.1.1.0.1.<u>1</u>.1
ǝǝ0.0.1.0.1.1.0.1.1.1.0.1.1.<u>1</u>
ǝǝ0.0.1.0.1.1.0.1.1.1.0.1.1.1.<u>.</u>
ǝǝ0.0.1.0.1.1.0.1.1.1.0.1.1.1<u>.</u>1
ǝǝ0.0.1.0.1.1.0.1.1.1.0.1.1<u>.</u>1.1
ǝǝ0.0.1.0.1.1.0.1.1.1.0.1<u>.</u>1.1.1
ǝǝ0.0.1.0.1.1.0.1.1.1.0<u>.</u>1.1.1.1
ǝǝ0.0.1.0.1.1.0.1.1.1<u>.</u>0.1.1.1.1
ǝǝ0.0.1.0.1.1.0.1.1<u>.</u>1.0.1.1.1.1
ǝǝ0.0.1.0.1.1.0.1<u>.</u>1.1.0.1.1.1.1
ǝǝ0.0.1.0.1.1.0<u>.</u>1.1.1.0.1.1.1.1
ǝǝ0.0.1.0.1.1<u>.</u>0.1.1.1.0.1.1.1.1
ǝǝ0.0.1.0.1<u>.</u>1.0.1.1.1.0.1.1.1.1
ǝǝ0.0.1.0<u>.</u>1.1.0.1.1.1.0.1.1.1.1
ǝǝ0.0.1<u>.</u>0.1.1.0.1.1.1.0.1.1.1.1
ǝǝ0.0<u>.</u>1.0.1.1.0.1.1.1.0.1.1.1.1
ǝǝ0<u>.</u>0.1.0.1.1.0.1.1.1.0.1.1.1.1
ǝ<u>ǝ</u>0.0.1.0.1.1.0.1.1.1.0.1.1.1.1
</pre>

Here we can see the scratch-pad being used. We can also see the beginnings of a pattern of execution; notice how the machine returns to the beginning of the tape after completing each set of `1`s.

### Opus Magnum

Next up was my main challenge. One of the major contributions of Turing’s paper was to display a machine which could emulate _any other_ machine you wanted. In essence, you don’t actually need lots of different machines. You can just build one, and it can do anything any of the other machines can do! This is the principle behind modern general-purpose computers.

This is a very big, and complex machine. Some of the intricacies not involved in the other machines are:

- _m_-functions; that is, configurations which accept parameters—luckily, this was surprisingly easy to implement using Scheme; I simply wrap the `delay`ed code inside another lambda
- _m_-configurations which are parametrized over all symbols on the machine—this is solved by just `map`ping a lambda over the list of symbols
- variadic _m_-functions—solved by the magic of `case-lambda`
- poorly-scanned journal document containing Fraktur and Greek letters

But not only was this by far the biggest and most complex machine supplied by Turing, I had read [texts](http://www.turing.org.uk/turing/scrapbook/machine.html) mentioning unspecified _bugs_ in the program.

… and sure enough, I ran into a ‘bug’. There were some configurations used in the machine which _weren’t defined in the paper_! At first I thought this was due to the low resolution of the PDF, but even enhancing the image didn’t help.

After much supplication and burnt offerings to the God of the Internet, I managed to find that:

#### Someone else did the work already

Yay!

More specifically, I found a paper entitled _[Understanding Turing’s Universal Machine — Personal Style in Program Description](https://doi.org/10.1093/comjnl/36.4.351)_ (which is unfortunately not available to the public), a marvelous paper that not only explains the errors made in detail, but also provides a nice, corrected version of Turing’s exposition of his machine.

After painstakingly re-checking all the states again, I arrived at this:

```scheme
; need this to generate a couple of cfgs
(define u-symbols (list
		   #\A #\C #\D #\0 #\1
		   #\u #\v #\w #\x #\y #\z
		   #\; #\L #\R #\N
		   #\∷ #\:
		   ))
 
; yo dawg
(define u (letrec
        (
         [f (lambda (C B a) (delay (list
                (make-m-cfg (list #\ǝ) (list L) (f1 C B a))
                (make-m-cfg (list 'any) (list L) (f C B a)))))]
         [f1 (lambda (C B a) (delay (list
                (make-m-cfg (list a) (list) C)
                (make-m-cfg (list 'empty) (list R) (f2 C B a))
                (make-m-cfg (list 'any) (list R) (f1 C B a)))))]
         [f2 (lambda (C B a) (delay (list
                (make-m-cfg (list a) (list) C)
                (make-m-cfg (list 'empty) (list R) B)
                (make-m-cfg (list 'any) (list R) (f1 C B a)))))]
         [fdash (lambda (C B a) (delay (list
                (make-m-cfg (list 'any) (list) (f (l C) B a)))))]
         [fdashdash (lambda (C B a) (delay (list
                (make-m-cfg (list 'any) (list) (f (r C) B a)))))]
         [r (lambda (C) (delay (list
                (make-m-cfg (list 'any) (list R) C))))]
         [l (lambda (C) (delay (list
                (make-m-cfg (list 'any) (list L) C))))]
         [q (case-lambda
              [(C) (delay (list
                (make-m-cfg (list 'empty) (list R) (q1 C))
                (make-m-cfg (list 'any) (list R) (q C))))]
         [(C a) (delay (list
                (make-m-cfg (list 'any) (list) (q (q1 C a)))))])]
         [q1 (case-lambda
               [(C) (delay (list
                (make-m-cfg (list 'empty) (list) C)
                (make-m-cfg (list 'any) (list R) (q C))))]
               [(C a) (delay (list
                (make-m-cfg (list a) (list) C)
                (make-m-cfg (list 'any) (list L) (q1 C a))))])]
         [pe (lambda (C b) (delay (list
                (make-m-cfg (list 'any) (list) (f (pe1 C b) c #\ǝ)))))]
         [pe1 (lambda (C b) (delay (list
                (make-m-cfg (list 'empty) (list (P b)) C)
                (make-m-cfg (list 'any) (list R R) (pe1 C b)))))]
         [pe2 (lambda (C a b) (delay (list
                (make-m-cfg (list 'any) (list) (pe (pe C b) a)))))]
         [c (lambda (C B a) (delay (list
                (make-m-cfg (list 'any) (list) (fdash (c1 C) B a)))))]
         [c1 (lambda (C) (delay (map
                    (lambda (b) (make-m-cfg (list b) (list) (pe C b)))
                      u-symbols)))]
         [ce (case-lambda
               [(C B a) (delay (list
                (make-m-cfg (list 'any) (list) (c (e C B a) B a))))]
               [(B a) (delay (list
                (make-m-cfg (list 'any) (list) (ce (ce B a) B a))))])]
         [ce2 (lambda (B a b) (delay (list
                (make-m-cfg (list 'any) (list) (ce (ce B b) a)))))]
         [ce3 (lambda (B a b g) (delay (list
                (make-m-cfg (list 'any) (list) (ce (ce2 B b g) a)))))]
         [ce5 (lambda (B a b g d e) (delay (list
                (make-m-cfg (list 'any) (list) (ce3 (ce2 B d e) a b g)))))] ; added
         [cp (lambda (C U F a b) (delay (list
                (make-m-cfg (list 'any) (list) (fdash (cp1 C U b) (f U F b) a)))))]
         [cp1 (lambda (C U b) (delay (map
                   (lambda (g) (make-m-cfg (list g) (list) (fdash (cp2 C U g) U b)))
                     u-symbols)))]
         [cp2 (lambda (C U g) (delay (list
                (make-m-cfg (list g) (list) C)
                (make-m-cfg (list 'any) (list) U))))]
         [cpe (case-lambda 
                [(C U F a b) (delay (list
                 (make-m-cfg (list 'any) (list) (cp (e (e C C b) C a) U F a b))))]
                [(U F a b) (delay (list
                 (make-m-cfg (list 'any) (list) (cpe (cpe U F a b) U F a b))))])]
         [e (case-lambda
               [(C) (delay (list
                (make-m-cfg (list #\ǝ) (list R) (e1 C))
                (make-m-cfg (list 'any) (list L) (e C))))]
               [(B a) (delay (list 
                (make-m-cfg (list 'any) (list) (e (e B a) B a))))]
               [(C B a) (delay (list 
                (make-m-cfg (list 'any) (list) (f (e1 C B a) B a))))])]
         [e1 (case-lambda
               [(C) (delay (list
                (make-m-cfg (list 'empty) (list) C) 
                (make-m-cfg (list 'any) (list R E R) (e1 C))))]
               [(C B a) (delay (list
                (make-m-cfg (list 'any) (list E) C)))])]
         [con (lambda (C a) (delay (list
                (make-m-cfg (list #\A) (list L (P a) R) (con1 C a))
                (make-m-cfg (list 'any) (list R R) (con C a)))))]
         [con1 (lambda (C a) (delay (list
                (make-m-cfg (list #\A) (list R (P a) R) (con1 C a))
                (make-m-cfg (list #\D) (list R (P a) R) (con2 C a)))))]
         [con2 (lambda (C a) (delay (list
                (make-m-cfg (list #\C) (list R (P a) R) (con2 C a))
                (make-m-cfg (list 'any) (list R R) C))))]
         [b (delay (list
                (make-m-cfg (list 'any) (list) (f b1 b1 #\∷))))]
         [b1 (delay (list
                (make-m-cfg (list 'any) (list R R (P #\:) R R (P #\D) R R (P #\A) R R (P #\D)) anf)))] ; added "R R PD"
         [anf (delay (list
                (make-m-cfg (list 'any) (list) (q anf1 #\:))))] ; corrected from "(g ..."
         [anf1 (delay (list
                (make-m-cfg (list 'any) (list) (con fom #\y))))]
         [fom (lambda () (list
                (make-m-cfg (list #\;) (list R (P #\z) L) (con fmp #\x))
                (make-m-cfg (list #\z) (list L L) fom)
                (make-m-cfg (list #\ǝ) (list H) fom)
                (make-m-cfg (list 'any) (list L) fom)))]
         [fmp (delay (list
                (make-m-cfg (list 'any) (list) (cpe (e (e anf #\x) #\y) sim #\x #\y))))] ; corrected
         [sim (delay (list
                (make-m-cfg (list 'any) (list) (fdash sim1 sim1 #\z))))]
         [sim1 (delay (list
                (make-m-cfg (list 'any) (list) (con sim2 'empty))))]
         [sim2 (delay (list
                (make-m-cfg (list #\A) (list) sim3)
                (make-m-cfg (list 'any) (list L (P #\u) R R R) sim2)))] ; corrected from "R ..."
         [sim3 (delay (list
                (make-m-cfg (list #\A) (list L (P #\y) R R R) sim3)
                (make-m-cfg (list 'any) (list L (P #\y)) (e mf #\z))))]
         [mf (delay (list
                (make-m-cfg (list 'any) (list) (q mf1 #\:))))] ; corrected from "(g mf ..."
         [mf1 (delay (list
                (make-m-cfg (list #\A) (list L L L L) mf2)
                (make-m-cfg (list 'any) (list R R) mf1)))]
         [mf2 (delay (list
                (make-m-cfg (list #\C) (list R (P #\x) L L L) mf2)
                (make-m-cfg (list #\:) (list) mf4)
                (make-m-cfg (list #\D) (list R (P #\x) L L L) mf3)))]
         [mf3 (delay (list
                (make-m-cfg (list #\:) (list) mf4)
                (make-m-cfg (list 'any) (list R (P #\v) L L L) mf3)))]
         [mf4 (delay (list
                (make-m-cfg (list 'any) (list) (con (l (l mf5)) 'empty))))]
         [mf5 (delay (list
                (make-m-cfg (list 'empty) (list (P #\:)) sh)
                (make-m-cfg (list 'any) (list R (P #\w) R) mf5)))]
         [sh (delay (list
                (make-m-cfg (list 'any) (list) (f sh1 inst #\u))))]
         [sh1 (delay (list
                (make-m-cfg (list 'any) (list L L L) sh2)))]
         [sh2 (delay (list
                (make-m-cfg (list #\D) (list R R R R) sh3) ; corrected from "sh2"
                (make-m-cfg (list 'any) (list) inst)))]
         [sh3 (delay (list
                (make-m-cfg (list #\C) (list R R) sh4)
                (make-m-cfg (list 'any) (list) inst)))]
         [sh4 (delay (list
                (make-m-cfg (list #\C) (list R R) sh5)
                (make-m-cfg (list 'any) (list) (pe2 inst #\0 #\:))))]
         [sh5 (delay (list
                (make-m-cfg (list #\C) (list) inst)
                (make-m-cfg (list 'any) (list) (pe2 inst #\1 #\:))))]
         [inst (delay (list                                 ; note that inst1 is forced here!
                                                            ; this is because it is a zero-arity varargs
                (make-m-cfg (list 'any) (list) (q (l (inst1)) #\u))))] ; corrected from "(g ..." 
         [inst1 (case-lambda
                 [() (delay (map
                       (lambda (a) (make-m-cfg (list a) (list R E) (inst1 a)))
                       u-symbols))]
                 [(x) (case x
                      [(#\L) (delay (list (make-m-cfg (list 'any) (list) (ce5 ov #\v #\y #\x #\u #\w))))]
                      [(#\R) (delay (list (make-m-cfg (list 'any) (list) (ce5 ov #\v #\x #\u #\y #\w))))]
                      [(#\N) (delay (list (make-m-cfg (list 'any) (list) (ce5 ov #\v #\x #\y #\u #\w))))])])]
         [ov (delay (list
               (make-m-cfg (list 'any) (list) (q (r (r ov1)) #\A))))] ; changed from original
         [ov1 (delay (list
               (make-m-cfg (list #\D) (list) (e anf))
               (make-m-cfg (list 'empty) (list (P #\D)) (e anf))))]
         ) b)) ;; start in state 'b'
```

I don’t think I can blame Turing much for the errors in the paper. It seems as though some arose through printing typos, and attempting to debug this baroque machine by hand, on paper, would have been a difficult task. (I don’t think he even had Visual Studio!)

### A machine in a machine

Of course, the final test of this is to see whether this machine can actually emulate another like it is supposed to. I defined a short procedure to set up a machine on a tape according to Turing’s ingenious encoding.

```scheme
(define (setup-tape tape inits) 
  (let ([t (move-right (print (move-right (print tape #\ǝ)) #\ǝ))]
  	[inits (append inits (list #\∷))])
   (go-far-left (fold-left (lambda (tape init)
  	       (move-right (move-right (print tape init)))) 
	     t inits))))     
```

#### A quick explanation of Turing’s encoding

The idea is to first simplify, then encode the states. Turing noted that many of the _m_-configurations’ operations could be considered redundant:

- as mentioned above, instead of having “erase” we can simply print the ‘blank’ symbol
- instead of _not_ modifying the symbol, we simply print the symbol already present

There are then only three types of operation each state needs to perform:

1. print something and go left
2. print something and go right
3. print something and stay put

States which have a sequence of operations can be split into a series of states, each of which transfers control to the next one.

Now, if we encode all the symbols and configurations as numbers, we can write them out. Turing chose to encode them via this scheme:

- configurations’ numbers are the symbol ‘D’ followed by _n_ ‘A’s, where _n_ is the number of the state
- symbols’ numbers are similar, with ‘D’ followed by _n_ ‘C’s. (Turing set ‘blank’ to always be symbol 0, represented as simply “D”)

So to encode each configuration, we write down its number, the symbol it accepts, the symbol it outputs, which direction to move, and which state to go to next. We prefix each configuration with ‘;’. (When we input it into the machine we also sandwich the whole thing between ‘ǝǝ’ and ‘∷’.)

Here is the example which Turing gives in his paper. I have formatted it to make it easier to read. Can you tell what it does?

```scheme
(define example (setup-tape empty-tape (list 
			     #\; #\D #\A             #\D #\D #\C     #\R #\D #\A #\A
			     #\; #\D #\A #\A         #\D #\D         #\R #\D #\A #\A #\A
			     #\; #\D #\A #\A #\A     #\D #\D #\C #\C #\R #\D #\A #\A #\A #\A
			     #\; #\D #\A #\A #\A #\A #\D #\D         #\R #\D #\A)))		     
```

If we translate the symbols to get numbers we have the following:

```scheme
; 1 0 0 R 2
; 2 0 . R 3
; 3 0 1 R 4
; 4 0 . R 1
```

This machine prints alternately `0.1.0.1.`. In fact, when I first typed it up, I left off an ‘A’ on the 3rd state and couldn’t figure out why the machine was printing `0.11111...`!

### Just to show it works

Here is some output of the universal machine running the ‘0101′ machine. I have only included a snippet as the machine takes a while to get to this stage. You’ll also notice the output format is different from the other machines; this one outputs some state information and the output of the emulated machine (in this case, 0 and 1), separated by colons. So far, after a minute or so, the machine has output `010` 😀

<pre style="overflow:auto">.ǝǝ;.D.A.D.DuCuR.DyAyAy;.D.A.A.D.D.R.D.A.A.A.;.D.A.A.A.D.D.C.C.R.D.A.A.A.A.;.D.A.A.A.A.D.D.R.D<u>.</u>A.∷.:.D.A.D.:.0.:.D.C.D.A.A.D.:.D.C.D.D.A.A.A.D.:.1.:.D.C.D.D.C.C.D.A.A.A.A.D.:.D.CvDvDvCvCvDxD.A.D.:.0.:.D.C
.ǝǝ;.D.A.D.DuCuR.DyAyAy;.D.A.A.D.D.R.D.A.A.A.;.D.A.A.A.D.D.C.C.R.D.A.A.A.A.;.D.A.A.A.A.D.D.R.D.<u>A</u>.∷.:.D.A.D.:.0.:.D.C.D.A.A.D.:.D.C.D.D.A.A.A.D.:.1.:.D.C.D.D.C.C.D.A.A.A.A.D.:.D.CvDvDvCvCvDxD.A.D.:.0.:.D.C
.ǝǝ;.D.A.D.DuCuR.DyAyAy;.D.A.A.D.D.R.D.A.A.A.;.D.A.A.A.D.D.C.C.R.D.A.A.A.A.;.D.A.A.A.A.D.D.R.D.A<u>.</u>∷.:.D.A.D.:.0.:.D.C.D.A.A.D.:.D.C.D.D.A.A.A.D.:.1.:.D.C.D.D.C.C.D.A.A.A.A.D.:.D.CvDvDvCvCvDxD.A.D.:.0.:.D.C
.ǝǝ;.D.A.D.DuCuR.DyAyAy;.D.A.A.D.D.R.D.A.A.A.;.D.A.A.A.D.D.C.C.R.D.A.A.A.A.;.D.A.A.A.A.D.D.R.D.A.<u>∷</u>.:.D.A.D.:.0.:.D.C.D.A.A.D.:.D.C.D.D.A.A.A.D.:.1.:.D.C.D.D.C.C.D.A.A.A.A.D.:.D.CvDvDvCvCvDxD.A.D.:.0.:.D.C
.ǝǝ;.D.A.D.DuCuR.DyAyAy;.D.A.A.D.D.R.D.A.A.A.;.D.A.A.A.D.D.C.C.R.D.A.A.A.A.;.D.A.A.A.A.D.D.R.D.A.∷<u>.</u>:.D.A.D.:.0.:.D.C.D.A.A.D.:.D.C.D.D.A.A.A.D.:.1.:.D.C.D.D.C.C.D.A.A.A.A.D.:.D.CvDvDvCvCvDxD.A.D.:.0.:.D.C
.ǝǝ;.D.A.D.DuCuR.DyAyAy;.D.A.A.D.D.R.D.A.A.A.;.D.A.A.A.D.D.C.C.R.D.A.A.A.A.;.D.A.A.A.A.D.D.R.D.A.∷.<u>:</u>.D.A.D.:.0.:.D.C.D.A.A.D.:.D.C.D.D.A.A.A.D.:.1.:.D.C.D.D.C.C.D.A.A.A.A.D.:.D.CvDvDvCvCvDxD.A.D.:.0.:.D.C
.ǝǝ;.D.A.D.DuCuR.DyAyAy;.D.A.A.D.D.R.D.A.A.A.;.D.A.A.A.D.D.C.C.R.D.A.A.A.A.;.D.A.A.A.A.D.D.R.D.A.∷.:<u>.</u>D.A.D.:.0.:.D.C.D.A.A.D.:.D.C.D.D.A.A.A.D.:.1.:.D.C.D.D.C.C.D.A.A.A.A.D.:.D.CvDvDvCvCvDxD.A.D.:.0.:.D.C
.ǝǝ;.D.A.D.DuCuR.DyAyAy;.D.A.A.D.D.R.D.A.A.A.;.D.A.A.A.D.D.C.C.R.D.A.A.A.A.;.D.A.A.A.A.D.D.R.D.A.∷.:.<u>D</u>.A.D.:.0.:.D.C.D.A.A.D.:.D.C.D.D.A.A.A.D.:.1.:.D.C.D.D.C.C.D.A.A.A.A.D.:.D.CvDvDvCvCvDxD.A.D.:.0.:.D.C
.ǝǝ;.D.A.D.DuCuR.DyAyAy;.D.A.A.D.D.R.D.A.A.A.;.D.A.A.A.D.D.C.C.R.D.A.A.A.A.;.D.A.A.A.A.D.D.R.D.A.∷.:.D<u>.</u>A.D.:.0.:.D.C.D.A.A.D.:.D.C.D.D.A.A.A.D.:.1.:.D.C.D.D.C.C.D.A.A.A.A.D.:.D.CvDvDvCvCvDxD.A.D.:.0.:.D.C
.ǝǝ;.D.A.D.DuCuR.DyAyAy;.D.A.A.D.D.R.D.A.A.A.;.D.A.A.A.D.D.C.C.R.D.A.A.A.A.;.D.A.A.A.A.D.D.R.D.A.∷.:.D.<u>A</u>.D.:.0.:.D.C.D.A.A.D.:.D.C.D.D.A.A.A.D.:.1.:.D.C.D.D.C.C.D.A.A.A.A.D.:.D.CvDvDvCvCvDxD.A.D.:.0.:.D.C
.ǝǝ;.D.A.D.DuCuR.DyAyAy;.D.A.A.D.D.R.D.A.A.A.;.D.A.A.A.D.D.C.C.R.D.A.A.A.A.;.D.A.A.A.A.D.D.R.D.A.∷.:.D.A<u>.</u>D.:.0.:.D.C.D.A.A.D.:.D.C.D.D.A.A.A.D.:.1.:.D.C.D.D.C.C.D.A.A.A.A.D.:.D.CvDvDvCvCvDxD.A.D.:.0.:.D.C
.ǝǝ;.D.A.D.DuCuR.DyAyAy;.D.A.A.D.D.R.D.A.A.A.;.D.A.A.A.D.D.C.C.R.D.A.A.A.A.;.D.A.A.A.A.D.D.R.D.A.∷.:.D.A.<u>D</u>.:.0.:.D.C.D.A.A.D.:.D.C.D.D.A.A.A.D.:.1.:.D.C.D.D.C.C.D.A.A.A.A.D.:.D.CvDvDvCvCvDxD.A.D.:.0.:.D.C
.ǝǝ;.D.A.D.DuCuR.DyAyAy;.D.A.A.D.D.R.D.A.A.A.;.D.A.A.A.D.D.C.C.R.D.A.A.A.A.;.D.A.A.A.A.D.D.R.D.A.∷.:.D.A.D<u>.</u>:.0.:.D.C.D.A.A.D.:.D.C.D.D.A.A.A.D.:.1.:.D.C.D.D.C.C.D.A.A.A.A.D.:.D.CvDvDvCvCvDxD.A.D.:.0.:.D.C
.ǝǝ;.D.A.D.DuCuR.DyAyAy;.D.A.A.D.D.R.D.A.A.A.;.D.A.A.A.D.D.C.C.R.D.A.A.A.A.;.D.A.A.A.A.D.D.R.D.A.∷.:.D.A.D.<u>:</u>.0.:.D.C.D.A.A.D.:.D.C.D.D.A.A.A.D.:.1.:.D.C.D.D.C.C.D.A.A.A.A.D.:.D.CvDvDvCvCvDxD.A.D.:.0.:.D.C
.ǝǝ;.D.A.D.DuCuR.DyAyAy;.D.A.A.D.D.R.D.A.A.A.;.D.A.A.A.D.D.C.C.R.D.A.A.A.A.;.D.A.A.A.A.D.D.R.D.A.∷.:.D.A.D.:<u>.</u>0.:.D.C.D.A.A.D.:.D.C.D.D.A.A.A.D.:.1.:.D.C.D.D.C.C.D.A.A.A.A.D.:.D.CvDvDvCvCvDxD.A.D.:.0.:.D.C
.ǝǝ;.D.A.D.DuCuR.DyAyAy;.D.A.A.D.D.R.D.A.A.A.;.D.A.A.A.D.D.C.C.R.D.A.A.A.A.;.D.A.A.A.A.D.D.R.D.A.∷.:.D.A.D.:.<u>0</u>.:.D.C.D.A.A.D.:.D.C.D.D.A.A.A.D.:.1.:.D.C.D.D.C.C.D.A.A.A.A.D.:.D.CvDvDvCvCvDxD.A.D.:.0.:.D.C
.ǝǝ;.D.A.D.DuCuR.DyAyAy;.D.A.A.D.D.R.D.A.A.A.;.D.A.A.A.D.D.C.C.R.D.A.A.A.A.;.D.A.A.A.A.D.D.R.D.A.∷.:.D.A.D.:.0<u>.</u>:.D.C.D.A.A.D.:.D.C.D.D.A.A.A.D.:.1.:.D.C.D.D.C.C.D.A.A.A.A.D.:.D.CvDvDvCvCvDxD.A.D.:.0.:.D.C
.ǝǝ;.D.A.D.DuCuR.DyAyAy;.D.A.A.D.D.R.D.A.A.A.;.D.A.A.A.D.D.C.C.R.D.A.A.A.A.;.D.A.A.A.A.D.D.R.D.A.∷.:.D.A.D.:.0.<u>:</u>.D.C.D.A.A.D.:.D.C.D.D.A.A.A.D.:.1.:.D.C.D.D.C.C.D.A.A.A.A.D.:.D.CvDvDvCvCvDxD.A.D.:.0.:.D.C
.ǝǝ;.D.A.D.DuCuR.DyAyAy;.D.A.A.D.D.R.D.A.A.A.;.D.A.A.A.D.D.C.C.R.D.A.A.A.A.;.D.A.A.A.A.D.D.R.D.A.∷.:.D.A.D.:.0.:<u>.</u>D.C.D.A.A.D.:.D.C.D.D.A.A.A.D.:.1.:.D.C.D.D.C.C.D.A.A.A.A.D.:.D.CvDvDvCvCvDxD.A.D.:.0.:.D.C
.ǝǝ;.D.A.D.DuCuR.DyAyAy;.D.A.A.D.D.R.D.A.A.A.;.D.A.A.A.D.D.C.C.R.D.A.A.A.A.;.D.A.A.A.A.D.D.R.D.A.∷.:.D.A.D.:.0.:.<u>D</u>.C.D.A.A.D.:.D.C.D.D.A.A.A.D.:.1.:.D.C.D.D.C.C.D.A.A.A.A.D.:.D.CvDvDvCvCvDxD.A.D.:.0.:.D.C
.ǝǝ;.D.A.D.DuCuR.DyAyAy;.D.A.A.D.D.R.D.A.A.A.;.D.A.A.A.D.D.C.C.R.D.A.A.A.A.;.D.A.A.A.A.D.D.R.D.A.∷.:.D.A.D.:.0.:.D<u>.</u>C.D.A.A.D.:.D.C.D.D.A.A.A.D.:.1.:.D.C.D.D.C.C.D.A.A.A.A.D.:.D.CvDvDvCvCvDxD.A.D.:.0.:.D.C
</pre>

And that’s enough for today! Feel free to post corrections, additions, your own Turing machines, and so on 🙂