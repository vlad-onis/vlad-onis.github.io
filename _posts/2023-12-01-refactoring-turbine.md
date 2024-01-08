---
title: Refactoring turbine 
date: 2023-12-01 13:00:00
categories: [Rust, Web Servers]
tags: [rust]
---

So here we are again, trying to build this little turbine. Before advancing in our quest we should refactor it and not allow tech debt to pile up. 

"Refactoring already?? But we barely wrote 5 lines of code", you may say. You see, a Lanister always pays his (tech) debt. Besides, that's a really fun way to leverage Rust's type system and make turbine more reliable. 

<img src="/assets/img/the_birth_of_turbine/turbine.png" width="400" height="200" style="border-radius:25% 50%;">

## Refactoring

To keep it short this time as well we want to just tackle the decoupling of reading and writing and sprinkle some more Rust types along the way.

Let's consider the code from part 1 a single function. You will quickly notice that the function reads from the stream and creates the request object and in the second chunk of code we parse the request extracting the resource from it.

Let's move the http related bits to a new module (we can call it http.rs for example).

First step we need the methods that we can serve. We can use **enum** in this case as a request can't be both Get and Post at the same time. Yes we only use these 2 methods for now to keep it simple stupid.

```rust
#[derive(Debug)]
pub enum Method {
    Get,
    Post,
}
```

Next up we want to represent our Request type somehow.

```rust
#[derive(Debug)]
pub struct Request {
    pub headers: Headers,
    pub body: Vec<u8>,
}
```
And now suddenly the Headers type just popped up. Feels like we're really designing something 🦀

Like we learned in the previous post, headers will contain the 3 words space separated - method, resource, version. And then any other headers.
 [List of valid headers and their meaning](https://en.wikipedia.org/wiki/List_of_HTTP_header_fields)

```rust
#[derive(Debug)]
pub struct Headers {
    pub method: Method,
    pub resource: String,
    pub version: String,
    pub other_headers: HashMap<String, String>,
}
```
Find below the entire http.rs module. What I added besides what we discussed are the constructor methods for **Headers** and **Request** and a new Error type to keep things idiomatic and handle error cases nicely.

```rust
use std::collections::HashMap;
#[allow(unused_imports)]
use std::io::{Read, Write};
use std::ops::Deref;
use std::path::Path;
use std::path::PathBuf;

use std::fs;

use thiserror::Error;

/// Errors that can occur when parsing a http request
#[derive(Error, Debug)]
pub enum ParseError {
    #[error("Http request cannot be empty")]
    EmptyRequest,

    #[error("Headers must have method, resource, version")]
    InvalidHeaders,

    #[error("IO error: {0}")]
    IO(#[from] std::io::Error),

    #[error("Unknown or unsupported http method : {0}")]
    InvalidMethod(String),

    #[error("Path {0} is invalid")]
    InvalidPath(PathBuf),
}

/// Supported HTTP methods
#[derive(Debug)]
pub enum Method {
    Get,
    Post,
}

/// Representation of HTTP headers
#[derive(Debug)]
pub struct Headers {
    pub method: Method,
    pub resource: String,
    pub version: String,

    // All the possible http headers will be stored here
    pub other_headers: HashMap<String, String>,
}

impl Headers {
    /// Creates a new [Headers] instance from a vector of strings
    pub fn new(headers: Vec<&str>) -> Result<Headers, ParseError> {
        // At least the method, resource and version should be present
        if headers.len() != 3 {
            return Err(ParseError::InvalidHeaders);
        }

        let method = match headers[0] {
            "GET" => Method::Get,
            "POST" => Method::Post,
            unknown => return Err(ParseError::InvalidMethod(unknown.to_string())),
        };

        let resource = headers[1].to_string();
        let version = headers[2].to_string();

        // TODO: Parse other headers
        let other_headers = HashMap::new();

        Ok(Headers {
            method,
            resource,
            version,
            other_headers,
        })
    }
}

/// Representation of a HTTP request
#[derive(Debug)]
pub struct Request {
    pub headers: Headers,
    pub body: Vec<u8>,
}

impl Request {
    pub fn new(request: String) -> Result<Request, ParseError> {
        let lines: Vec<_> = request.split("\r\n").collect();

        let first_line = lines.first().ok_or(ParseError::EmptyRequest)?;

        let words = first_line.split_whitespace().collect::<Vec<_>>();

        let headers = Headers::new(words)?;

        // todo: Extract the body when we're dealing with POST requests
        let body = Vec::new();

        Ok(Request { headers, body })
    }
}

/// Specifies a valid HTTP path after parsing
#[derive(Debug)]
pub struct HttpPath(PathBuf);

impl Deref for HttpPath {
    type Target = PathBuf;

    fn deref(&self) -> &Self::Target {
        &self.0
    }
}

impl AsRef<Path> for HttpPath {
    fn as_ref(&self) -> &Path {
        self.0.as_path()
    }
}

/// Usage:
/// ```rust
/// let path = HttpPath::try_from("/web_resources/index.html".to_string());
/// ```
impl TryFrom<PathBuf> for HttpPath {
    type Error = ParseError;

    fn try_from(path: PathBuf) -> Result<Self, Self::Error> {
        let canonicalized_path = fs::canonicalize(path)?;

        if canonicalized_path.is_file() {
            return Ok(HttpPath(canonicalized_path));
        }

        // assume index.html as the default file to look for when the path is a directory
        if canonicalized_path.is_dir() {
            return Ok(HttpPath(canonicalized_path.join("index.html")));
        }

        Err(ParseError::InvalidPath(canonicalized_path))
    }
}
```

Let's quickly discuss the new type in there HttpPath. You can see it's just a wrapper over simple path type, why do we need it. Well you see this type implements the TryFrom<PathBuf> trait which will try to figure out what resource the server should serve. Our resources will live as we'll see later in a resources folder. What if the client requests something outside that folder? A lot of malicious stuff can happen. 

```bash
# With this request the client can try to request files outside our document root
curl --path-as-is 0.0.0.0:12345/../../../index.html -vvvvv
```

The nice part about our object is that by design it will only hold valid paths otherwise an error will be thrown and it will be thrown early, no later validation needed.

Now let's leave the http module and start splitting our server code in functions

```rust
/// This function reads the raw request bytes and creates our HttpRequest object
fn read_stream_content_to_end(stream: &mut TcpStream) -> Result<http::Request, http::ParseError> {
    let mut buffer = [0; 1024]; // Adjust buffer size as needed
    let mut request = Vec::new();

    loop {
        let bytes_read = stream.read(&mut buffer)?;

        if bytes_read == 0 {
            break; // Connection was closed
        }

        request.extend_from_slice(&buffer[..bytes_read]);

        // Check if the end of the request is reached
        if request.ends_with(b"\r\n\r\n") {
            break;
        }
    }

    let request = String::from_utf8_lossy(&request).to_string();
    let request = HttpRequest::new(request)?;

    Ok(request)
}
```

Parsing the request becomes very simple. Read it and return the resource.

```rust
fn parse_request(stream: &mut TcpStream) -> Result<String, ParseError> {
    let request = read_stream_content_to_end(stream)?;

    let resource = request.headers.resource;

    Ok(resource)
}
```

Let's finish by serving the right file to the caller

```rust
fn serve_file(mut stream: TcpStream) -> Result<(), ServerError> {
    let resource = parse_request(&mut stream)?;
    println!("Parsed Resource : {:?}", resource);

    let file_content = fs::read_to_string("index.html")?;
    let content_length = file_content.len() + END_OF_CONTENT.len();

    stream.write_all(HEADER_STATUS.as_bytes())?;
    stream.write_all(HEADER_CONTENT_TYPE.as_bytes())?;

    let content_length = format!("Content-Length: {}\r\n", content_length);
    stream.write_all(content_length.as_bytes())?;

    stream.write_all(NEW_LINE.as_bytes())?;

    stream.write_all(file_content.as_bytes())?;
    stream.write_all(END_OF_CONTENT.as_bytes())?;

    Ok(())
}
```

In the Annex below you can find the entire code of my main file so you can go through it once more and test it for yourself.


## Outro

It was not easy, we had to get our hands a bit dirty and set up the stage for what is to come. Now reading and parsing are decoupled and we have some new types that only hold valid states. We also implemented traits like TryFrom and Deref for them so our code is idiomatic and well decoupled.

Keep in mind that our server is still very much a toy and going forward we want to come up with more types and separate concerns a bit with new modules, new objects and a different design.

## Annex

```rust
mod http;

use http::{ParseError, Request as HttpRequest};

use std::fs;
use std::io::Result as IOResult;
#[allow(unused_imports)]
use std::io::{Read, Write};
use std::net::TcpListener;
use std::net::TcpStream;
use std::string::FromUtf8Error;

use thiserror::Error;


#[derive(Error, Debug)]
enum ServerError {
    #[error("IO error: {0}")]
    IO(#[from] std::io::Error),

    #[error("Failed to convert byte stream to String: {0}")]
    Conversion(#[from] FromUtf8Error),

    #[error("Parsing the request failed because: {0}")]
    RequestParsing(#[from] http::ParseError),
}

// Static lifetime is infered here
const END_OF_CONTENT: &str = "\r\n\r\n";
const HEADER_STATUS: &str = "HTTP/1.1 200 OK\r\n";
const HEADER_CONTENT_TYPE: &str = "Content-Type: text/html; charset=UTF-8\r\n";
const NEW_LINE: &str = "\r\n";

fn read_stream_content_to_end(stream: &mut TcpStream) -> Result<http::Request, http::ParseError> {
    let mut buffer = [0; 1024]; // Adjust buffer size as needed
    let mut request = Vec::new();

    loop {
        let bytes_read = stream.read(&mut buffer)?;

        if bytes_read == 0 {
            break; // Connection was closed
        }

        request.extend_from_slice(&buffer[..bytes_read]);

        // Check if the end of the request is reached
        if request.ends_with(b"\r\n\r\n") {
            break;
        }
    }

    let request = String::from_utf8_lossy(&request).to_string();
    let request = HttpRequest::new(request)?;

    Ok(request)
}

fn parse_request(stream: &mut TcpStream) -> Result<String, ParseError> {
    let request = read_stream_content_to_end(stream)?;

    let resource = request.headers.resource;

    Ok(resource)
}

fn serve_file(mut stream: TcpStream) -> Result<(), ServerError> {
    let resource = parse_request(&mut stream)?;
    println!("Parsed Resource : {:?}", resource);

    let file_content = fs::read_to_string("index.html")?;
    let content_length = file_content.len() + END_OF_CONTENT.len();

    stream.write_all(HEADER_STATUS.as_bytes())?;
    stream.write_all(HEADER_CONTENT_TYPE.as_bytes())?;

    let content_length = format!("Content-Length: {}\r\n", content_length);
    stream.write_all(content_length.as_bytes())?;

    stream.write_all(NEW_LINE.as_bytes())?;

    stream.write_all(file_content.as_bytes())?;
    stream.write_all(END_OF_CONTENT.as_bytes())?;

    Ok(())
}

fn _greet(mut stream: TcpStream) {
    // // Set up reading
    // let _ = stream.set_read_timeout(Some(Duration::from_micros(10)));
    // let mut buf: Vec<u8> = Vec::new();
    // let _ = stream.read_to_end(&mut buf);

    let _ = stream.shutdown(std::net::Shutdown::Read);

    let response = "HTTP/1.1 200 OK\r\nConnection: Closed\r\n\r\n";
    let _ = stream.write_all(response.as_bytes());
}

fn main() -> IOResult<()> {
    println!("Starting turbine");
    let listener = TcpListener::bind("0.0.0.0:12345")?;

    for stream in listener.incoming() {
        println!("#### New connection received");
        if let Ok(s) = stream {
            let res = serve_file(s);
            println!("Serving: {:?}", res);
        }
    }

    Ok(())
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    pub fn test_parse_headers_fail() {
        assert!(Headers::new(vec![]).is_err());
        assert!(Headers::new(vec!["GET", "/"]).is_err());
        assert!(Headers::new(vec!["GWET", "/", "HTTP/1.1"]).is_err());
    }

    #[test]
    pub fn test_parse_headers() {
        let header = Headers::new(vec!["GET", "/", "HTTP/1.1"]);
        println!("{header:?}");

        assert!(Headers::new(vec!["GET", "/", "HTTP/1.1"]).is_ok());
        assert!(Headers::new(vec!["POST", "/", "HTTP/1.1"]).is_ok());
        assert!(Headers::new(vec!["GET", "/foo", "HTTP/1.1"]).is_ok());
    }
}
```