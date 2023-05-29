### Preface

Here's just a quick post for something I've been meaning to test out for a while, and that is the combination of [PyO3](https://github.com/PyO3/pyo3) and [Maturin](https://github.com/PyO3/maturin). PyO3 is Rust crate which provides Python bindings, giving the ability to call Rust code from a Python application. Maturin is what is used to build a local Rust library (using PyO3 bindings) and subsequently use it as a Python package.

Now I'm fully aware that there are countless blog posts which have already done this, but I am yet to find one that targets something that I work with almost everyday, [Flask](https://flask.palletsprojects.com/en/2.3.x/). When building out a web API, performance is naturally an important metric, with optimisation often being a necessary step. Yet with the PyO3 + Maturin combination, simply finding the bottleneck and palming it off to a blazingly-fast-Rust-written counterpart often does the job.

### Setting up the project

Time to start a new Python project. I'm yet to delve in [Poetry](https://python-poetry.org/) (although it is another item on my list of things to look into), so I'm just going to go without for now:

```bash
$ mkdir rusty_flask
$ cd rusty_flask
$ python -m venv .venv && source .venv/bin/activate
$ python -m pip install flask maturin
$ touch main.py
```

And setting up the bare minimum Flask app in `main.py`:

```python
from flask import Flask

app = Flask(__name__)

@app.route("/")
def home():
    return "hi"

if __name__ == "__main__":
    app.run("0", 5000, debug=True)
```

Now for those unfamiliar with Flask, this is a very simple server with a single endpoint which just returns `"hi"`.

Running our application with `python main.py` and sending a quick `curl` to `localhost:5000` and we can confirm the everything is working as intended:

``` bash
$ python main.py
$ curl localhost:5000
hi
```

Cool, that's the python side setup, now to initiate our Rust module by running `maturin init --bindings pyo3` which will do the necessary setup for our Rust module with pyo3 bindings.

There are a couple of main things this generates: (1) `cargo.toml` and (2) `src/lib.rs`. The `cargo.toml` will look something like the following:

```toml
[package]
# Package name comes from directory name
name = "rusty_flask"
version = "0.1.0"
edition = "2021"

[lib]
# This seems to be the default lib name at time of writing
name = "string_sum"
crate-type = ["cdylib"]

[dependencies]
pyo3 = "0.18.3"
```

`src/lib.rs` will look as follows:

```rust
use pyo3::prelude::*;

#[pyfunction]
fn sum_as_string(a: usize, b: usize) -> PyResult<String> {
    Ok((a + b).to_string())
}

#[pymodule]
fn string_sum(_py: Python<'_>, m: &PyModule) -> PyResult<()> {
    m.add_function(wrap_pyfunction!(sum_as_string, m)?)?;
    Ok(())
}
```
Thanks to the macros, `lib.rs` is fairly self explanatory. We define a function that we want to access from Python with `#[pyfunction]` and then we create a module definition `string_sum` and add the function to the module.

In order to access these functions, we need to run `maturin develop` which will compile the Rust lib and allow it to be used.

```python
$ maturin develop
$ python
>>> import string_sum
>>> string_sum.sum_as_string(5, 20)
'25'
```

Aweesome, we have our project set up and are now able to call Rust functions from Python.

### Flask Handlers

Now I'm thinking of a hypothetical situation here where you have a Flask application in production and you've been tasked with adding a highly critical endpoint for clients which returns the sum of all prime numbers up to the given number (I said hypothetical what more do you want).

So you think to yourself okay I'll add a new endpoint handler to my handlers module and create the endpoint:

```python
# handlers.py

import math

def sum_of_primes(n:int) -> int:
    return sum(i for i in range(2, n) if is_prime(i))

def is_prime(n:int) -> bool:
    return not any(n%i == 0 for i in range(2, int(math.sqrt(n))+1))

# main.py

import handlers

@app.route("/sum_of_primes/<int:n>")
def super_slow_snakey_endpoint(n):
    return str(handlers.sum_of_primes(n))

```

Cool all is working fine and as expected, if we give that endpoint a quick curl with the number 6, we receive 10:

```bash
$ curl localhost:5001/sum_of_primes/6
10
```

There's one issue though, your task clearly mentions that the largest possible value of `n` that will be being received is 1,000,000 and the response time is too slow as of it's current moment.

Using the CLI tool [hey](https://github.com/rakyll/hey) you run a quick test and find that running 50 requests takes over 100 seconds at an average of 2.03 seconds per request.

Not ideal, I suppose we could implement a [sieve](https://en.wikipedia.org/wiki/Sieve_of_Eratosthenes) function for determining whether a number is prime to try and optimise the code ... or we could just write it in Rust?

Opting for the latter:

```rust
// lib.rs

use pyo3::prelude::*;

#[pyfunction]
fn sum_of_primes(n:usize) -> usize {
    (2..n).filter(|x| is_prime(x)).sum()
}

fn is_prime(n:&usize) -> bool {
    let lim = (*n as f64).sqrt() as usize;
    !(2..(lim+1)).any(|i| n%i == 0)
}

#[pymodule]
fn handle_rs(_py: Python, m: &PyModule) -> PyResult<()> {
    m.add_function(wrap_pyfunction!(sum_of_primes, m)?)?;
    Ok(())
}
```

With our Rust translation of our endpoint handler, we need to make one change to `cargo.toml` to ensure that our `lib.name` matches that of the function with the `#pymodule` macro:
```toml
[lib]
name = "handle_rs"
````

Now just to confirm everything works as intended and we get the same results from both `handlers.py` and `handle.rs` (the namespacing is confusing, I saw 'rs' in handlers and simple had no choice):

```bash
$ python
>>>import handlers
>>>import handle_rs
>>>handlers.sum_of_primes(6)
10
>>>handle_rs.sum_of_primes(6)
10
```

Everything works, great news!

### Time for production

You make your final changes to `main.py`:

```python
import handle_rs

@app.route("/sum_of_primes/<int:n>")
def blazingly_fast_rust_endpoint(n):
    return str(handle_rs.sum_of_primes(n))
```

As a final check you run one more curl:

```bash
$ curl localhost:5001/sum_of_primes/6
10
```

Same behaviour + more speed == better

Push MR/PR to Git/GitLab, test pipeline pass, build pipeline starts. Uh oh, the build failed. Of course it did though, we're calling a new module and it's not getting installed in the Dockerfile.

### It's Docker Time

Now beforehand your super complex production Flask app had the following Dockerfile
```dockerfile
FROM python:3.10

RUN pip install flask gunicorn

COPY . /app
WORKDIR /app

ENTRYPOINT ["gunicorn","-b", "0:5000", "main:app"]
```

Okay so all we need to do is `pip install maturin` as well and run `maturin develop  -b pyo3 --release` right?

```dockerfile
FROM python:3.10

RUN pip install flask gunicorn maturin

COPY . /app
WORKDIR /app
RUN maturin develop -b pyo3 --release

ENTRYPOINT ["gunicorn","-b", "0:5000", "main:app"]
```

Ah but of course, this is a Python image, not a Rust image so we need to install `rustc` as well, for which we'll need `curl` to install `rustc` and add it to `$PATH`.

```dockerfile
FROM python:3.10

RUN pip install flask gunicorn maturin

RUN apt-get install -y curl
RUN apt-get update

RUN curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
ENV PATH="/root/.cargo/bin:${PATH}"

COPY . /app
WORKDIR /app
RUN maturin develop -b pyo3 --release

ENTRYPOINT ["gunicorn","-b", "0:5000", "main:app"]
```

Unfortunately we get an error yet still:
```
maturin failed
Caused by: Couldn't find a virtualenv or conda environment,
but you need one to use this command
```


Riiighhht, using a virtualenv inside of a Docker, which is a virtual environment itself, now how does that work.

I admit defeat here and couldn't work it out easily, thankfully this [awesome article](https://pythonspeed.com/articles/activate-virtualenv-dockerfile/) solved the issue for me, we simply need to create the env and add it to path as well:

```dockerfile
FROM python:3.10

RUN apt-get install -y curl
RUN apt-get update

RUN curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
ENV PATH="/root/.cargo/bin:${PATH}"

ENV VIRTUAL_ENV=/opt/venv 
RUN python3 -m venv $VIRTUAL_ENV
ENV PATH="$VIRTUAL_ENV/bin:$PATH"

RUN pip install flask gunicorn maturin

COPY . /app
WORKDIR /app
RUN maturin develop -b pyo3 --release

ENTRYPOINT ["gunicorn","-b", "0:5000", "main:app"]
```
There is also a much simpler alternative method I missed in the later parts of the error - "or use `maturin build` and `pip install <path/to/wheel>` instead":

```dockerfile
FROM python:3.10

RUN apt-get install -y curl
RUN apt-get update

RUN curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
ENV PATH="/root/.cargo/bin:${PATH}"

ENV VIRTUAL_ENV=/opt/venv 
RUN python3 -m venv $VIRTUAL_ENV
ENV PATH="$VIRTUAL_ENV/bin:$PATH"

RUN pip install flask gunicorn maturin

COPY . /app
WORKDIR /app
RUN maturin build -b pyo3 --release --out /opt/handle_rs
RUN pip install /opt/handle_rs/*

ENTRYPOINT ["gunicorn","-b", "0:5000", "main:app"]
```

### The Final Test

Before revolutionising your Flask app with a Rust powered endpoint, you realise it's probably worth getting some statistics on how much difference it's actually made. Duplicating the endpoints like so:

```python
import handlers
import handle_rs
from flask import Flask

app = Flask(__name__)

@app.route("/rust/<int:n>")
def blazingly_fast_endpoint(n):
    return str(handle_rs.sum_of_primes(n))

@app.route("/python/<int:n>")
def super_slow_snakey_endpoint(n):
    return str(handlers.sum_of_primes(n))

if __name__ == "__main__":
    app.run("0", 5001, debug=True)
```

Using `hey localhost:5000/<rust|python>/100000` yields the following results:

For python we have a total time of 16.9 seconds for 200 requests, an average of serving 11.77 requests/second.
Rust did all 200 requests in 0.61 seconds serving an average of 329.82 requests/second.

Our simple rewrite of our handler in Rust allowed our Flask app to serve approximately **28x** as many requests per second as the Python implementation.

### Final Remarks

Extending an existing Python application with a Rust module and Python bindings has a very significant potential gain to be made, while only requiring some slight changes to container setup (although granted it will bloat the image a bit more).
