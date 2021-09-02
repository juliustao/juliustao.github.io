---
title: "Compiler Hide-and-Seek"
published: true
---

# Setting

I have an embedded processor that calls a `C++` function per frame of a video stream.

- The input is a large dataset derived from the frame.
- The output is a small vector.
- Each call takes 32 ms, which sounds fast to me.
- However, this function eats up 1/3 of total time per frame.
- Then each frame takes 100 ms, or **10 FPS**.
- That's a sad video.

We need to optimize this function for speed.

## Benching

How do I benchmark each optimization trick I try?

I first set the RNG seed to generate a fixed dataset for each of 1000 trials.

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

### Pros & Cons

In offline, I generate datasets and run the original once beforehand, so testing times are quicker.

However, I must assume the test environment is controlled, or that I can trust an absolute time from last week instead of retiming.

I cannot assume a controlled environment.

Also, I develop off-device on a beefier machine which runs 1000 test trials in under 1 second.

Therefore, I choose online.

## Set & Forget

For a month, I use online benchmarks and optimize the time *ratio*.

One optimization I try is switching from `double` to `float`.
Tests show 0.2% accuracy loss and **40%** speedup relative to the original.

It sounds too good to be true.
And after looking at absolute times, I saw the optimized has no speedup.

The original slows down!

# Digging

Here's the high-level `C++` code run each trial.

```cpp
SummaryStruct trial(bool is_optimized, DatasetStruct* dataset) {
  // construct problem
  ProblemStruct* problem = construct_problem(dataset);

  // run function and return summary stats
  SummaryStruct summary;
  if (is_optimized) {
    summary = optimized_run(problem);
  } else {
    summary = original_run(problem);
  }

  // logging
  log_summary(summary);
  return summary;
}
```

## Bench Procedure

- 1000 trials
  - `trial(true,  dataset)  // optimized`
  - `trial(false, dataset)  // original`
  - Add to the total times for both
- Compute the ratio of optimized to original

## Setup

How could changing `optimized_run()` from `double` to `float` affect the speed of `original_run()`?

I define these functions independently, so the bug is likely in `trial()`.

What if I just bench `trial(false, dataset)`? Then I only call `original_run()`.

Now I switch from `double` to `float` in `optimized_run()`.

Even though I never call `optimized_run()`, I see the same slowdown in `original_run()`.

Huh?

## Cache coherence?

No, `cachegrind` shows similar cache miss rates for both `original_run()` with `double` and `float` in `optimized_run()`.

However, the instruction counts of `original_run()` for `float` were much higher.

**50%** higher.

## Compiler antics?

For laughs, I try the following.

I first split `trial()` in two.

```cpp
SummaryStruct optimized_trial(DatasetStruct* dataset) {
  // construct problem
  ProblemStruct* problem = construct_problem(dataset);

  // run function and return summary stats
  SummaryStruct summary = optimized_run(problem);

  // logging
  log_summary(summary);
  return summary;
}

SummaryStruct original_trial(DatasetStruct* dataset) {
  // construct problem
  ProblemStruct* problem = construct_problem(dataset);

  // run function and return summary stats
  SummaryStruct summary = original_run(problem);

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

### Silly Setup

Since I only bench `original_trial()`, I only call `original_run()`.

Now I switch from `double` to `float` in `optimized_run()`.

As expected, I see the same slowdown in `original_run()`.

Now I simply comment out `optimized_trial()` and bench `original_trial()`.

And oddly enough, I see no slowdown in `original_trial()`.

### Legit Findings

I think the compiler combines `original_run()` and `optimized_run()`.

When I switch `optimized_run()` from `double` to `float`, `original_run()` is still `double`.

Thus, I can see why `original_run()` slows down and has higher instruction count.

There probably are a ridiculous number of casts from `float` to `double` and back.

How do I prevent the compiler from merging these functions?

# Hide-and-Seek

My hypothesis is that the compiler only merges functions in the same file.

What if I separate the two functions into different files?

`optimized_run()` and `original_run()` live in `solve.cpp`, and I break them into `optimized_solve.cpp` and `original_solve.cpp`.

Now with `optimized_run` in `float`, I actually see no slowdown in `original_trial()`.

## Abstraction

Of course, I'm not done yet.

The codebase contains two functions with nearly identical code.

How can I abstract?

## Burger

I like the burger analogy.

`optimized_run()` and `original_run()` have identical code in the top and bottom but differ in the middle.

Identical code is the *bread*, and differing code is the *patty*.


