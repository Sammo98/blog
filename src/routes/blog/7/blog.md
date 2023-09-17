### Preface

So this is something I've been wanting to do for a while now, and actually have attempted and failed (in Rust) due to a lack of knowledge with respect to dealing with multiple tasks concurrently.

The idea is fairly simple, I want a TUI-based metronome with some kind of an interative interface to allow users to control the behaviour of the metronome. On the surface I thought this initially sounded very easy, but shortly realised it was slighty more complex than I thought.

The issue is that the interactive part of the program will need to be the main thread and I need the program to block on that. I.e. The main purpose of the program is to receive user input, and use that input to do something, in this case to control the metronome.

Ignoring the actual metronome as for now, I decided to opt for sorting out the parsing of the user input first.

### Defining our Input Types

Dealing with potentially complex user inputs means we need a fairly complex parser. I've always wanted to give the crate [nom](https://crates.io/crates/nom) a go, as I've seen this used frequently in various other projects I've come across. My first question with `nom` was basically, okay what is this crate actually for. It's been deemed as a "combinatorial parser", which my simple mind interprets as a crate for combining various parsers? Let's go with that.

I've been looking for an excuse to delve into this crate for a long time now, and thought this was finally a good choice for it. In past I'd likely have opted for a large number of poorly written regexes (hot take, I love writing regex, also is "regexes" how you write that?), but I really like the way `nom` interacts with Rust's enums, more on this later.

As per my post on the [Filewatcher](/blog/5), Rust is perfect for test-driven-developmenet, and Nom even more so. Let's add nom to our `cargo.toml` with `cargo add nom` and define our states and structure for the user input and then write some tests. I'm going to call my module for this `parse.rs`.

My parse module wants to be able to parse user input into an ***exhaustive*** list of operations that will be actioned on the metronome. Did I just write "exhaustive list" ? What I meant to say was Enum.

Rust enums are perfect for defining an exhaustive list of states due to the forced handling of all variants when using `match`. Some slight design needs to be done now, what are the user input actions I want the metronome to be able to support?

1. Start/Stop
3. Change Tempo
4. Toggle Downbeat
5. Change Time Signature
6. Show Help Menu
7. Quit

This seems like a reasonable start, let's define the enum:


```rust
pub enum InputType {
    StartStop,
    TempoChange,
    DownbeatToggle,
    TimeSignatureChange,
    Help,
    Quit,
}
```

The only variants of this enum that will actually require any data attached to them are `TempoChange` and `TimeSignatureChange` as the user will need to specify what tempo or time signature(s) they want.
`TempoChange` is easy to specify a datatype for, it must be positive and doesn't need more than 16 bits, so `u16` it is.

The datatype for `TimeSignatureChange`, is slightly more complex as I want to support complex combinations of time signatures, such as alternating 4/4 3/4, or even 4/4 3/8 alternating. If I'm looking to accept non-quarter-notes as input (signified by the second number not being a 4), then I'll need to keep the data for both the numerator and denominator of the time signature, e.g. 4 & 4 for 4/4.

I could use a struct for this, but I'm actually going to use `Vec<(u8, u8)>` for reasons that will become obvious later on. E.g. For the above example of 4/4 3/8 alternating, I want to parse this into `vec![(4_u8, 4_u8), (3_u8, 8_u8)]`.

With these data types in mind, alongside a stubbed method for doing the actual parsing, the enum becomes:

```rust
pub enum InputType {
    StartStop,
    TempoChange(u16),
    DownbeatToggle,
    TimeSignatureChange(Vec<(u8, u8)>),
    Help,
    Quit,
}

impl InputType {
    pub fn parse(input: &str) -> Self {
        todo!()
    }
}
```

For some initial tests, I just want to test parsing one for each of the types. As the main thing we're testing is the parsing to the correct variants of the enum, I'm also going to add a helper method for checking this:

```rust

#[cfg(test)]
mod test {
    use super::*;

    fn assert_variant(actual: &InputType, expected: &InputType) {
        assert_eq!(
            std::mem::discriminant(actual),
            std::mem::discriminant(expected)
        );
    }

    #[test]
    fn test_parse_help(){
    }

    #[test] 
    fn test_parse_quit(){
    }

    #[test]
    fn test_parse_downbeat_toggle() {
    }

    #[test]
    fn test_parse_time_signature() {
    }

    #[test]
    fn test_parse_bpm() {
    }

    #[test]
    fn test_parse_start_stop() {
    }
```

### Nom

Now to actually get into using Nom. As I read on one of my favourite blogs, [fasterthanli.me](www.fasterthanli.me), the trick to nom is starting small, so lets start with a simple parser for parsing an input for quitting the application.

For now, I'm thinking that any of `q`, `quit`,`exit`, or `:q` (for my fellow vim users) should quit the application. But before considering multiple, let's start with just `q`.

```rust
use nom::IResult;
use nom::bytes::complete::tag;

fn parse_quit(i: &str) -> IResult<&str, &str> {
    tag("q")(i)
}
```

Now going over this code let's look at what is actually happening. The `tag` simply "recognizes a pattern" and "The input data will be compared to the tag combinator’s argument and will return the part of the input that matches the argument". Perfect.

Now although this is parsing a `q`, it isn't actually returning what we want it to return. If the `tag` is successful in identifying a `"q"` we then want to map whatever the `IResult` is into `InputType::Quit`.
Thankfully Nom has a specific mapping function for this, which works similarly to `map` that you will likely be familiar with when working with Iterators. `map` takes two arguments, the first being a parser, and the second being something that implements `FnMut`.

We can update our function as follows:

```rust
use nom::combinator::map;

fn parse_quit(i: &str) -> IResult<&str, InputType> {
    map(
        tag("q"),
        |_:&str| InputType::Quit
    )(i)
}
```

Note that the return type has changed to `InputType`, the above code is mapping the closure to the result (in the event of success) to the remainder of whatever has been tagged by `"q"`.
Once we have indentified the q, we bind to the remainder of the input in the closure with the underscore and just return `InputType::Quit`.

Now to flesh out the test:

```rust
#[test] 
fn test_parse_quit(){
    let actual = parse_quit("q").unwrap().1;
    let expected = InputType::Quit;
    assert_variant(&actual, &expected);
}
```

The test passes as expected, however there is an issue. What happens if the input is not just `'q'`, but something like `"qwerty"`? The test also passes. This is because we are successfully tagging with `'q'` and then presuming that `InputType::Quit` is the input and going from there.

Thankfully there is `nom::combinator::all_consuming` which will make sure the entire input is consumed, using this becomes:


```rust
use nom::combinator::{all_consuming, map};

fn parse_quit(i: &str) -> IResult<&str, InputType> {
    map(
        all_consuming(tag("q")),
        |_:&str| InputType::Quit
    )(i)
}
```

Now for the final part for parsing quit input, which is extending the `all_consuming(tag("q"))` to include the other aforementioned inputs. Essentially we want to execute `all_consuming(tag("q")) || all_consuming(tag("quit"))` and so on, thankfully Nom has our backs here with it's `alt` parser which will attempt to run through multiple parsers and only erroring in the event of all failing.

And so we can extend it as follows:

```rust
fn parse_quit(i: &str) -> IResult<&str, InputType> {
    map(
        alt((
            all_consuming(tag("q")),
            all_consuming(tag("quit")),
            all_consuming(tag("exit")),
            all_consuming(tag(":q")),
        )),
        |_: &str| InputType::Quit,
    )(i)
}
```

And then change our test to go over the valid cases and a couple of invalid ones:

```rust
#[test]
fn test_parse_quit() {
    const EXPECTED: InputType = InputType::Quit;

    for input in ["q", "quit", "exit", ":q"] {
        let actual = parse_quit(input).unwrap().1;
        assert_variant(&actual, &EXPECTED);
    }

    for bad_input in ["qwerty", "foo", ":wq"] {
        let actual = parse_quit(bad_input);
        assert!(actual.is_err());
    }
}
```
And our test passes! Now although this looks like a fairly complex function for just passing a couple of letters which you could do with a very simple regex (`^(q|quit|exit|:q)$`), it's fun (and more performant I believe). Furthermore, it's simple to build up part by part.

Given that we don't actually need to parse any specific data from the input string for `InputType::Quit`, we can use very similar parser functions for our similar input types that just want to convert some text to an enum variant:


```rust
fn parse_help(i: &str) -> IResult<&str, InputType> {
    map(
        alt((
            all_consuming(tag("h")),
            all_consuming(tag("help")),
            all_consuming(tag("?")),
        )),
        |_: &str| InputType::Help,
    )(i)
}

fn parse_downbeat_toggle(i: &str) -> IResult<&str, InputType> {
    map(
        alt((
            all_consuming(tag("db")),
            all_consuming(tag("downbeat")),
        )),
        |_: &str| InputType::DownbeatToggle,
    )(i)
}

fn parse_start_stop(i: &str) -> IResult<&str, InputType> {
    map(
        all_consuming(tag("")),
        |_: &str| InputType::StartStop
    )(i)
}
```

That's four of six input types done! Time for the trickier ones ...

### Utilizing the Tuple Struct Expression

So this took me a while to figure out (although seems obvious in hindsight), and required some help from trusty ol' Reddit, but a tuple struct constructor is a function which implements `FnMut`. If you recall earlier, this is useful when it comes to using `nom::combinator::map` and allows one to do some cool stuff.

For parsing the bpm, I want to allow the following expressions: `bpm <number>` and `tempo <number>`, where as stated above, number should be `u16`. Note that the actual "target" value from the input string will be the number, and this is what we want to use to construct our enum variant `InputType::TempoChange(u16)`. Thankfully there is a nice nom parser `nom::sequence::preceded` which "Matches an object from the first parser and discards it, then gets an object from the second parser."


So we want to use `nom::character::complete::u16` as the second parser in `preceded`, and a `tag` parser in the first parser:

```rust

fn parse_bpm(i: &str) -> IResult<&str, InputType> {
    preceded(tag("bpm "), u16)
}
```

And then comes the magic utilizing the tuple constructor. We want to map the result from that parser, which is a `u16`, to the Tuple constructor, which takes a `u16` argument:

```rust

fn parse_bpm(i: &str) -> IResult<&str, InputType> {
    map(
        preceded(tag("bpm "), u16),
        InputType::TempoChange,
    )(i)
}
```

And there we have it! Instead of using some closure as our `FnMut` we are using the tuple constructor itself, epic.

Only thing now is to allow for `tempo` as well, we can do this by building up our first parser within `preceded`:

```rust
fn parse_bpm(i: &str) -> IResult<&str, InputType> {
    map(
        preceded(alt((tag("bpm "), tag("tempo "))), u16),
        InputType::TempoChange,
    )(i)
}
```
And there we have it. We're going to follow a similar technique for the time signature change. This is the most complex so we're going to build up two custom parsers for this, which will ultimately be used in the parser which converts the output to our enum. The first to parse a single time signature such as `4/4` and the second to use this parser to parse multiple time signatures `4/4 3/4`. Given that our full parser is wanting to return a `Vec` of `(u8, u8)`, it makes sense for our single time signature parser to return `(u8, u8)`.

This is actually relatively simple once you look through the docs, as there is a `separated_pair` parser which "Gets an object from the first parser, then matches an object from the sep_parser and discards it, then gets another object from the second parser." Essentially we provide a 3 parsers, the 2nd of which is ignored, therefore we can use 1. `u16`, 2. `char('/'),` and 3. `u16` to convert a single "4/4" input to `(4_u8, 4_u8)`: 

```rust
fn parse_individual_time_signature(i: &str) -> IResult<&str, (u8, u8)> {
    separated_pair(u8, char('/'), u8)(i)
}
```

Now to use this parser within another parser (so many turtles) to get our `Vec<(u8, u8)>`. The key here is the `separated_list0` parser which "Alternates between two parsers to produce a list of elements." The example given in the [docs](https://docs.rs/nom/latest/nom/multi/fn.separated_list0.html) shows exactly what we want to do here, except that the first parser needs to be our `parse_individual_time_signature` parser, and the second just needs to be `tag(" ")` for the whitespace.

```rust
fn parse_time_signatures_to_vec(i: &str) -> IResult<&str, Vec<(u8, u8)>> {
    separated_list0(tag(" "), parse_individual_time_signature)(i)
}
```

And the final layer, similarly to our `TempoChange` we want to discard the "ts" or "time signature" part of the input string and just get to the actual time signatures themselves by way of the `preceded` parser and then map the value to our enum:


```rust
fn parse_time_signature(i: &str) -> IResult<&str, InputType> {
    map(
        preceded(
            alt((tag("ts "), tag("time signature "))),
            parse_time_signatures_to_vec,
        ),
        InputType::TimeSignatureChange,
    )(i)
}
```

And there we have it, all of our inputs done! Okay not quite we now need to combine all of the above parsers into one mega parser so that we can parse into any of these enum variants from one input. As might be expected at this point, this uses the `alt` parser:

```rust
impl InputType {
    pub fn parse(input: &str) -> IResult<&str, Self> {
        alt((
            parse_time_signature,
            parse_bpm,
            parse_downbeat_toggle,
            parse_help,
            parse_quit,
            parse_start_stop,
        ))(input)
    }
}
```


### Taking User Input

I found this crate through SurrealDB and thought it was nice so decided to use it here, `rustyline`. With a quick `cargo add rustyline --features with-sqllite-history` (allowing us to access command history), we can then set up our main as follows:
```rust
use rustyline::{Config, Editor};

fn main() {

    let config = Config::builder().auto_add_history(true).build();
    let history = rustyline::sqlite_history::SQLiteHistory::with_config(config)
        .expect("Failed to initialise");
    let rl: Editor<(), _> = Editor::with_history(config, history).expect("Failed to initialise");
    loop {
        let line = editor.readline(">>> ").expect("Failed to read line");
        let line_lower = line.to_lowercase();
        let input_type = InputType::parse(line_lower.trim());
        println!("{input_type:?}");
    }
}
```

And now we have something that takes our input and correctly converts it to our enum.

### Final Remarks

This was my first dive into using `nom`, and I gotta say it's pretty awesome. I felt like the initial barrier to entry was that there are so many default parsers it's often hard to find the one you want and you might not even know that what you want exists. Reading through the docs is incredibly useful to get a feel for what can be done with this crate and just how powerful it is.
