---
title: "Call by Name: An Evaluation Problem"
date: 2019-01-17 00:00:00 +0100
tags: [programming, functional-programming, evaluation-strategies, computer-science]
description: "An explanation of call-by-name evaluation in functional programming — when arguments are evaluated lazily rather than eagerly, why it matters, and where it appears in real languages."
---

In computer science, programming languages are standardized notations that allow us to describe very precisely a set of instructions for a computer to execute. Languages differ from one another by their semantics, their syntax, and their paradigm.

One of the best-known paradigms is the **functional paradigm**, which treats computation as the evaluation of functions.

## What is evaluation?

Programming languages can take different approaches to deciding *when* to evaluate the arguments of a function call, and *how* to pass those arguments to the function.

The most common strategy is **call by value**. In call by value, arguments are evaluated first, then the function is applied.

But other strategies exist — notably **call by name**.

In call by name, the function is evaluated first. Arguments are only evaluated if they are actually used, and when they are, they are re-evaluated *each time* they appear.

## The consequences of call by name

To better understand what these different strategies imply, consider the following function:

```
f(x: int, y: int) -> x + x
```

Take `f(4, 3)` and look at how each strategy handles it.

With **call by value**, the computation unfolds in three steps:

```
f(4, 3) -> 4 + 4 -> 8
```

With **call by name**, the function is evaluated first, and only the arguments that are actually used are then evaluated. Here, the result `x + x` only depends on the first argument. So the computation takes two steps:

```
4 + 4 -> 8
```

Call by name reaches the same result in fewer steps. Does that mean it's more efficient?

Not necessarily. Consider a different function:

```
g(x: int, y: int) -> x * y
```

With `x = 3+1` and `y = 2*4`:

**Call by value** evaluates the arguments first — three steps:

```
g(4, 8) -> 4 * 8 -> 32
```

**Call by name** evaluates the function first, then each argument where it appears — four steps:

```
(3+1) * (2*4) -> 4 * (2*4) -> 4 * 8 -> 32
```

Call by name can also **change the result** of a function entirely. Consider:

```
h(i: int) -> [i, i, i]   where i is drawn at random
```

With **call by value**, `i` is evaluated (drawn) once, then placed three times in the list:

```
h(i) -> [3, 3, 3]
```

With **call by name**, the function is evaluated first, and each occurrence of `i` is evaluated independently:

```
h(i) -> [2, 5, 9]
```

In the first case, `i` is evaluated once. In the second, it is evaluated three times — giving a different result and costing more computation.

## Does evaluation always terminate?

For any algorithm, **termination** is a fundamental property: the operations described must complete in finite time. Define the recursive function `loop`, which calls itself forever:

```
loop(x) -> loop(x)
```

Now apply our original function `f` with arguments `3` and `loop(2)`:

```
f(3, loop(2))
```

With **call by name**, only the first argument is evaluated: `3 + 3 -> 6`. Done in two steps.

With **call by value**, all arguments are evaluated — including `loop(2)`, which runs forever.

So there are cases where call by name terminates when call by value does not.

## What is it actually used for?

Call by name is one of the evaluation strategies — alongside call by value — that defines functional programming.

An **abstract machine** is a set of inputs, states, and transitions that rigorously defines the computational steps used as a model for implementation. Two abstract machines exist for functional languages: the **SECD machine**, based on call by value, and the **Krivine machine**, based on call by name. The SECD machine was used to build Lispkit, a purely functional Lisp dialect. The Krivine machine is not used in practice.

**ALGOL 60** was one of the first languages to use call by name, in the 1960s. Today, **Scala** allows call-by-name evaluation as an opt-in, but like most languages, it defaults to call by value.

There is, however, an optimized variant of call by name: **call by need**.

In call by need, evaluations are **memoized** — stored in memory. The function maintains a kind of dictionary, mapping variable names to their already-computed values. If a value in the dictionary needs to be used again, it is retrieved directly rather than recomputed. This is called **lazy evaluation**: only used arguments are evaluated, and each only once.

*[Illustration: a cartoon of a student procrastinating — lazy evaluation, a bit like someone who puts things off, turns out to be very efficient.]*

Languages that support lazy evaluation are mostly functional. **Haskell** is lazy by default. **Scheme** and **OCaml** also allow lazy evaluation when needed.

## Which evaluation strategy should you choose?

There is no definitive answer. The choice of evaluation strategy is a longstanding debate in functional programming, and no single strategy is optimal in all cases.

Call by name terminates more often than call by value. It is also more efficient when some function arguments are never used — since those are never computed. However, the order in which arguments are evaluated is difficult to predict.

Call by value tends to be more efficient overall, since it strongly reduces redundant evaluations. The evaluation order is predictable, which makes side effects — modifications to state outside the function's local environment — easier to manage.

---

**Sources**

1. David Walker — *Call-by-name, call-by-value and lazy evaluation*
2. Lambda Calculus — University of Colorado
3. Brian Ambielli — *Call by value vs call by name*
