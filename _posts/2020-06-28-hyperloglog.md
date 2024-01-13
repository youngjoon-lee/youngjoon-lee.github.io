---
layout: post
title: "HyperLogLog"
date:   2020-06-28 14:53:42 +09:00
---

Given a stream of strings, let's think how to count the number of distinct strings (Cardinality).


## Using in-memory data structure

The simplest way might be using a in-memory data structure, such as HashSet. Whenever we read a string from the stream, we can add it to the HashSet. Finally, we can return the final size of HashSet.

But, this way needs `O(n)` of memory at most, if `n` is the number of strings in the stream. We need a smarter way. It might be so hard to implement a fast algorithm which count the exact number of distinct strings. Let's think about the probabilistic counting.


## Probabilistic counting

Let's use a hash function. Then, we can convert strings into randomly distributed numbers in a certain range. And, let's take a look at the least significant `k` bits in the numbers (hash values). The figure below illustrates an example of the probability of observing a sequence of three consecutive `0`s (`k = 3`).
```
            ....... 0 0 0 ---> prob = 1/(2^3) = 1/8
            ....... 0 0 1
            ....... 0 1 0
 hash(x) -> ....... 0 1 1
            ....... 1 0 0
            ....... 1 0 1
            ....... 1 1 0
            ....... 1 1 1
```

**In other words, on average, a sequence of `k` consecutive `0`s will occur once in every `2^k` distinct entries.** To estimate the number of distinct elements using this pattern, all we need to do is just record the length of the longest sequence of consecutive `0`s. Mathematically, if we denote `p(x)` as the number of consecutive `0`s in `hash(x)`, the cardinality of the set `{x1, x2, ..., xm}` is `2^R`, where `R = max(p(x1), p(x2), ..., p(xm))`

But, there are two disadvantages:
1. The estimate is too sparse. The `2^R` can only be one of `{1, 2, 4, 8, 16, ..., 1024, 2048, ...}`.
2. The estimator has high variability. Because it's recoding the maximum `p(x)`, it requires only one entry whose hash value has too many consecutive `0`s to produce a drastically inaccurate (overestimated) estimate of cardinality.

On the plus side, the estimator has a very small memory footprint. We only need a memory space to record the maximum number of consecutive `0`s. 


## Improving accuracy: LogLog

We can use many estimators instead of one, and average the results, in order to improve accuracy. It means we can use `m` independent hash functions: `{h1(x), h2(x), ..., hm(x)}`. Then, `2^R = 2 ^ (1/m * (R1+...+Rm))`, where the corresponding maximum number of consecutive `0`s for each one: `R1, ..., Rm`.

However, this requires a lot of computations for `m` hash values for each input string. The workaround proposed by [Durand and Flajolet](http://www.ic.unicamp.br/~celio/peer2peer/math/bitmap-algorithms/durand03loglog.pdf) is to use a single hash function, but use part of its output to split values into one of many buckets.

```python
h = hash(x)
bucket_no = k_msb(h, k)   # Most Significant k-Bits
num_zeros = count_zeros_from_right(h)

bucket[bucket_no] = max(bucket[bucket_no], num_zeros)
```

By having `m` buckets, we are bascially simulating a situation in which we had `m` different hash functions. Finally, the formula below is used to get an estimate on the count of distinct values using the `m` bucket values: `{R1, R2, ..., Rm}`.
```
CARDINALITY = constant * m * 2 ^ (1/m * (R1+...+Rm))
```
Durand-Flajolet derived the `constant = 0.79402`. For `m` buckets, this reduces the standard error to about `1.3 * sqrt(m)`. Thus, with 2048 buckets where each bucket is 5 bits (which can record a maximum of 32 consecutive `0`s), we can expect an average error of about 2.8%; 5 bits per bucket is enough to estimate cardinalities up to `2 ^ 27` per the original paper and required only `2048 * 5 = 1.2 KB` of memory.


## Improving accuracy even further: HyperLogLog

1. Durand-Flajolet observed that outliers greatly decreases the accuracy. Thus, we can throw out the largest values before averaging. Specifically, when collecting the bucket values, accuracy can be improved from `1.3 * sqrt(m)` to only `1.05 * sqrt(m)` by only retaining the 70% smallest values and discarding the rest for averaging (aka. `SuperLogLog`).
2. For HyperLogLog, use the harmonic mean instead of the geometric mean, which edging down the error to slightly less than `1.04 * sqrt(m)` with no increase in required storages.

```
CARDINALITY = constant * m * m / (2^(-R1) + ... + 2^(-Rm))
```

## References

https://engineering.fb.com/data-infrastructure/hyperloglog/
