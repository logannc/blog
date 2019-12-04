+++
slug="aoc-2019-3"
title = "Advent of Code 2019: Day 3"
description = "What harm can a simple swap do?"
date = 2019-12-03
in_search_index = true
render = true
+++

### Day 3 Problem:

#### Part One:

I hate these kinds of problems. Really anything with geometry is just a bunch of bookkeeping, fundamentally. Get a few layers of abstraction higher, sure, the algebras can be interesting. But *implementing* geometry primitives is a pain in the ass. It's part of why I haven't done much ML or graphics work. Sure I *can* do ML now a days with relatively little manual linear algebra. Unless you're doing cutting edge stuff, most of it's already bundled up. But I really feel like I ought to sink my teeth in on some baby Neural Networks first, implemented by hand. And I just don't want to. It kind of makes me sad, but linear algebra and matrices are so powerful and I love them from a theoretical standpoint.  It's kind of like how I really love group theory - groups, rings, fields and such are such fun concepts! But I don't want to do a bunch of long division!

We are told that we have two wires, specified by a series of cardinal direction vectors (U5 (up 5), R12 (right 12), etc). We are asked to find the intersection closest to the origin (that isn't the origin).

There are multiple things that would have simplified the problem, some of which I assumed and it worked out okay. Others I didn't and others were small optimizations that aren't really necessary on the scale we're dealing with. For example:

1. By construction of the line segments we'll be dealing with, they are all either vertical or horizontal.
2. With that, I assumed we wouldn't be dealing with overlapping lines (except for a single potential coterminous endpoint). There is nothing to support this in the problem statement, I just didn't want to deal with it and it seemed likely from the problem statement. If I didn't make this assumption, I would have had to return `Vec<Point>` instead of `Option<Point>`... which is fine, I guess. It just would have added a lot more logic to an already tedious intersection method. Did I mention I don't like these kinds of problems?
3. I could have split up the problem space to try to avoid (another) `O(mn)` solution (where each term is the number of line segments in each wire). There wasn't an obvious way to do that, however, as the examples were quite... swirly. There wasn't an obvious way to exclude any two segments from intersecting.
4. I also could have taken a grid-based approach instead of a collection of line segments tracing the wire paths. I didn't do this for a couple reasons: a) without manually inspecting the input, it isn't possible a priori to determine a grid size b) with points down and left of the origin, an array backed grid is not simple c) that leaves a hash map backed grid which felt wrong for this application d) and finally, a hash map backed approach would be `O(k)` where `k` is the total path length of both wires. A cursory glance at the input shows that `mn << k`. *THAT SAID*, the constant term for the line segment `O(mn)` approach represents finding the point of intersection of two line segments, which is a lot of branches and comparisons. The constant term for the array backed `O(k)` approach is inserting into the map or reading from the map. The loop can't be trivially unrolled (but is doable), but it still might be more architecturally aligned. Maybe I should go back and benchmark both solutions sometime.

#### Rust

In some sense, this problem played to Rust's strengths. This problem begins to have enough structure (in particular, mathematical structure) that the types become helpful. We start off with some utility types:

```rust
#[derive(Debug, Clone, Copy)]
enum DirectionVector {
    Up(u32),
    Down(u32),
    Left(u32),
    Right(u32),
}

impl DirectionVector {
    fn parse(s: &str) -> Result<Self, Error> {
        Ok(match s.trim().split_at(1) {
            ("U", n) => Self::Up(str::parse::<u32>(n)?),
            ("D", n) => Self::Down(str::parse::<u32>(n)?),
            ("L", n) => Self::Left(str::parse::<u32>(n)?),
            ("R", n) => Self::Right(str::parse::<u32>(n)?),
            _ => return Err(Error::DirectionParseError(s.to_owned())),
        })
    }
}

#[derive(Debug, Default, Clone, Copy, PartialEq, Eq)]
struct Point {
    x: i32,
    y: i32,
}

impl Ord for Point {
    fn cmp(&self, other: &Self) -> Ordering {
        let self_mag = self.from_origin();
        let other_mag = other.from_origin();
        self_mag.cmp(&other_mag)
    }
}

impl PartialOrd for Point {
    fn partial_cmp(&self, other: &Self) -> Option<Ordering> {
        Some(self.cmp(other))
    }
}

impl Point {
    fn from_origin(&self) -> u32 {
        (self.x.abs() + self.y.abs()) as u32
    }
    fn travel(&self, to: DirectionVector) -> Point {
        match to {
            DirectionVector::Up(n) => Point {
                x: self.x,
                y: self.y + n as i32,
            },
            DirectionVector::Down(n) => Point {
                x: self.x,
                y: self.y - n as i32,
            },
            DirectionVector::Left(n) => Point {
                x: self.x - n as i32,
                y: self.y,
            },
            DirectionVector::Right(n) => Point {
                x: self.x + n as i32,
                y: self.y,
            },
        }
    }
}
```

These set us up for success, but the meat of the work is done by:

```rust

#[derive(Debug, Clone, Copy)]
struct LineSegment {
    start: Point,
    end: Point,
}

impl LineSegment {
    ... omitted ...
}

#[derive(Default, Debug)]
struct SparseLineBoard {
    lines: Vec<LineSegment>,
}

impl SparseLineBoard {
    ... omitted ...
}
```

With those implementations in place, the solution is fairly simple:

```rust
pub fn part_one() -> Result<u32, Error> {
    let (wire_one, wire_two) = get_wires()?;
    let mut line_board = SparseLineBoard::default();
    // This initializes the board with wire_one.
    line_board.bulk_travel(wire_one);
    let mut cursor = Point::default();
    let mut intersections = BTreeSet::new();
    // walk wire_two, checking each segment for intersections
    for vector in wire_two.into_iter() {
        let destination = cursor.travel(vector);
        let line = LineSegment{ start:cursor, end: destination};
        cursor = destination;
        for point in line_board.intersections(&line) {
            // if BTreeSet has an `extend` method, I missed it
            intersections.insert(point);
        }
    }
    intersections.remove(&Point::default());
    // we've defined comparisons on `Point` as distance from origin, so let's grab it
    // BTreeSet iterates in ascending order
    match intersections.iter().next() {
        Some(point) => Ok(point.from_origin()),
        None => Err(Error::NoSolutionFound),
    }
}
```

I've omitted the tedious method implementations on the `LineSegment` and `SparseLindBoard`. They're terribly boring, if you really want to see them, you can look on GitHub.

#### Part Two:

They change it up on us. We now no longer want the nearest intersection, we want the intersection with the shortest total wire path!

This means that we must know how far we have traveled along each path at each intersection point. This turns out to be not too hard to recover from the intersection algorithms. Since we're essentially just looping over an in-order vector of wire segments, we just accumulate our distance traveled and get the partial distance once we find an intersection. We can also short circuit *slightly* as for any given segment on the path we're traveling, we only care about the 'first' intersection.

#### Rust

Again, we've done the bulk of the work already, we just need to set the stage and do a little bit of book keeping.

```rust
pub fn part_two() -> Result<u32, Error> {
    let (wire_one, wire_two) = get_wires()?;
    let mut line_board = SparseLineBoard::default();
    line_board.bulk_travel(wire_one);
    let mut cursor = Point::default();
    let mut traveled = 0;
    let mut intersection_latencies = Vec::new();
    for vector in wire_two.into_iter() {
        let destination = cursor.travel(vector);
        let line = LineSegment{ start: cursor, end: destination };
        if let Some((other_traveled, i)) = line_board.first_intersection(&line) {
            if i != Point::default() {
                let partial = LineSegment{ start: cursor, end: i}.length();
                intersection_latencies.push(other_traveled + traveled + partial);
            }
        }
        cursor = destination;
        traveled += line.length();
    }
    match intersection_latencies.into_iter().min() {
        Some(latency) => Ok(latency),
        None => Err(Error::NoSolutionFound),
    }
}
```

I *will* say that this second part took me entirely longer than it should have. Because `LineSegment{ start: cursor, end: i }` was the only bit that ever really relied on the intersection point, I didn't realize that the point returned by the intersection method accidentally swapped the x and y coordinates! It passed in Part One because the manhattan distance from the origin of (x,y) is the same as (y,x). But, when trying to find the length of a `LineSegment`, that can cause trouble.

### Summary

Overall, it was a good problem. I'm just not a fan of these kind of fiddly problems and it was doubly frustrating because I had a simple, subtle bug that took up a lot of my time.