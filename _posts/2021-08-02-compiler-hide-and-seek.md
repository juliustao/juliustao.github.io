---
title: "Hide and Seek with the Compiler"
published: true
---

# Surface

My embedded ARM processor calls a function per frame of a video stream.

- This function accepts a large dataset as input and outputs a small vector, and it takes 32 ms per call.
- That might sound fast, but this function eats up 1/3 of total time per frame.
- So each frame takes 100 ms, which is **10 FPS**.
- That's a sad video.

The goal is optimizing this function for speed.
How would I benchmark each optimization trick I try?

I first set the RNG seed to generate a dataset for each of 1000 trials.

Do I benchmark offline or online?

- Offline
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

- Online
  - For each optimization trick
    - 1000 trials
      - Generate dataset
      - Time call to optimized function
      - Time call to original function
    - Record average times
    - Compute ratio of optimized to original average time

In offline, I generate datasets and run the original once beforehand, so testing times are quicker.
However, I must assume the test environment is controlled, or that I can trust an absolute time from last week instead of retiming.

However, I cannot assume a controlled environment.
Also, I develop off-device on a beefier machine, where 1000 test trials takes under 1 second.

Therefore, I choose online.

For a month, I use online benchmarks and optimize the time *ratio*.
I optimize by switching from `double` to `float` with 0.2% accuracy loss and **40%** speedup relative to the original.

It sounds too good to be true.
And after looking at absolute times, I saw the optimized has no speedup. The original slows down!

# Digging

Here's the high-level code run each trial.
For each of 1000 trials, I run `trial(false, dataset)` and `trial(true, dataset)`, sum up the total times, and compute the ratio of optimized to original.

```cpp
SummaryStruct trial(bool is_optimized, DatasetStruct* dataset) {
  // construct problem
  ProblemStruct* problem = construct_problem(dataset);

  // run function and return summary stats
  SummaryStruct summary;
  if (is_optimized) {
    summary = run_optimized(problem);
  } else {
    summary = run_original(problem);
  }

  return summary;
}
```

How could changing `run_optimized()` to use single-precision floats change the speed of `run_original()`?
I define the functions in separate files, so something strange is happening in `trial()`.

Let's experiment with running just `trial(false, dataset)` and `run_original()` by extension.
When I switch from `double` to `float` in `run_optimized()`, I see the same slowdown in `run_original()`.

Huh?

Maybe it's a cache miss issue?
`cachegrind` shows similar cache miss rates for both `run_original()` with `double` and `float` in `run_optimized()`.
However, the instruction counts for `float` were much higher.

Wait, is it the compiler?
