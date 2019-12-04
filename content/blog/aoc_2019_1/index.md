+++
slug="aoc-2019-1"
title = "Advent of Code 2019: Day 1"
description = "I've never done it before, so why not jump in entirely too hard, all at once. Publicly."
date = 2019-12-01
in_search_index = true
render = true
+++

### Advent of Code

For the uninitiated, the [Advent of Code](https://adventofcode.com/) is a lovely little twist on the normal Christmas Advent put on by Eric Wastl. I'll steal from his description:

> Advent of Code is an Advent calendar of small programming puzzles for a variety of skill sets and skill levels that can be solved in any programming language you like.

A variety of public figures often participate (for example, [Peter Norvig](https://github.com/norvig/pytudes)).

While I've known it was going on and even read through some of Norvig's solutions, I've never personally participated. I've just always been a bit too busy in December - there seemed to be better things to do. This year, however, my December is both regimented and somewhat more open than ever before. So I thought this would be a good year to 1) jump into the AoC both feet first and 2) revitalize my blog that I never really started. 

I mean, come on. My introductory blog post is outdated before it got finished. Ouch. I will likely just post a new entry rather than fix it seeing as how things need to change.

So! In the spirit of jumping back in blindly, with both feet, with my hands tied behind my back... I'm going to be doing the AoC in SIX different languages, all as idiomatic as I have time to make them! I'm going to be doing it in Rust, Python, Go, Haskell, Scheme, and Swift:

Rust - because I am a bit of a Rust fanboy and I'd like to have a little more experience with it than from the sidelines. Even if the AoC isn't particularly fancy.

Python - because it's a delightful, pragmatic, elegant language that I have the most experience in.

Go - because it first captured my attention before Rust and brings lots of lovely things to the table - despite it's glaring lack of certain niceities.

Haskell - because pure functional programming *is cool*. Although, to be honest, if I abandon any language, Haskell will likely be first. It's the biggest pain the butt and I'm woefully unfamiliar with how the ecosystem has evolved. GHC, Cabal, Stack? What? *AND* I'm a newbie on NixOS so figuring out how to properly do development in each language is it's own task.

Scheme - because, like Haskell, Lisp has something fundamental about it, despite being about the opposite of Haskell (pure functions vs. functions that mutate other functions!).

Finally, Swift - because it occupies a similar niche as Rust, has some comparable features, and - most importantly - my wife kind of wants to learn it and I figure this will be as good as anything to introduce myself first. I'm a little intimidated on this one. I'm developing on NixOS so it's a *very* non-standard platform for Swift and I have basically zero experience with Apple development ecosystems.

Like I said, I'll be trying to do things fairly idiomatic... but, well, I don't know the idioms yet and it might take a while to learn for all of these. So while the solutions for each challnge might not be idiomatic by the end of each day, I'll be going back as I learn more to try to clean things up. Further, I'll be focusing on keeping up with the challenges than perfecting all six solutions each day, so there might be holes until I go back and fill them. For example, on Day 1, I only completed Rust and Swift because we were driving back from where we had Thanksgiving! The Rust turned out fairly well, the Swift was hacked together as quickly as possible with my complete lack of Swift experience.

All code mentioned here is from https://github.com/logannc/adventofcode/. It may or may not match exactly as I will not be updating code sections in posts if I go back to clean things up.

Let's take a look.

### Day 1 Problem:

#### Part One:

I'm going to leave off the fluff because, while it's adorable, you can go read it yourself [here](https://adventofcode.com/2019/day/1).

We are given a text file. It is a newline deliminated list of integers representing spacecraft module weights.

We are given a terrible rocket fuel estimation of `max(0, floor(weight / 3) - 2)`.

Rather than adding the weight of all modules together and calculating the fuel required, we calculate the fuel required for each module and then add that together. This is important. It tripped me up later.

It also means that the sum of all small enough components could be large and we'd add no fuel for it. As Part Two will later say, `the remaining mass, if any, is instead handled by wishing really hard, which has no mass and is outside the scope of this calculation`. Love it.

#### Rust

I was riding shotgun in a car and already had Rust installed, so that's how I solved it first. 

I did end up tethering my laptop to my phone in order to install `gcc` as Rust has an uncaptured dependency on `cc` on NixOS. I then also had to install the `Rust (rls)` VSCodium extension and Racer, etc. After these components were installed (zero configuration required), everything else was offline. I had Rust Stable installed via Rustup which means that I had the entire Rust Book and Rust Standard Library documentation available (and more!) . Take a look at `rustup doc -h` to see just how much comes bundled.

#### Rust: Project Structure

The first big hurdle I faced was: how do I structure this? I have a pretty good foundation of the language's syntax, data model, and general language goals and that serves me well once I'm actually implementing something. However, the module system has rightly been criticized as being more difficult than it needs to be. This was improved with the 2018 Edition of Rust (which, as an aside, if you aren't familiar with the concept of Editions, Rust got this *RIGHT* - brief overview [here](https://doc.rust-lang.org/edition-guide/editions/index.html)) by making a bunch of un-needed keywords... actually be no longer needed. Further, some flexibility was added to where files could go, but I haven't really seen much benefit there. In theory, you can have a module at `./foo.rs` instead of `./foo/mod.rs`, but if you want files *within* that module, you still need the latter to re-export the other files. I'm also not entirely sure how the module visibility system works. I have `./utils/errors.rs` and `./utils/files.rs` in a `utils` module but I have to use an absolute reference like `crate::utils::errors::Error` in `files.rs`? I assume I'm missing some things here that I'll come back and clean up.

#### Rust: Error Handling

Finally, before we get into the actual implementation, the last hang up was error handling. Rust's error handling story has an extremly robust base with `Result<T, E>`, `Option<T>` and `?` (the [`Try`](https://doc.rust-lang.org/std/ops/trait.Try.html) operator sigil). However, one thing that quickly becomes clear using Rust is that *there are a lot of different types*. In Rust's effort to be explicit, there is either a lot of redundant code to make the types match or some boiler plate necessary to make transitioning between types easier. This comes up a lot with errors as we often have multiple function calls with `Result<T, E>` return values that may or may not unify with the calling context's return value.

The DIY way to handle ths is for the crate to define it's own `enum Error` (or multiple, if necessary) to wrap around errors for unification purposes. This let's us do things like:

```rust

impl From<std::io::Error> for Error {
    fn from(err: std::io::Error) -> Error {
        Self::IoError(err)
    }
}

fn read_file_split_whitespace(file: &Path) -> Result<Vec<T>, Error> {
    let content = fs::read_to_string(file)?;
    ...
}
```

Because we've defined a wrapper enum instance (not included above) and a way to convert from some other error into that instance ([the `From` trait](https://doc.rust-lang.org/std/convert/trait.From.html)), we can fearlessly unwrap the `Result<String, std::io::Error>` from `fs::read_to_string(...)` and either use the `String` or return an error.

Without the `From` implementation, it might look like:

```rust
fn read_file_split_whitespace(file: &Path) -> Result<Vec<T>, Error> {
    let content = match fs::read_to_string(file) {
        Ok(s) => s,
        Err(e) => { return Error::IoError(e) }
    };
    ...
}
```

The original version is clearly more legible, saves lines if you need this conversion more than once, and concentrates the error conversion logic outside of the happy path so it doesn't distract us.

There *are* crates that implement macros and patterns that make this a little bit less manual (the `From` implementation is utterly repetitive for most implementations for errors). However, there are several different evolutions of crates for this and I haven't gotten a feel for which approach I like yet. I expect to come back to it in a few days and refactor.

#### Rust: Solution

It's only day one, so this isn't going to be anything complicated. Let's just take a peek:

```rust

fn fuel_cost(weight: f32) -> u32 {
    ((weight / 3.0) as u32).checked_sub(2).unwrap_or(0)
}

pub fn part_one() -> Result<u32, Error> {
    let input_path = problem_input_path(1, None);
    let numbers = read_file_split_whitespace(&input_path)?;
    let sum = numbers.into_iter().map(fuel_cost).sum();
    Ok(sum)
}
```

Alright! That looks pretty good! We get our input file, we read it, get the cost for each one, and add it up! Simple!

...

Yea, if you think I'm skipping some steps here, you're right.

Every time I dive back into Rust I'm blown away by one simple fact: *there are so many freakin' types*! You don't see them here because Rust has amazing type-inference powers, but it's very easy to be overwhelmed when you're trying to figure out how to juggle all of the different types. While Rust's enables very powerful, safe patterns due to it's strong type system, the breadth and flexibility of the Rust standard library also means there are many different paths from `Type A` to `Type B`. Finding the shortest and/or most efficient path requires a bit of exposure to get a feel for where to go. That isn't to say that this is always the case in Rust. You can also build State Machine implementations where the type system requires you to take certain paths through types to enforce certain invariants. But, for many common tasks, there are many ways to do it.

Some interesting notes:

1. In some previous implementations, you couldn't move the `?` from `let sum = numbers?...` to the end of the declaration of `numbers` without having to manually specify types (not the end of the world, but less 'clean').
2. The implementation of those utility methods took me *ages*! I'm still not sure that `part.as_ref().map_or(String::new(), u8::to_string)` is the best way to turn an `Option<u8>` into either a string of the number or an empty string.
3. Notice that we don't have a conversion to `f32` anywhere! I'm happy with how this turned out, actually. I anticipate that the puzzle nature of the AoC will require parsing newline deliminated text files quite often and that I might want the contents to be of various types. So `read_file_split_whitespace` is actually polymorphic in it's return type!

```rust

pub fn read_file_split_whitespace<T: std::str::FromStr>(file: &Path) -> Result<Vec<T>, Error>
where
    Error: std::convert::From<<T as std::str::FromStr>::Err>,
{
    let content = fs::read_to_string(file)?;
    let parsed: Result<Vec<T>, _> = content.split_whitespace().map(str::parse::<T>).collect();
    parsed.map_err(|e| e.into())
}
```

Normally, calling a function like this would require something called the ['turbofish'](https://stackoverflow.com/questions/52360464/what-is-the-syntax-instance-methodsomething/52361559). ([More info here](https://matematikaadit.github.io/posts/rust-turbofish.html) and the [surprising etymology here](https://www.reddit.com/r/rust/comments/3fimgp/why_double_colon_rather_that_dot/ctozkd0/).) However, because `fuel_cost` accepts `f32`, we know that we must be mapping over `Iterator<Item=f32>` (kind of - the type jungle around iterators is dense, use `.iter()` for an iterator which borrows values, use `.into_iter()` to consume the source into an 'owned iterator' which iterates over owned values) which must come from a `Vec<f32>` which means that `read_file_split_whitespace` must return `Result<Vec<f32>, Error>`.

The rest of the type shenanigans in this function guarantee we can actually do what we are trying to do. 1) The return type must be able to be created from a `String`, which we guarantee by `T: FromStr`, and 2) if the conversion fails, the `where` clause guarantees that the parsing error has a `From<ParsingError> for Error` implementation so that we have a unified return type.

Overall, this problem was a fun way to jump back into the hang of Rust and explore the common types and methods we'll need to be dealing with. The only thing I think I really would like to add is tests to make sure future changes don't break old solutions. We'll call this a Day 2 task.

#### Part Two:

An obvious extension of the previous part: we need to account for the fuel needed to carry the fuel! The [Tyrrany of the Rocket Equation](https://what-if.xkcd.com/38/) strikes again!

#### Rust: Solution

The solution to part two builds off of the first part. Besides an issue where I was using the answer to part one as the input to my second part (instead of solving individually and then summing), it was pretty straightforward.

```rust
fn iterative_cost(weight: f32) -> u32 {
    let mut cost = fuel_cost(weight);
    let mut additional_cost = fuel_cost(cost as f32);
    while additional_cost > 0 {
        cost += additional_cost;
        additional_cost = fuel_cost(additional_cost as f32);
    }
    cost
}

pub fn part_two() -> Result<u32, Error> {
    let input_path = problem_input_path(1, None);
    let numbers = read_file_split_whitespace(&input_path)?;
    let sum = numbers.into_iter().map(iterative_cost).sum();
    Ok(sum)
}
```

#### Swift: Combined Solution

I told you I hacked the Swift solution together very quickly. It was late, I wanted to go to sleep and I'm not even sure how to import from adjacent swift files yet.

```swift
import Foundation

func fuel_cost(weight: Float) -> Int {
    let result = (weight/3.0) - 2
    if result > 0 {
        return Int(result)
    }
    return 0
}

func recursive_cost(weight: Float) -> Int {
    var cost = fuel_cost(weight: weight)
    var additional = fuel_cost(weight: Float(cost))
    while additional > 0 {
        cost += additional
        additional = fuel_cost(weight: Float(additional))
    }
    return cost
}

let path = "../advent_problems/day01/input"

if let handle = FileHandle.init(forReadingAtPath: path) {
    let inputString = String(data: handle.readDataToEndOfFile(), encoding: String.Encoding.utf8)!
    let numbers = inputString.split(separator: "\n").map({ Float($0)! })
    print(numbers.map(fuel_cost).reduce(0, +), numbers.map(recursive_cost).reduce(0, +))
} else {
    print("oh no!")
}
```

It is, minus infrastructure, a direct port of the Rust solution. Until I get to Haskell, this will likely be true of all the solutions (how do I do the `recursive_cost` implementation in Haskell without iteration - oh wait).

So far, I'm not *particularly* fond of Swift. I dislike how it handles function parameters and I have no idea whats going on with `map({ Float($0)!})`. Day 2 will likely involve some actually learning the language to figure out what I should have done here. And hopefully finding better documentation.