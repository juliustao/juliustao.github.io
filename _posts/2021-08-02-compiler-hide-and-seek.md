---
title: "Compiler Hide-and-Seek 🔍"
published: true
---

# Overview 🗺️

I have an embedded processor that calls a `C++` function per frame of a video stream.

- The input is a large dataset derived from the frame.
- The output is a small vector.
- Each call takes 32 ms, or 30 FPS.
- However, this function eats up 1/3 of total time per frame.
- Then each frame takes 100 ms, or **10 FPS**.
- That's a sad video.

We need to optimize this function!

## Benching

How can I benchmark each optimization trick I try?

I first set the RNG seed to generate a fixed dataset for each of 1000 trials.

I can bench the original function offline or online.

### Offline

- Precompute once
  - 1000 trials
    - Generate dataset
    - Time call to original function
  - Record average time
- For each optimization trick
  - 1000 trials
    - Generate dataset
    - Time call to optimized function
  - Record average time
  - Compute ratio of optimized to original average time

### Online

- For each optimization trick
  - 1000 trials
    - Generate dataset
    - Time call to optimized function
    - Time call to original function
  - Record average times
  - Compute ratio of optimized to original average time

### 👍 & 👎

In offline, I can generate datasets and bench the original once beforehand, so tests run faster.

However, I must assume the test environment is controlled, or that I can trust an absolute time from last week instead of retiming.

I cannot assume a controlled environment.

Also, I develop off-device on a beefier machine which runs 1000 test trials in under 1 second.

Thus, I choose to bench online.

## Set & Forget

For a month, I use online benchmarks and optimize the time *ratio*.

One trick I try is switching from `double` to `float`.
Tests show just 0.2% accuracy loss but **40%** speedup relative to the original.

It sounds too good to be true.
After looking at absolute times, I saw the optimized has no speedup.

The original slows down!

# ⛏️ Deeper

Here's the high-level `C++` code I run each trial.

```cpp
SummaryStruct trial(bool is_optimized, DatasetStruct* dataset) {
  // construct problem
  ProblemStruct* problem = construct_problem(dataset);

  // run function and return summary stats
  SummaryStruct summary;
  if (is_optimized) {
    summary = optimized_solve(problem);
  } else {
    summary = original_solve(problem);
  }

  // logging
  log_summary(summary);
  return summary;
}
```

## Bench Steps

- 1000 trials
  - `trial(true,  dataset)  // optimized`
  - `trial(false, dataset)  // original`
  - Add to the total times for both
- Compute the ratio of optimized to original

## Setup

My tests show that changing `optimized_solve()` from `double` to `float` slows down `original_solve()`.

How is that possible?

I define these functions independently, so the bug is likely in `trial()`.

What if I just bench `trial(false, dataset)`? Then I only call `original_solve()`.

Now I switch from `double` to `float` in `optimized_solve()`.

Even though I never call `optimized_solve()`, I see the same slowdown in `original_solve()`.

Huh?

## Cache coherence

`cachegrind` shows similar cache miss rates for both `original_solve()` with `double` and `optimized_solve()` with `float`.

However, the instruction count of the latter are much higher then the former.

**50%** higher.

## Compiler antics

Out of curiosity, I try the following.

I first split `trial()` in two.

```cpp
SummaryStruct optimized_trial(DatasetStruct* dataset) {
  // construct problem
  ProblemStruct* problem = construct_problem(dataset);

  // run function and return summary stats
  SummaryStruct summary = optimized_solve(problem);

  // logging
  log_summary(summary);
  return summary;
}

SummaryStruct original_trial(DatasetStruct* dataset) {
  // construct problem
  ProblemStruct* problem = construct_problem(dataset);

  // run function and return summary stats
  SummaryStruct summary = original_solve(problem);

  // logging
  log_summary(summary);
  return summary;
}
```

### Bench Procedure

- 1000 trials
  - `original_trial(dataset)  // original`
  - Add to the total time
- Compute average time
- Record the absolute time

### Setup

Since I only bench `original_trial()`, I only call `original_solve()`.

Now I switch from `double` to `float` in `optimized_solve()`.

As expected, I see the same slowdown in `original_solve()`.

Now I simply comment out `optimized_trial()` before I bench `original_trial()`.

Oddly enough, I see no slowdown in `original_trial()`!

### Theories 🤔

I believe the compiler combines `original_solve()` and `optimized_solve()`.

When I switch `optimized_solve()` from `double` to `float`, `original_solve()` is still `double`.

Thus, I can see how `original_solve()` would be slower and have higher instruction count.

There probably are a ridiculous number of casts from `float` to `double` and back.

How do I prevent the compiler from merging these functions?

# Hide-and-Seek 🔍

My hypothesis is that the compiler only merges functions in the same file.

`optimized_solve()` and `original_solve()` both live in `solve.cpp`.

What if I break the two functions into separate files `optimized_solve.cpp` and `original_solve.cpp`?

Now with `optimized_solve()` using `float`, I see that `original_trial()` runs no slower.

Nice! The fix seems too simple to be true, but I'll take it.

## Abstraction

Of course, I'm not done yet.

I have two functions with duplicate code that I should abstract out.

## 🍔 Analogy

I like analogies.

`optimized_trial()` and `original_trial()` have identical code in the top and bottom but differ in the middle.

Identical code is the bread, and differing code is the meat.

## 🥩 Subclasses

I don't like having two `trial()` functions.

Can I still use one `trial()` function without the compiler noticing?

My idea is to abstract `optimized_solve()` and `original_solve()` as object methods `solve()` of subclasses.

I must access data across function calls, and classes package data nicely with one pointer.

Tests show that using `float` in optimized `solve()` once again slows down `trial(false, dataset)`.

The compiler sees past my ruse!

## 🍞 Utils

Maybe I need separate `trial()` functions to trick the compiler.

Can I instead abstract the bread?

My solution is to move construction and logging into shared `util` methods.

Tests show that using `float` in `optimized_solve()` does not slow down `original_trial()`.

Thank goodness!

# Next 👣

My work in optimization is often more empirical than exact.

I brainstorm many tricks to reduce computations, but I only know if they work after I implement and benchmark them.

I especially treat the `C++` compiler as a blackbox.

Taking a class like *6.035 Computer Language Engineering* seems a fun way to look inside.

Of course, compilers in practice are way more complicated, but 6.035 should be a great start.
