+++
slug="aoc-2019-4"
title = "Advent of Code 2019: Day 4"
description = "Fun Predicate. Boring Structure."
date = 2019-12-04
in_search_index = true
render = true
+++

### Day 4 Problem:

#### Part One:

This problem is actually really similar to the [Day 2 problem](./blog/aoc_2019_2/index.md). It feels like we should be exploiting something, but I feel like the kind of combinatorics tricks necessary to reduce our search space would end up just giving us the answer (being how many valid matches there are) instead of, well, just making it easier to program.

The problem is to find the number of candidate codes that match a predicate:

1. it's within the given range
2. it has at least one pair of equal adjacent numbers
3. the digits in each candidate are monotonically non-decreasing (from left to right).

#### Rust

```rust
fn is_valid_part_one(candidate: u32) -> bool {
    let digits: Vec<u32> = candidate
        .to_string()
        .chars()
        .flat_map(|c| c.to_digit(10))
        .collect();
    // has any two same digits adjacent
    let (has_adjacent, _) = digits
        .iter()
        .skip(1)
        .fold((false, digits[0]), |(found, prev), curr| {
            (found || prev == *curr, *curr)
        });
    // left-to-right monotonic non-decreasing
    let (monotonic, _) = digits
        .iter()
        .skip(1)
        .fold((true, digits[0]), |(nondecreasing, prev), curr| {
            (nondecreasing && prev <= *curr, *curr)
        });
    has_adjacent && monotonic
}

pub fn part_one() -> Result<u32, Error> {
    let input_path = problem_input_path(4, None);
    let range = read_file_split_on(&input_path, "-")?;
    let (min, max) = (range[0], range[1]);
    let mut count = 0;
    for candidate in min..max {
        if is_valid_part_one(candidate) {
            count += 1;
        }
    }
    Ok(count)
}
```

I wish it was a bit easier to destructure arrays/slices/vectors. You can do it with `if let [...] = foo {}`, but I'm typically only destructuring when I *know the structure*. With this kind of syntax, your happy path ends up nested a level deeper rather than `if not value { abort}; do something with value`.

Also, I probably lost some performance doing this functionally here. Barring a sufficiently smart compiler, I can't abort halfway through iterating on the digits once - and I have to do it for two conditions. It actually runs rather slow in Debug mode. It's fine in Release mode. I haven't checked Godbolt or anything to see what changed.

#### Part Two:

In Part Two, the only difference is that it must have a least one set of adjacent equal numbers of cardinality 2. Specifically, equal number pairs may not be part of larger sets of adjacent equal numbers. i.e., 122234 worked for Part One but doesnt now because the 2's are in a set of size 3.

We can't really re-use much from Part One except for our understanding of the problem.

#### Rust

```rust
fn is_valid_part_two(candidate: u32) -> bool {
    let digits: Vec<u32> = candidate
        .to_string()
        .chars()
        .flat_map(|c| c.to_digit(10))
        .collect();
    // has any two same digits adjacent
    let (found, last_run_length, _) =
        digits
            .iter()
            .skip(1)
            .fold((false, 1, digits[0]), |(found, run_length, prev), curr| {
                let continues_run = prev == *curr;
                (
                    found || (run_length == 2 && !continues_run),
                    if continues_run { run_length + 1 } else { 1 },
                    *curr,
                )
            });
    let has_adjacent = found || last_run_length == 2;
    // left-to-right monotonic non-decreasing
    let (monotonic, _) = digits
        .iter()
        .skip(1)
        .fold((true, digits[0]), |(nondecreasing, prev), curr| {
            (nondecreasing && prev <= *curr, *curr)
        });
    has_adjacent && monotonic
}

pub fn part_two() -> Result<u32, Error> {
    let input_path = problem_input_path(4, None);
    let range = read_file_split_on(&input_path, "-")?;
    let (min, max) = (range[0], range[1]);
    let mut count = 0;
    for candidate in min..max {
        if is_valid_part_two(candidate) {
            count += 1;
        }
    }
    Ok(count)
}
```

The only complex part here is making sure the run length is appropriate. We also have to check one last time at the end because the runs terminate on the number after - so we need to make sure runs that end on the last number are checked.

### Summary

Again, a fun but not a super interesting problem. At least in some of the others, we were building off of structure we built in Part One. Also, being fancy and using functional programming probably hurt us here in the benchmark game, which is unfortunate. There is probably a functional programming technique that handles early termination on folding. Once I get around to the Haskell implementation, I might find it, though it won't be necessary to finish any implementation.