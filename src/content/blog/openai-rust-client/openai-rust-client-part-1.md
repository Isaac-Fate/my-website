---
author: Isaac Fei
pubDatetime: 2024-05-24T08:02:29.727Z
modDatetime: null
title: "Building a Rust Client for OpenAI - Part 1: Authentication"
slug: openai-rust-client-part-1
featured: true
draft: false
tags:
- rust
- openai
description: How to load the API key from environment variables into a constant, how to wrap Rewest client using builder pattern, and how to send requests with bearer authentication.
---

I have just started diving into Rust and thought it would be interesting to document my journey. In these series of posts, I am going to share my process of building a Rust client for the OpenAI API. Along the way, I will be introducing some Rust concepts and cool skills that I pick up.

However, to keep things clear and focused, I will not list every single detail or pasting the complete code here.


## Getting Started

Create a new project: (Since this should be a library, I will use the `--lib` flag.)

```sh
cargo new --lib rustyopenai
```

Then, remove everything form `lib.rs`.
It is also common to have both `main.rs` and `lib.rs` in the same project 
in case that you actually want an executable to run and test the library.

Create a file `error.rs` under the source directory:

```
src
├── lib.rs
├── error.rs
└── ...
```

This file will contain the result and error type of the library.
For error handling, I recommend the crate [`thiserror`](https://docs.rs/thiserror/latest/thiserror/), which helps you define errors easily with macros.

```rs
/// The result type of this library.
pub type Result<T> = std::result::Result<T, Error>;

/// The error type of this library.
#[derive(thiserror::Error, Debug)]
pub enum Error {
    #[error("OpenAI API key is not set")]
    ApiKeyNotSet,

    // Define more errors as you continue working on the project...
}
```


## Lazy Static Loading from Environment Variables

To call OpenAI's API, we need an API key.
It is important to remember, for security reasons, never expose the key directly within your code. Instead, place it within a `.env` file.

By default, our library should try to load the API key from the environment variables, 
and then this loaded API key should be shared statically across all instances of the library. 

This sounds like we need to define the API key as a constant, say `OPENAI_API_KEY`.
But we need to initialized it once (and only once) with some procedures.
> We could not define it like `const OPENAI_API_KEY: &str = "sk-xxx";` since we should not hard code and expose the API key.

The crate [`lazy_static`](https://docs.rs/lazy_static/latest/lazy_static/) provides a way to do that. (Of course, we also need [`dotenv`](https://docs.rs/dotenv/latest/dotenv/) to read the `.env` file.)

```rs
use lazy_static::lazy_static;
use dotenv;

lazy_static! {
    /// The OpenAI API key.
    pub static ref OPENAI_API_KEY: Option<String> = {
        let _ = DOTENV_FILE_PATH.as_ref();
        dotenv::var("OPENAI_API_KEY").ok()
    };

    /// The path to the .env file.
    static ref DOTENV_FILE_PATH: Option<PathBuf> = {
        match dotenv::dotenv().ok() {
            Some(path) => {
                info!("loaded environment variables from {:?}", path);
                Some(path)
            }
            None => {
                warn!("failed to load environment variables");
                None
            }
        }
    };
}
```


## Builder Pattern

In order to send requests to the OpenAI API, we will first build a struct `OpenAIClient` which is basically a simple wrapper around `reqwest::Client`.
It is designed to assist us in constructing requests with authentication already incorporated into the header.

```rs
/// The OpenAI client.
/// It is a simple wrapper around the `reqwest` HTTP client
/// with the authorization when building requests.
pub struct OpenAIClient {
    api_key: String,
    // The inner `reqwest` HTTP client
    http_client: Client,
}
```

Normally, we would implement a `new` method for creating an instance of a struct.
For example, we could do something like this:
```rs
impl OpenAIClient {
    /// Creates a new `OpenAIClient`.
    pub fn new(api_key: String, http_client: Client) -> Self {
        Self {
            api_key,
            http_client,
        }
    }
}
```
There are some flaws with this approach.
First, it enforces the user to provide the API key.
(Well, we have some workarounds for that. For example, we could revise `new` to read the API key from environment variables by default, and create another method to allow the user to set the API key manually if they want.)
But another problem is that we are passing a (Reqwest) *client* into an (OpenAI) *client*, which is rather awkward.
Instead, we would like the user to use our `OpenAIClient` just like they would use the Reqwest client.


We can fix this with a builder pattern.
First, define a helper struct `OpenAIClientBuilder` and then implement the `build` method to create the `OpenAIClient` instance:
> Note that in `build`, we will make use of the `OPENAI_API_KEY`.

```rs
/// Builder for `OpenAIClient`.
pub struct OpenAIClientBuilder {
    api_key: Option<String>,
    http_client_builder: ClientBuilder,
}

impl OpenAIClientBuilder {
    /// Creates a new builder.
    pub fn new() -> Self {
        Self {
            api_key: None,
            http_client_builder: Client::builder(),
        }
    }

    /// Sets the API key.
    pub fn api_key<S: AsRef<str>>(mut self, api_key: S) -> Self {
        self.api_key = Some(api_key.as_ref().to_string());
        self
    }

    /// Sets the request timeout.
    pub fn timeout(mut self, timeout: Duration) -> Self {
        self.http_client_builder = self.http_client_builder.timeout(timeout);
        self
    }

    /// Builds the OpenAI client.
    pub fn build(self) -> Result<OpenAIClient> {
        // Get the API key
        // If the API key is not set, try to get it from the environment variables
        let api_key = self.api_key
            .or(OPENAI_API_KEY.as_ref().map(|api_key| api_key.to_string()))
            .ok_or(Error::ApiKeyNotSet)?;

        // Build an HTTP client
        match self.http_client_builder.build() {
            // Return the OpenAI client
            Ok(http_client) => {
                Ok(OpenAIClient {
                    api_key,
                    http_client,
                })
            }

            // Return the error
            Err(error) => { Err(Error::BuildHttpClient { source: error }) }
        }
    }
}

```

In `OpenAIClient`, define a method `builder` to create a new `OpenAIClientBuilder` for building itself:

```rs
impl OpenAIClient {
    /// Creates a builder for OpenAIClient.
    pub fn builder() -> OpenAIClientBuilder {
        OpenAIClientBuilder::new()
    }
}
```

Then, one may use `OpenAIClient` as follows:

```rs
// Create the OpenAI client with the API key in the environment and with the default HTTP client
let client = OpenAIClient::builder().build();

// Create the OpenAI client with manually set API key
let client = OpenAIClient::builder().api_key("sk-xxx").build();

// Create the OpenAI client with specified timeout
let client = OpenAIClient::builder().timeout(Duration::from_secs(10)).build();
```

## Bearer Authentication

Finally, we will end this post by implementing the `get` method for sending GET requests.

We will mimic the Reqwest client's `get` method.
Its function signature is 
```rs
reqwest::async_impl::client::Client
pub fn get<U>(&self, url: U) -> RequestBuilder
where
    U: IntoUrl,
```

Two things we can learn from this signature:
- We can use the trait bound `U: IntoUrl` to specify the type of the `url` parameter instead of `&str` or `String`. (Actually, for string parameters, we often use the trait bound `S: AsRef<str>`.)
- This method returns a `RequestBuilder`, which is as its name suggests, a builder for a request. (The builder pattern appears again.) By returning a request builder, we may further modify the request before sending it.

To include the authorization header in the request, we may simply use the `bearer_auth` method to modify the request builder:
> Try to feel how handy is the builder pattern.

```rs
impl OpenAIClient {
    /// Creates a GET request builder.
    /// Authorization header will be set with the API key.
    pub fn get<U: IntoUrl>(&self, url: U) -> RequestBuilder {
        self.http_client
            .get(url)

            // Set the authorization header
            .bearer_auth(self.api_key.as_str())
    }
}
```

Alternatively, we can set the authorization header manually using a formatted string:


```rs
impl OpenAIClient {
    /// Creates a GET request builder.
    /// Authorization header will be set with the API key.
    pub fn get<U: IntoUrl>(&self, url: U) -> RequestBuilder {
        self.http_client
            .get(url)

            // Set the authorization header
            .header("Authorization", format!("Bearer {}", self.api_key.as_str()))
    }
}
```

Now, we can use our authenticated OpenAI client to request APIs!
Observe the response you get with the following code:

```rs
// Create a client
let client = OpenAIClientBuilder::new().build().unwrap();

// What do you get?
let response = client
            .get("https://api.openai.com/v1/models")
            .send().await
            .unwrap()
            .json::<serde_json::Value>().await
            .unwrap();

println!("{:#?}", response);
```