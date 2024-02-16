---
title: Configuring Turbine 
date: 2024-02-16 13:00:00
categories: [Rust, Web Servers]
tags: [rust]
---

We meet again, trying to build this little turbine 🚀. This is the third post in this series; we are building a didactic webserver meant to teach us Rust alongside generic computer science knowledge. For a proper introduction please check:
* [part 1 - The birth of Turbine](https://vlad-onis.github.io/posts/birth-of-turbine/)
* [part 2 - Refactoring Turbine](https://vlad-onis.github.io/posts/refactoring-turbine/)

What do I even mean by configuring? Turbine currently is pretty hardcoded so if we want to change its behaviour from outside we will need ways to configure it. We will do that through config files, a pretty common way to configure web servers. Let's jump right into it.

<img src="/assets/img/the_birth_of_turbine/turbine.png" width="400" height="200" style="border-radius:25% 50%;">

### Parsing configs

What could be a good thing to configure as a first step?
You may have noticed that in the previous post we were serving files from a certain path and we were even talking about the threat of serving from the wrong place. This path is usually called "the document root". Let's use this for our config, we want the user to be able to choose a document root for his webserver.

We will use [clap](https://crates.io/crates/clap) which allows you to create command line parsers declaratively or procedurally. We will send the config_file as cli argument at server startup.

I created a new module and called it config.rs, let's break it down.

```rust
use std::path::PathBuf;

use anyhow::Result;
use serde::Deserialize;

use clap::Parser;

#[derive(Parser, Debug)]
pub struct Args {
    #[clap(short, long, default_value = "turbine.toml")]
    pub config_file: PathBuf,
}
```

In the above snippet we declare the Args struct which will consume the cli args you send from the command line and construct the object. The config file will be a PathBuf which does not hold any guarantees at this point that the path is valid, that will be our job soon.

Then in main.rs we can extract the arguments with something along the lines of:
```rust
let args = Args::parse();
println!("{:?}", args);
```

And then we can call it from the terminal with:
```bash
cargo run -- --config-file turbine_config.toml
```

Notice that if you pass an invalid argument that can't be parsed like "--some_config_arg" clap will print the error and exit.

Now let's go back to our config.rs and finish the module by creating the Config struct. Here's the entire module code.

```rust
use std::path::PathBuf;

use anyhow::Result;
use serde::Deserialize;

use clap::Parser;

#[derive(Parser, Debug)]
pub struct Args {
    #[clap(short, long, default_value = "turbine.toml")]
    pub config_file: PathBuf,
}

#[derive(Debug, Clone, Deserialize)]
pub struct Config {
    pub document_root: PathBuf,
}

impl Default for Config {
    fn default() -> Self {
        Config {
            document_root: PathBuf::from("web_resources"),
        }
    }
}

impl Config {
    pub fn new(config_file: PathBuf) -> Result<Self> {
        if !config_file.exists() || config_file.ends_with("toml") {
            return Err(anyhow::anyhow!(
                "Config file does not exist or is not a toml file"
            ));
        }

        let content = std::fs::read_to_string(config_file)?;

        let config = match toml::from_str(&content) {
            Ok(config) => config,
            Err(_e) => Config::default(),
        };

        Ok(config)
    }
}

```

As you can observe, we are creating this new Config struct. Upon creation we validate that the config_file is actually valid, we read it and we construct the Config object. This operation can fail on file reading, on toml deserialisation or if the config_file does not exist at all. To avoid complex error handling for now, [the anyhow crate](https://crates.io/crates/anyhow) is used to abstract away this complexity.

Here's the toml config file as an example
```toml
document_root = "./web_resources"
```

That's it for parsing the config, let's go use it!

### Using the document root config

The Server object will use the document_root to know where to serve from so we can just add that as a parameter to our Server, right.

```rust

pub struct Server {
    resolver: Resolver,
}

impl Server {
    pub fn new(config: Config) -> Result<Self, ServerError> {
        let document_root = config.document_root;
        let canonicalized_document_root = fs::canonicalize(document_root)?;
        Ok(Self {
            resolver: Resolver::new(canonicalized_document_root),
        })
    }
}
```

Here's the snippet that does something like that. It gets the document_root from the config object we already parsed, canonicalizes it and then creates the server alongised its Resolver. 

Yeah I know, I cheated and added a Resolver object all of a sudden without telling you about it. This is a refactoring move but I'll quickly explain it. The resolver just ensures that the server serves files only inside the document root. It's a path resolver actually, making sure paths are valid while also making things easy to unit test ^^. Here's a sneak peak into it.

```rust
/// Errors that can occur when parsing a http request
#[derive(Error, Debug)]
pub enum ResolveError {
    #[error("Resource {0} could not be found because it points outside the document root")]
    PathOutsideDocumentRoot(HttpPath),

    #[error("Path {0} should start with a slash")]
    PathShouldStartWithSlash(String),

    #[error("HttpError: {0}")]
    HttpPathError(#[from] crate::http::ParseError),
}

pub struct Resolver {
    /// The canonicalized document root
    document_root: PathBuf,
}

impl Resolver {
    pub fn new(document_root: PathBuf) -> Self {
        Self { document_root }
    }

    /// Parses the request and returns the resource path as an absolute path
    ///
    /// The path is validated to ensure that it is a file inside the
    /// web_resources directory
    ///
    /// # Errors
    ///
    /// Returns an error if the path
    /// - cannot be converted to an `HttpPath`
    /// - is outside the document root
    pub fn resolve(&self, resource: String) -> Result<HttpPath, ResolveError> {
        if !resource.starts_with('/') {
            return Err(ResolveError::PathShouldStartWithSlash(resource));
        }

        // Absolute paths replace the document root
        // Therefore we need to remove the leading slash
        let trimmed = resource.trim_start_matches('/');
        let resource = self.document_root.join(trimmed);

        // this is an absolute path
        let http_path = HttpPath::try_from(resource)?;

        // check if the absolute path file is inside the document root
        if !http_path.starts_with(&self.document_root) {
            return Err(ResolveError::PathOutsideDocumentRoot(http_path));
        }
        Ok(http_path)
    }
}
```

### Wrap up

I think this wraps up the first part of our journey - The (single threaded) Turbine. It was a joy to explore the basics of webservers with you up to this point. You can find all the code up to this point with comments everywhere on [this branch](https://github.com/vlad-onis/turbine/tree/task/sync_single_thread_turbine)

We still have a few things to do, here's the plan:
* Load test our turbine
* Warp speed turbine and parallelism
* Allow turbine to write scripts and reinvent the first backend ever 😅

### Outro

As always, it was fun! I will dedicate this outro to the mighty Rust that makes these concepts so rewarding to learn and code! See you very soon! 🦀🚀