fast-float
==========

[![Build](https://github.com/aldanor/fast-float-rust/workflows/CI/badge.svg)](https://github.com/aldanor/fast-float-rust/actions?query=branch%3Amaster)
[![Latest Version](https://img.shields.io/crates/v/fast-float.svg)](https://crates.io/crates/fast-float)
[![Documentation](https://docs.rs/fast-float/badge.svg)](https://docs.rs/fast-float)
[![Apache 2.0](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![MIT](https://img.shields.io/badge/License-MIT-blue.svg)](https://opensource.org/licenses/MIT)
[![Rustc 1.37+](https://img.shields.io/badge/rustc-1.37+-lightgray.svg)](https://blog.rust-lang.org/2019/08/15/Rust-1.37.0.html)

This crate provides a super-fast decimal number parser from strings into floats.

```toml
[dependencies]
fast-float = "0.1"
```

There are no dependencies and the crate can be used in a no_std context by disabling the "std" feature.

*Compiler support: rustc 1.37+.*

## Usage

There's two top-level functions provided: 
[`parse()`](https://docs.rs/fast-float/latest/fast_float/fn.parse.html) and 
[`parse_partial()`](https://docs.rs/fast-float/latest/fast_float/fn.parse_partial.html), both taking
either a string or a bytes slice and parsing the input into either `f32` or `f64`: 

- `parse()` treats the whole string as a decimal number and returns an error if there are
  invalid characters or if the string is empty.
- `parse_partial()` tries to find the longest substring at the beginning of the given input
  string that can be parsed as a decimal number and, in the case of success, returns the parsed
  value along the number of characters processed; an error is returned if the string doesn't
  start with a decimal number or if it is empty. This function is most useful as a building
  block when constructing more complex parsers, or when parsing streams of data.

Example:

```rust
// Parse the entire string as a decimal number.
let s = "1.23e-02";
let x: f32 = fast_float::parse(s).unwrap();
assert_eq!(x, 0.0123);

// Parse as many characters as possible as a decimal number.
let s = "1.23e-02foo";
let (x, n) = fast_float::parse_partial::<f32, _>(s).unwrap();
assert_eq!(x, 0.0123);
assert_eq!(n, 8);
assert_eq!(&s[n..], "foo");
```

## Details

This crate is a direct port of Daniel Lemire's [`fast_float`](https://github.com/fastfloat/fast_float)
C++ library (valuable discussions with Daniel while porting it helped shape the crate and get it to 
the performance level it's at now), with some Rust-specific tweaks. Please see the original
repository for many useful details regarding the algorithm and the implementation.

The parser is locale-independent. The resulting value is the closest floating-point values (using either 
`f32` or `f64`), using the "round to even" convention for values that would otherwise fall right in-between 
two values. That is, we provide exact parsing according to the IEEE standard. 

Infinity and NaN values can be parsed, along with scientific notation.

Both little-endian and big-endian platforms are equally supported, with extra optimizations enabled
on little-endian architectures.

## Performance

The presented parser seems to beat all of the existing C/C++/Rust float parsers known to us at the
moment by a large margin, in all of the datasets we tested it on so far – see detailed benchmarks 
below (the only exception being the original fast_float C++ library, of course – performance of
which is within noise bounds of this crate). On modern machines, parsing throughput can reach
up to 1GB/s.

In particular, it is faster than Rust standard library's `FromStr::from_str()` by a factor of 2-8x
(larger factor for longer float strings).

While various details regarding the algorithm can be found in the repository for the original
C++ library, here are few brief notes:

- The parser is specialized to work lightning-fast on inputs with at most 19 significant digits
  (which constitutes the so called "fast-path"). We believe that most real-life inputs should
  fall under this category, and we treat longer inputs as "degenerate" edge cases since it
  inevitable causes overflows and loss of precision.
- If the significand happens to be longer than 19 digits, the parser falls back to the "slow path",
  in which case its performance roughly matches that of the top Rust/C++ libraries (and still
  beats them most of the time, although not by a lot).
- On little-endian systems, there's additional optimizations for numbers with more than 8 digits
  after the decimal point.

## Benchmarks

Below is the table of average timings in nanoseconds for parsing a single number 
into a 64-bit float.

|                  | `canada` | `mesh`   | `uniform` | `iidi` | `iei`  | `rec32` |
| ---------------- | -------- | -------- | --------- | ------ | ------ | ------- |
| fast-float       | 22.08    | 11.10    | 20.04     | 40.77  | 26.33  | 29.84   |
| lexical          | 61.63    | 25.10    | 53.77     | 72.33  | 53.39  | 72.40   |
| lexical/lossy    | 61.51    | 25.24    | 54.00     | 71.30  | 52.87  | 71.71   |
| from_str         | 175.07   | 22.58    | 103.00    | 228.78 | 115.76 | 211.13  |
| fast_float (C++) | 22.78    | 10.99    | 20.05     | 41.12  | 27.51  | 30.85   |
| abseil (C++)     | 42.66    | 32.88    | 46.01     | 50.83  | 46.33  | 49.95   |
| netlib (C++)     | 57.53    | 24.86    | 64.72     | 56.63  | 36.20  | 67.29   |
| strtod (C)       | 286.10   | 31.15    | 258.73    | 295.73 | 205.72 | 315.95  |

Parsers:

- `fast-float` - this very crate
- `lexical` – from `lexical_core` crate, v0.7
- `lexical/lossy` - from `lexical_core` crate, v0.7 (lossy parser)
- `from_str` – Rust standard library, `FromStr` trait
- `fast_float (C++)` – original C++ implementation of 'fast-float' method
- `abseil (C++)` – Abseil C++ Common Libraries
- `netlib (C++)` – C++ Network Library
- `strtod (C)` – C standard library

Datasets:

- `canada` – numbers in `canada.txt` file
- `mesh` – numbers in `mesh.txt` file
- `uniform` – uniform random numbers from 0 to 1
- `iidi` – random numbers of format `%d%d.%d`
- `iei` – random numbers of format `%de%d`
- `rec32` – reciprocals of random 32-bit integers

Notes:

- Test environment: macOS 10.14.6, clang 11.0, Rust 1.49, 3.5 GHz i7-4771 Haswell.
- The two test files referred above can be found in 
[this](https://github.com/lemire/simple_fastfloat_benchmark) repository.
- The Rust part of the table (along with a few other benchmarks) can be generated via
  the benchmark tool that can be found under `extras/simple-bench` of this repo.
- The C/C++ part of the table (along with a few other benchmarks and parsers) can be
  generated via a C++ utility that can be found in [this](https://github.com/lemire/simple_fastfloat_benchmark)
  repository.

<br>

#### License

<sup>
Licensed under either of <a href="LICENSE-APACHE">Apache License, Version
2.0</a> or <a href="LICENSE-MIT">MIT license</a> at your option.
</sup>

<br>

<sub>
Unless you explicitly state otherwise, any contribution intentionally submitted
for inclusion in this crate by you, as defined in the Apache-2.0 license, shall
be dual licensed as above, without any additional terms or conditions.
</sub>
