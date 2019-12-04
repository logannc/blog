+++
slug="aoc-2019-2"
title = "Advent of Code 2019: Day 2"
description = "Immediately wrong."
date = 2019-12-02
in_search_index = true
render = true
+++

### Day 2 Problem:

#### Part One:

The problem is that we have to build a small tape-reader-like CPU emulator. It really only has two instructions (add and multiply) and 'halt', so this is pretty easy to do.

The comma deliminated list we read in is the initialization of our tape. We start at index 0. The structure is `opcode, argument address, argument address, result destination address`. This is very easy to do, but it looks particularly nice in Rust.

#### Rust

First of all, I was immediately wrong. My lovely newline-delimited helper is completely useless because this time we're given a comma deliminated list.

I'd have liked to have made generic version that works on both... and it turns out you can! I just didn't notice at the time!

[std::trait::Pattern](https://doc.rust-lang.org/std/str/pattern/trait.Pattern.html) is a *trait bound* on `String::split<P: Pattern>(P)`. I had previously assumed it was just a type alias on another string or something. So, while `&str` implements `Pattern`, `FnMut(char) -> bool` *also* implements this trait. 

So, actually the only thing I'll need to do to adapt my version is change my pattern from being `&str` to being `P: Pattern`. Something for another day... Anyway, on to the problem!

```rust
fn emulate_computer(tape: &mut Vec<u32>) {
    let mut idx = 0;
    loop {
        let opcode = tape[idx];
        match opcode {
            1 => {
                let first = tape[idx + 1];
                let second = tape[idx + 2];
                let location = tape[idx + 3];
                tape[location as usize] = tape[first as usize] + tape[second as usize];
            }
            2 => {
                let first = tape[idx + 1];
                let second = tape[idx + 2];
                let location = tape[idx + 3];
                tape[location as usize] = tape[first as usize] * tape[second as usize];
            }
            99 => break,
            _ => panic!("got bad opcode"),
        }
        idx += 4;
    }
}
```

In Part One, we are told to initialize the first two arguments to some (given) magic numbers and let it run to completion, with the programs 'result' being stored back in index 0, which is our answer.

```rust
pub fn part_one() -> Result<u32, Error> {
    let input_path = problem_input_path(2, None);
    let mut tape = read_file_split_on(&input_path, ",")?; // <- (|c| c == ',') would have also worked!
    tape[1] = 12;
    tape[2] = 2;
    emulate_computer(&mut tape);
    Ok(tape[0])
}
```

I am curious if more junior programmers/developers/hobbyists who have not been exposed to the idea of tape-machines would have more difficulty on this problem. It seems simple enough, but maybe thats only because I have the proper lens to frame it?

#### Part Two:

For Part Two, there is a lot of explanation. More than seems needed. I almost worry that I misunderstood the problem or assumed it was more difficult than it was?

Basically, they want to know which two magic input numbers result in an output of 19690720.

Now, I didn't very closely analyze the input file to see if we can come up with some clever algorithmic steps to not have to simply brute force it... so I just brute forced it. It's very quick, we're already told that the numbers are between 0 and 99, inclusive. So, instead of considering my algorithm `O(n^2)`, I'm going to consider it `O(10,000 * 1)`. :)

#### Rust

```rust
pub fn part_two() -> Result<u32, Error> {
    let input_path = problem_input_path(2, None);
    let orig_tape = read_file_split_on(&input_path, ",")?;
    for noun in 0..=99 {
        for verb in 0..=99 {
            let mut tape = orig_tape.clone();
            tape[1] = noun;
            tape[2] = verb;
            emulate_computer(&mut tape);
            if tape[0] == 19690720 {
                return Ok(100 * noun + verb);
            }
        }
    }
    Err(Error::NoSolutionFound)
}
```

### Summary

All in all, uninteresting problem, though using match statements is always satisfying.