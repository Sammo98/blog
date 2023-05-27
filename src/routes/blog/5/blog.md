### Preface

Now I wouldn't say I was necessary lazy by nature, however, I am lazy when it comes to keeping my directories clean and organised. Granted, I could just go through and manually sort everything, but that's boring, so I'm going to overengineer(?) my issues instead.

This project/application was actually something I wanted to attempt when I was initially learning Python (my first language) but it was beyond me at that point and I couldn't really understand what was going on using the [watchdog](https://pypi.org/project/watchdog/) library.

I had intended to take this time to go slightly lower level and look into interacting with the filesystem API itself - delving into the world of `unsafe`. However I found there is an awesone Rust crate called [notify-rs](https://github.com/notify-rs/notify) which implements exactly what I needed. There's only so far I'll go in overengineering my silly issues.

So the plan for this project is to build something that can run in the background and watch over my Downloads directory, moving files into sub-directories based on their extension type. E.g. `Downloads/some_file.txt` -> `Downloads/txt/some_file.txt`.

### Stubbing The Project

Before getting bogged down with implementation details, stubbing Rust projects always feels like a good way to start thanks to the strong type system. We're going to need the following:

1. A struct with one field - a target directory to watch.
2. A function to initialise the filewatcher itself.
3. An event handler to coordinate what needs to happen when an event is encountered.
4. A file handler to coordinate what needs to be done to the file on which the event is triggered.
5. A function to create a directory if we do not already have a directory present for that extension.
6. A function to actually move the file to the target directory.

I'm sure I'll realise that something else might be required as I'm writing this, but it's a starting point for now, here's what it looks like:

```rust
pub struct FileWatcher {
    target_dir: String,
}

impl FileWatcher {

    pub fn run(&self) {
        todo!()
    }

    fn handle_event(&self) {
        todo!()
    }

    fn handle_file(&self) {
        todo!()
    }

    fn create_dir_if_not_exists(&self) {
        todo!()
    }

    fn move_file(&self) {
        todo!()
    }

}
```

Granted there are no parameters yet, because I'm not certain yet what I need. Stubbing your project out like this is a great way to introduce test-driven development in Rust, the best part of it being that you get a much quicker feedback loop than making incremental changes, re-compiling, and running your binary.

Setting up a test module in Rust is simple:

```rust

#[cfg(test)]
mod test {
    // Tests go here
}
```

As The Book states, the `#[cfg(test)]` annotation means that Rust will only compile the code within this module when `cargo test` is run.

So what tests will we need? We're gonna start with the core functionality and build upwards towards the completed application as, if you notice, there is sort of a dependency tree of what is required. `run` will require `handle_event`, `handle_event` will require `handle_file`, and `handle_file` will require `create_dir_if_not_exists` & `move_file`. So we can start with the last two as our building blocks and go from there. 

As we're working with moving files and creating directories, we want to find a way to setup some temporary files which will be cleared up when the test has finished, for this I've found the crate [tempfile](https://docs.rs/tempfile/latest/tempfile/) to be the most useful.

We're also going to want (although not strictly necessary, just preference) a helper function to create a `FileWatcher` instance as the behaviour we're checking are methods of the FileWatcher struct. I'm also going to use [anyhow](https://docs.rs/anyhow/latest/anyhow/) for error handling in the tests.

```rust
#[cfg(test)]
mod test {
    use super::FileWatcher;
    use anyhow::Result;
    use std::{fs::File, path::PathBuf};

    fn create_fw_instance(target_dir: &str) -> FileWatcher {
        FileWatcher {
            target_dir: target_dir.into(),
        }
    }

    #[test]
    fn test_move_file() -> Result<()> {

        // Create src file
        let src_dir:TempDir = tempfile::tempdir()?;
        let original_file_location:PathBuf = src_dir.path().join("some_file.txt");
        let _ =  File::create(&original_file_location)?;

        // Setup a target directory to move the file to
        let target_dir:TempDir = tempfile::tempdir()?;
        let expected_file_location:PathBuf = target_dir.path().join("some_file.txt");

        // Create filewatcher instance - we don't care what target_dir
        // is supplied to it as it's not actually doing anything at this point.
        // We just want access to the method
        let fw = create_fw_instance("foo");

        // We created the original file, so it should exist, the expected one should not
        assert_eq!(original_file_location.exists(), true);
        assert_eq!(expected_file_location.exists(), false);

        let _ = fw.move_file();

        // Once move file has run the original file should not exist, the expected file should
        assert_eq!(original_file_location.exists(), false)
        assert_eq!(expected_file_location.exists(), true);


    }
}

```

Okay now obviously this doesn't work as `move_file` is unimplemented, if we run `cargo test` we'll get a quick failure with the message `panicked at 'not yet implemented'`, which makes total sense at least.

Setting up our test for move file has given us a good idea of what our arguments need to be, two instances of a `PathBuf`, the source and the destination. As for the actual logic in the function itself, to move a file in Rust we use `std::fs::rename`, which funnily enough has two arguments, a source and a destination:

```rust
pub fn rename<P: AsRef<Path>, Q: AsRef<Path>>(from: P, to: Q) -> Result<()>
```

With this, we can rewrite our function and update our test to be:

```rust
fn move_file(&self, src_file:&PathBuf, destination:&PathBuf) -> Result<()> {
    rename(src_file, destination)?;
    Ok(())
}

#[test]
fn test_move_file() -> Result<()> {

    let src_dir:TempDir = tempfile::tempdir()?;
    let original_file_location:PathBuf = src_dir.path().join("some_file.txt");
    let _ =  File::create(&original_file_location)?;

    let target_dir:TempDir = tempfile::tempdir()?;
    let expected_file_location:PathBuf = target_dir.path().join("some_file.txt");

    let fw = create_fw_instance("foo");

    assert_eq!(original_file_location.exists(), true);
    assert_eq!(expected_file_location.exists(), false);

    let _ = fw.move_file(original_file_location, &expected_file_location);

    assert_eq!(original_file_location.exists(), false)
    assert_eq!(expected_file_location.exists(), true);
}
```

And low and behold our test passes, great news! I find this way of developing to be particular useful in Rust. Before we start actually writing any code for our application, we define the behaviour we want to see, and then write some tests to test that behaviour for what we expect, subsequently making the code conform to the expectations of the tested behaviour. Let's follow the same process for `create_dir_if_not_exists`:

```rust

#[test]
fn test_create_dir_if_not_exists() -> Result<()> {

    // Create a temp directory
    let dir:TempDir = tempfile::tempdir()?;
    let dir:&Path = dir.path();

    // Create filewatcher instance 
    let fw = create_fw_instance("Foo");

    // Assert txt directory does not exist
    let non_existent_dir = dir.join("txt");
    assert_eq!(non_existent_dir.exists(), false);

    // Run function and assert that the directory has been created
    let _ = fw.create_dir_if_not_exists(&non_existent_dir);
    assert_eq!(non_existent_dir.exists(), true);

    Ok(())
}
```

So this looks like the simple behaviour we want, passing a non-existent directory as `PathBuf` into our function should then create it, we can use `std::fs::create_dir` for this. The function signature for which is:

```rust
pub fn create_dir<P: AsRef<Path>>(path: P) -> Result<()>
```

As expected, it takes some path to a directory and creates its.

Let's update the our function to run this only if that directory does not already exist:

```rust
fn create_dir_if_not_exists(&self, dir: &PathBuf) -> Result<()> {
    match dir.exists() {
        true => Ok(()),
        false => {
            Ok(create_dir(&dir)?)
        }
    }
}
```

And our test succeeds! Now this is the core functionality sorted, time to glue these parts together in the `handle_file` function. Let's start by once again setting up the expected behaviour of this function with some tests:

```rust
    #[test]
    fn test_handle_file_moves() -> Result<()> {
        // Create a file in a temp directory
        let temp_dir:TempDir = tempfile::tempdir()?;
        let src_dir:&Path = temp_dir.path();
        let src_path:PathBuf = src_dir.join("some_file.txt");
        let _ = File::create(&src_path)?;

        // This should be the destination, we want handle file to both create the txt directory
        // and then move some_file.txt to it
        let dest_path = src_dir.join("txt").join("some_file.txt");

        // For this test we need to instantiate the filewatcher with it's target directory
        // as it will need this to construct the path to the sub-directory 
        let fw = create_fw_instance(&src_dir.display().to_string());

        assert_eq!(src_path.exists(), true);
        assert_eq!(dest_path.exists(), false);

        let result = fw.handle_file(&src_path);

        assert!(result.is_ok());
        assert_eq!(src_path.exists(), false);
        assert_eq!(dest_path.exists(), true);
        Ok(())
    }
```
Again at this point the test won't succeed, nor will it run as we are supplying an argument to `handle_file` that we haven't specified. Let's give it a go.

```rust

fn handle_file(&self, file: &PathBuf) -> Result<()> {
    // Grab the file name and extension from the pathbuf
    let file_name:Option<&OsStr> = handle.file_name();
    let ext:Option<&OsStr> = handle.extension();

    // We need both of these to continue, so match on both being Some variants
    match (file_name, ext) {
        (Some(file_name), Some(ext)) => {

            // Construct the PathBuf for the directory which we need to
            // ensure exists and then create if it doesn't
            let dir = Path::new(&self.target_dir).join(ext);
            self.create_dir_if_not_exists(&dir)?;

            // construct destination PathBuf and move file
            let destination = dir.join(file_name);
            self.move_file(handle, destination)?;
            Ok(())
        }
        _ => Err(anyhow!(
            "File Name or Extension does not exist for {:?}",
            handle
        )),
    }
}
```

And just like magic, it works. We're one step closer. Now to actually implement notify-rs.

### Event Handling

I lied, before we actually implement notify-rs, we need to look at what happens when the filewatcher encounters an event and define our behaviour for it. At the bottom of the notify-rs [docs](https://docs.rs/notify/latest/notify/) we see the `Event` struct, this seems like what we're looking for. The definition is:

```rust
pub struct Event {
    pub kind: EventKind,
    pub paths: Vec<PathBuf>,
    pub attrs: EventAttributes,
}
```

Where `kind` is the "Kind of type of the event", `paths` being "Paths the event is about...", and `attrs` being "Additional attributes of the event. Arbitrary data may be added to this field".

For our use case, we only care about `kind` and `paths`. The type of `paths` is exactly what we're looking for, a vector of `PathBuf`, although I imagine the length of the vector will always be 1 for new files?

Let's dig into EventKind, here's the definition:

```rust
pub enum EventKind {
    Any,
    Access(AccessKind),
    Create(CreateKind),
    Modify(ModifyKind),
    Remove(RemoveKind),
    Other,
}
```

I won't go into details on variants of the enum other than `Create` as that is the only variant within the scope of what is trying to be done here. We're down to the last turtle, `CreateKind`:

```rust
pub enum CreateKind {
    Any,
    File,
    Folder,
    Other,
}
```
And there we have it. We only care about file creation events, so we need to setup our handle event to handle the `Event` variant `EventKind::Create(CreateKind::File)`.

Once again, tests first:

```rust
#[test]
fn test_handle_event_executes_on_create_file_event() {

    // Set up temp directory and create a file to emulate the mocking of a create file event
    let temp_dir:TempDir = tempfile::tempdir()?;
    let src_dir:&Path = temp_dir.path();
    let src_path:PathBuf = src_dir.join("some_file.txt");
    let _ =  File::create(&src_path)?;

    // Construct the mock event using the file just created above as the Paths
    // Default for attrs as we don't care about it
    let mock_event = Event {
        kind: EventKind::Create(CreateKind::File),
        paths:vec![src_path],
        ..Default::default()
    };

    // Create the file watcher instance and handle the event
    // Check it doesn't error and src file doesn't exist (as has been moved)
    let fw = create_fw_instance(&src_dir.display().to_string());
    let result = fw.handle_event(mock_event);
    assert!(result.is_ok());
    assert_eq!(src_path.exists(), false)
}
```

Now once again, this won't even compile as we're passing an argument to function we've stubbed out with no arguments. We have our logic in place however, the function should accept an `Event` and return an `Result`, giving it a go:

```rust
fn handle_event(&self, event: Event) -> Result<()> {
    match event.kind {
        // Ignore any other events other than create file
        EventKind::Create(CreateKind::File) => {

            // Iterate through the file paths
            for src in event.paths.iter() {

                // Handle the file - our tests already confirm this works as expected
                match self.handle_file(&src) {
                    Ok(_) => println!("File moved successfully!"),
                    Err(e) => println!("Error handling file {src:?}: {e:?}. Skipping ..."),
                }
            }
        }
        _ => println!("Event {:?} encountered. Skipping ...", event.kind)
    }
    Ok(())
}
```

Run the test et voila, it's working!

### The last piece of the puzzle


Now whether this is a cop out or not, I just took the simplest example I could find on the notify-rs GitHub [here](https://github.com/notify-rs/notify/blob/main/examples/monitor_raw.rs). The example looks as follows:

```rust
use notify::{RecommendedWatcher, RecursiveMode, Watcher, Config};
use std::path::Path;

fn main() {
    let path = std::env::args()
        .nth(1)
        .expect("Argument 1 needs to be a path");
    println!("watching {}", path);
    if let Err(e) = watch(path) {
        println!("error: {:?}", e)
    }
}

fn watch<P: AsRef<Path>>(path: P) -> notify::Result<()> {
    let (tx, rx) = std::sync::mpsc::channel();

    // Automatically select the best implementation for your platform.
    // You can also access each implementation directly e.g. INotifyWatcher.
    let mut watcher = RecommendedWatcher::new(tx, Config::default())?;

    // Add a path to be watched. All files and directories at that path and
    // below will be monitored for changes.
    watcher.watch(path.as_ref(), RecursiveMode::Recursive)?;

    for res in rx {
        match res {
            Ok(event) => println!("changed: {:?}", event),
            Err(e) => println!("watch error: {:?}", e),
        }
    }

    Ok(())
}
```

Looks simple enough. Ignoring the CLI part in the main, we set up a multi-producer single-consumer channel, and we pass that in to the `RecommendedWatcher` with default configuration and then consume the events as they come. `RecommendedWatcher`, as the comment states, will select the best implementation depending on the platform being used. OSX, for example, will use the `FsEventWatcher`.

Also important to note the `RecursiveMode` enum which has a very simple definition:

```rust
pub enum RecursiveMode {
    Recursive,
    NonRecursive,
}
```

We only want to watch the individual directory itself, so `NonRecursive` is the one to go for.

Adding this into our struct looks as follows:

```rust
pub fn run(&self) -> notify::Result<()> {

    // Set up the watch directory and the watcher
    let watch_dir = Path::new(&self.target_dir);
    let (tx, rx) = std::sync::mpsc::channel();
    let mut watcher = RecommendedWatcher::new(tx, Config::default())?;
    watcher.watch(watch_dir.as_ref(), RecursiveMode::NonRecursive)?;

    // Handle each event
    // I know there is no real error handling... I added proper logging in the end.
    for res in rx {
        match res {
            Ok(event) => {
                // As per our last test, we've already sorted out the logic of handling events
                // which in turn handles files, which in turn creates directories and moves files
                // ... turtles
                if let Err(_) = self.handle_event(event) {
                }
            }
            Err(_) => {}
        }
    }
    Ok(())
}
```

Okay and we're basically done, last let's chuck the whole thing into `main.rs` and watch the magic happen.

```rust 
fn main() {
    let fw = FileWatcher{target_dir:"some_directory"};
    if let Err(e) = fw.run() {
        println!("Error initiating filewatcher {}, exiting...", e.to_string());
        std::process::exit(1);
}
```
### Quality of life 

The idea of having to pass a path manually into the code, when in fact you might want to change it up easily seems silly. One of my favourite ever crates is [clap](https://github.com/clap-rs/clap), so let's wrap everything up with that.

```rust
// watch.rs
use clap::Parser;

#[derive(Parser)] // Just add this line here
pub struct FileWatcher {
    target_dir: String,
}

// main.rs
use clap::Parser;
pub mod watch;
use watch::FileWatcher;

fn main() {
    let mut fw = FileWatcher::parse(); // And instantiate the FileWatcher like this instead

    if let Err(e) = fw.run() {
        println!("Error initiating filewatcher {}, exiting...", e.to_string());
        std::process::exit(1);
    }
}
```
Yes it really is that simple to turn your struct into a CLI program. Now we can run `cargo run -- <directory_to_watch>`, or install the binary with `cargo install --path .` at the root of our project, and then run the binary itself by `<name_of_binary>  <path_to_watch>`.

Now if you're like me, you may well be sitting there thinking that your desired folder to clean is still an absolute mess, and you would be correct. There needs to be some sort of backloading logic.
We've got everything in place, we simply need to iterate any files in the directory at the start and call `handle_file` on them.

```rust

// watch.rs
use std::fs::read_dir;

pub fn backload(&self) -> Result<()> {
    // Read target directory and backload any missed target files
    let paths = read_dir(&self.target_dir)?;
    for path in paths {
        let path:PathBuf = path?.path();
        if path.is_file() {
            if let Err(_) = self.handle_file(&path) {
            }
        }
    }
    Ok(())
}

// main.rs

fn main() {
    let mut fw = filewatcher::parse();

    if let err(e) = fw.backload() {
        println!("error initiating filewatcher {}, exiting...", e.to_string());
        std::process::exit(1);
    }

    if let err(e) = fw.run() {
        println!("error initiating filewatcher {}, exiting...", e.to_string());
        std::process::exit(1);
    }
}
```

### Final remarks

The practical use of this project is arguably nil, I am fully aware, but it's something I always wanted to create in Python, and Rust gave me an excuse to find the time to do it (as I enjoy writing in Rust).
Rust is an excellent language for test-driven development which I have attempted to showcase here, granted I probably should have written some more tests but it is what is.
