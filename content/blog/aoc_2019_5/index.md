+++
slug="aoc-2019-5"
title = "Advent of Code 2019: Day 5"
description = "Intuitive Foreshadowing"
date = 2019-12-05
in_search_index = true
render = true
+++

### Day 5 Problem:

#### Part One:

Aha! I knew that Day 2: Part Two had entirely too much information! Presumably they did it that way so the information was spread out over multiple parts, though it wasn't needed until this one.

We are told to add two new Op Codes: `Input` and `Output` as well as 'parameter modes'.

I could have used a bit more information on the new opcodes. They're strange, out of band opcodes but it doesn't *say* they're out of band. I was puzzled for a little while about where these 'outputs' were supposed to go. I had originally assumed it was a shortcut for writing into index 0 (because that was how we got our output in Day 2), but because it can be called more than once, that's clearly not sustainable. 

I do hope that other days continue to build on this problem as there is some more I'd like to be able to do but don't need yet.

#### Rust

I'm going to omit the `part_one()` and `part_two()` business today as you've seen it and it isn't interesting. Everything that matters is in the implementation of the tape machine exposed through `emulate_computer()`.

First, let's get some types:

```rust
type Address = usize;
type Value = i32;

enum Parameter {
    Position(Address),
    Immediate(Value),
}

enum OpCode {
    Add(Parameter, Parameter, Address),
    Mul(Parameter, Parameter, Address),
    Input(Address),
    Output(Parameter),
    JumpIfTrue(Parameter, Parameter),
    JumpIfFalse(Parameter, Parameter),
    LessThan(Parameter, Parameter, Address),
    Equal(Parameter, Parameter, Address),
    Halt,
}

fn decode_opcode(tape: &Vec<Value>, ip: Address) -> Result<OpCode, Error> {
    let mut instruction = tape[ip].to_string();
    let split_loc = instruction.len().checked_sub(2).unwrap_or(0);
    let opcode = format!("{:0>2}", instruction.split_off(split_loc));
    let modes: Vec<Address> = instruction
        .chars()
        .flat_map(|c| c.to_digit(10))
        .map(|c| c as usize)
        .rev()
        .collect();
    match opcode.as_ref() {
        "01" => Ok(OpCode::Add(
            param(&tape, ip + 1, modes.get(0))?,
            param(&tape, ip + 2, modes.get(1))?,
            usize::try_from(tape[ip + 3])?,
        )),
        "02" => Ok(OpCode::Mul(
            param(&tape, ip + 1, modes.get(0))?,
            param(&tape, ip + 2, modes.get(1))?,
            usize::try_from(tape[ip + 3])?,
        )),
        "03" => Ok(OpCode::Input(usize::try_from(tape[ip + 1])?)),
        "04" => Ok(OpCode::Output(param(&tape, ip + 1, modes.get(0))?)),
        ... omitted ...
        "99" => Ok(OpCode::Halt),
        _ => Err(Error::BadOpcode(instruction)),
    }
}

pub fn emulate_computer(tape: &mut Vec<Value>, inputs: &Vec<Value>) -> Result<Vec<Value>, Error> {
    let mut inputs = inputs.into_iter();
    let mut outputs = Vec::new();
    let mut ip = 0;
    loop {
        let opcode = decode_opcode(&tape, ip)?;
        match opcode {
            OpCode::Add(p1, p2, a) => {
                tape[a] = p1.get_value(&tape) + p2.get_value(&tape);
                ip += 4;
            }
            OpCode::Mul(p1, p2, a) => {
                tape[a] = p1.get_value(&tape) * p2.get_value(&tape);
                ip += 4;
            }
            OpCode::Input(a) => {
                tape[a] = *inputs.next().unwrap();
                ip += 2;
            }
            OpCode::Output(p1) => {
                outputs.push(p1.get_value(&tape));
                ip += 2;
            }
            ... omitted ...
            OpCode::Halt => break,
        }
    }
    Ok(outputs)
}
```

It feels a little redundant to have both match statements. We could have moved the implementation of the execution of the opcodes and the parsing of them into the same match branches. But I like the separation of concerns of parsing and execution. If we return to this, I'd like to move the computer's state into a struct and have `emulate_computer()` almost look like

```rust
loop {
    let opcode = decode_opcode();
    computer_state.execute(opcode);
    // or even this, though I'm not sure it's better.
    opcode.applyto(computer_state);
}
```

Besides originally breaking on the split location when decoding because I failed to use `checked_sub` and needing to pad the opcodes for the match... it worked on the first try. 

#### Part Two:

Part two is the same, we're just adding a few more opcodes: `JumpIfTrue`, `JumpIfFalse`, `LessThan`, `Equal`.

#### Rust

```rust
enum OpCode {
    Add(Parameter, Parameter, Address),
    Mul(Parameter, Parameter, Address),
    Input(Address),
    Output(Parameter),
    JumpIfTrue(Parameter, Parameter),
    JumpIfFalse(Parameter, Parameter),
    LessThan(Parameter, Parameter, Address),
    Equal(Parameter, Parameter, Address),
    Halt,
}

... decode match
    "06" => Ok(OpCode::JumpIfFalse(
        param(&tape, ip + 1, modes.get(0))?,
        param(&tape, ip + 2, modes.get(1))?,
    )),
    "07" => Ok(OpCode::LessThan(
        param(&tape, ip + 1, modes.get(0))?,
        param(&tape, ip + 2, modes.get(1))?,
        usize::try_from(tape[ip + 3])?,
    )),
    "08" => Ok(OpCode::Equal(
        param(&tape, ip + 1, modes.get(0))?,
        param(&tape, ip + 2, modes.get(1))?,
        usize::try_from(tape[ip + 3])?,
    )),

... exec match
    OpCode::JumpIfTrue(p1, p2) => {
        if p1.get_value(&tape) != 0 {
            ip = usize::try_from(p2.get_value(&tape))?;
        } else {
            ip += 3
        }
    }
    OpCode::JumpIfFalse(p1, p2) => {
        if p1.get_value(&tape) == 0 {
            ip = usize::try_from(p2.get_value(&tape))?;
        } else {
            ip += 3
        }
    }
    OpCode::LessThan(p1, p2, a) => {
        if p1.get_value(&tape) < p2.get_value(&tape) {
            tape[a] = 1
        } else {
            tape[a] = 0
        }
        ip += 4;
    }
    OpCode::Equal(p1, p2, a) => {
        if p1.get_value(&tape) == p2.get_value(&tape) {
            tape[a] = 1
        } else {
            tape[a] = 0
        }
        ip += 4;
    }
```

Again, pretty simple extensions to what we're already doing. The only bug was a slightly perplexing infinite loop because I forgot the increase the instruction pointer when the `JumpIf*` instructions didn't set it somewhere else.

### Summary

I really enjoyed this problem. I'm really hoping that it continues to build off of this problem so I can go back and do some more refactoring to make things nicely compartmentalized. I'm not really sure *how* this will get extended again in an interesting way - maybe division with exceptions or remainders being put somewhere special? - but this theme of problem is my favorite so far.