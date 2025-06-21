# PRNGLIB
A retro-programming compatible collection of Pseudo Random Number Program Generators (PRNG)

![small logo](https://cldup.com/MWyAWo2qLY.png) 

PRNGLIB is written in C99 using the [Zed Editor](https://zed.dev/) with the [Open Watcom 2](https://open-watcom.github.io/) Compiler and tested with [Doxbox-X](https://dosbox-x.com/) and [EMU86](https://gcallah.github.io/Emu86/index.html).

## Usage:

```c
// Example 1: Basic initialization with default seed, 16 warmup rounds and no logging
    prng_state_t rng;
    prng_init(&rng, PRNG_PCG32, prng_default_seed(), 16, NULL);
    printf("  Float: %.6f, U32: %u\n", prng_next_float(&rng), prng_next_u32(&rng));

// Example 2: Dice rolls, time seed, no warmups, no logging
    prng_state_t rng;
    prng_init(&rng, PRNG_XORSHIFT, prng_time_seed(), 0, NULL);
    printf("Dice roll %u\n", prng_range_u32(prng_next_u32(&rng), 1, 6));

// Example 3: Pick an unbiased card, time seed, no warmups, no logging
    prng_state_t rng;
    prng_init(&rng, PRNG_XORSHIFT, prng_time_seed(), 0, NULL);
    printf("Card choice %u\n", prng_range_exact(&rng, 1, 52));

// Example 4: Seed validation
    uint64_t bad_seed = 0;
    if (!prng_is_valid_seed(bad_seed, PRNG_MARSAGLIA)) {
        printf("Seed 0 is invalid for Marsaglia MWC (as expected)\n");
    }
    
    // Get a valid seed
    uint64_t good_seed = prng_time_seed();
    while (!prng_is_valid_seed(good_seed, PRNG_XORSHIFT)) {
        good_seed = prng_time_seed();  // Should be extremely rare
    }
```
## Seed Logging

PRNGLIB has simple seed logging - important for reproducibility e.g. debugging or experiments:
```c
prng_init(&rng, PRNG_PCG32, 0xABCD1234, 16, logfile);
// Logs: "[PRNG] Engine=pcg32 Seed=0xABCD1234 Timestamp=1712345678"
```

## Performance:

| Algorithm       | Period      | Speed  | TestU01 (BigCrush) | PractRand (2⁶⁴) | Best For                          |
|-----------------|------------|--------|--------------------|------------------|-----------------------------------|
| Marsaglia MWC   | ~2⁶⁰       | 1.0x   | ❌ Fails           | ✅ Passes (~2⁴⁰) | Game physics, Monte Carlo         |
| XORShift128*    | 2¹²⁸−1     | 1.3x   | ✅ Passes          | ✅ Passes        | High-speed simulations            |
| C99 `rand()`    | ~2³¹       | 0.8x   | ❌ Fails           | ❌ Fails (~2³⁵) | Legacy/testing                   |
| PCG32           | 2⁶⁴        | 1.1x   | ✅ Passes          | ✅ Passes        | General-purpose, secure-ish RNG  |
| SplitMix64      | 2⁶⁴        | 1.2x   | ❌ Mild failures   | ✅ Passes        | Parallel streams, seeding        |

- **PractRand**: Tests up to 2⁶⁴ bytes (16 exabytes). A modern standard for statistical flaws.
- **TestU01 (BigCrush)**: The gold standard (~2³⁸ tests). PCG and XORShift* pass; LCGs/SplitMix fail.
- **SmallCrush**: Quick check (~2³⁵ tests). MWC passes; `rand()` fails.

### 1. Marsaglia's MWC (Multiply With Carry)
- **History**: Introduced by George Marsaglia (1990s), MWC combines multiplication and carry propagation for efficiency.
- **Test Results**:
  - ✅ **Passes**: SmallCrush (TestU01), most PractRand tests (up to ~2⁴⁰ bytes)
  - ❌ **Fails**: Some longer PractRand tests (~2⁵⁰+), occasional linear artifacts
- **Pros**: Fast (~1.0x ref), decent statistical quality
- **Cons**: Period (~2⁶⁰) is shorter than modern alternatives
- **Use Case**: Monte Carlo, game physics, general RNG
- **Reference**: Marsaglia, G. (1994). ["Yet another RNG."](https://arxiv.org/abs/math/9605194) arXiv:math/9605194.

### 2. XORShift128*
- **History**: Proposed by Marsaglia (2003), improved with multiplication (Vigna's `*` variant)
- **Test Results**:
  - ✅ **Passes**: PractRand (up to 2⁶⁴), Crush (TestU01)
  - ❌ **Fails**: BigCrush (vanilla XORShift128), but XORShift* passes
- **Pros**: Very fast (1.3x ref), excellent statistical properties
- **Cons**: Fails BigCrush without scrambling (fixed in `**` variants)
- **Use Case**: High-performance simulations, procedural generation
- **Reference**: Vigna, S. (2017). ["Further scramblings of Marsaglia's xorshift generators."](https://arxiv.org/abs/1404.0390) arXiv:1404.0390.

### 3. C99 `rand()` (LCG)
- **History**: Classic Linear Congruential Generator (Lehmer, 1949), used in C/C++ `rand()`
- **Test Results**:
  - ❌ **Fails**: PractRand (after ~2³⁵ bytes), SmallCrush, BigCrush
  - ✅ **Passes**: Only trivial tests
- **Pros**: Simple, widely compatible
- **Cons**: Predictable, fails all rigorous tests
- **Use Case**: Legacy code, benchmarking
- **Reference**: Knuth, D. E. (1997). *The Art of Computer Programming, Vol. 2*

### 4. PCG32 (Permuted Congruential Generator)
- **History**: Developed by Melissa O'Neill (2014) as an LCG upgrade
- **Test Results**:
  - ✅ **Passes**: PractRand (full 2⁶⁴ tested), BigCrush (TestU01)
  - ❌ **Fails**: None in standard configurations
- **Pros**: Excellent quality, configurable, decent speed (1.1x ref)
- **Cons**: Not cryptographic
- **Use Case**: Secure shuffling, simulations, seeding
- **Reference**: O'Neill, M. (2014). ["PCG: A Family of Fast RNGs."](https://www.pcg-random.org) pcg-random.org.

### 5. SplitMix64
- **History**: Designed by Guy L. Steele (2014) for Java 8
- **Test Results**:
  - ✅ **Passes**: PractRand (2⁶⁴), Crush (TestU01)
  - ❌ **Fails**: BigCrush (mild failures in matrix tests)
- **Pros**: Extremely fast (1.2x ref), splittable
- **Cons**: Weaker than PCG/XORShift* in statistical tests
- **Use Case**: Parallel streams, hashing, seeding
- **Reference**: Steele, G. L. et al. (2014). ["Fast splittable PRNGs."](https://doi.org/10.1145/2660193.2660195) ACM OOPSLA.

