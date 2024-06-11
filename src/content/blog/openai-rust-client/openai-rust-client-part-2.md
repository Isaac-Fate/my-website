---
author: Isaac Fei
pubDatetime: 2024-06-09T09:12:45.396Z
modDatetime: null
title: "Building a Rust Client for OpenAI - Part 2: Models API"
slug: openai-rust-client-part-2
featured: true
draft: false
tags:
- rust
- openai
description: Through implementing functions for models API, we will learn basics of sending requests, serializing and deserializing data, handling errors, and more.
---

In this second part of the series, we will begin by developing functions for interacting with OpenAI's APIs.

Please refer to the [OpenAI API documentation](https://platform.openai.com/docs/api-reference/introduction) for additional details.

My project structure will mirror the organization of the API reference page:

```
src
├── lib.rs
├── models
│   ├── mod.rs
│   └── ...
├── chat
│   ├── mod.rs
│   └── ...
├── embeddings
│   ├── mod.rs
│   └── ...
└── ...
```

Each submodule is associated with an API. 

## Overview of the Submodule `models`

We will kick things off with the simplest [Models API](https://platform.openai.com/docs/api-reference/models). Over the post, we will cover the basics of sending requests, serializing and deserializing data, handling errors, and more.

I choose to implement functions 
- `list_models`
- `retrieve_model`

for the corresponding APIs.

> I will not implement the function for deleting a fine-tuned model since I have never used it in my projects. Maybe I will implement it in the future when I find it necessary.

The module structure for `models` is as follows:

```
models
├── endpoint.rs // Constant of API endpoint
├── response.rs // The response data model
├── api_calls
│   ├── list.rs // list_models
│   ├── retrieve.rs // retrieve_model
│   └── mod.rs
└── mod.rs
```

File `endpoint.rs` simply contains a constant:

```rust
// models/endpoint.rs

/// Endpoint of the models API.
pub const MODELS_API_ENDPOINT: &str = "https://api.openai.com/v1/models";
```

Now, we start implementing `list_models` function.

## Exploring the APIs with Postman

Postman is a great tool for exploring and testing APIs.

Create a new HTTP request in Postman as illustrated below:
  
![Postman](/images/postman-list-models.png)

- Create a new GET HTTP request and enter the API endpoint
- Select Bearer Token in the Authorization tab
- Enter your API key (It is recommended to store it as a variable)

Click Send, the response is like:

```json
{
    "object": "list",
    "data": [
        {
            "id": "dall-e-3",
            "object": "model",
            "created": 1698785189,
            "owned_by": "system"
        },
        {
            "id": "gpt-4-1106-preview",
            "object": "model",
            "created": 1698957206,
            "owned_by": "system"
        },
        ...
    ]
}
```

## Modelling the Response Struct

### First Attempt

Based on the JSON response received with Postman, 
a straightforward modelling may be as follows:

```rust
pub struct ListModelsResponse {
    pub object: String,
    pub data: Vec<Model>,
}

pub struct Model {
    pub id: String,
    pub object: String,
    pub created: u32,
    pub owned_by: String,
}
```

Personally, I think the field `object` is redundant
since the name of the struct itself already provides this information.


### Second Attempt

Removing the `object` fields, we have:

```rust
pub struct ListModelsResponse {
    pub data: Vec<Model>,
}

pub struct Model {
    pub id: String,
    pub created: u32,
    pub owned_by: String,
}
```

However, this layout still seems a bit counter-intuitive. The only information we actually need is the vector of models, `Vec<Model>`, which is nested inside the `data` field.

So, why not just use `Vec<Model>`?


### Final Decision 

The signature of function `list_models` is:


```rust
pub async fn list_models(client: &OpenAIClient) -> Result<Vec<Model>>
```

To parse the JSON response to our model, we need the struct to implement the `Deserialize` trait from the `serde` crate.
The canonical way to do this is to use the `#[derive(Deserialize)]` macro.

For complex data models and customized serialization and deserialization logic,
we will then need to implement the `Serialize` and `Deserialize` traits by hand.
We will have such examples in later parts of the series.

```rust
// models/response.rs

use serde::Deserialize;

#[derive(Bebug, Deserialize)]
pub struct Model {
    pub id: String,
    pub created: u32,
    pub owned_by: String,
}
```


## Sending Requests

```rs
// Send the GET request
let response = match client.get(MODELS_API_ENDPOINT).send().await {
    Ok(response) => response,

    // How to handle this Reqwest error?
    Err(error) => {
        return Err(todo!());
    }
};
```

If something goes wrong, we will get a Reqwest error.
We need to define a new error variant to wrap it since the return type of the function is our own `Result` type.

Define a new variant `Reqwest` of the `Error` enum, and add the `source` field to hold the original error:

```rust
// error.rs

/// The result type of this library.
pub type Result<T> = std::result::Result<T, Error>;

/// The error type of this library.
#[derive(thiserror::Error, Debug)]
pub enum Error {
    // ...

    #[error("got a Reqwest error: {source}")] 
    Reqwest {
        #[source]
        source: reqwest::Error,
    },

    // ...
}
```

### Refining Reqwest Errors

However, in some cases, we can provide more specific information of the Reqwest error.
For example, we may define variants `Timeout`, `Connection`, etc.

The Reqwest crate does not define these variants. 
Instead, its error has methods `is_timeout()`, `is_connection()`, etc. to distinguish different types of errors.



```rust
/// The error type of this library.
#[derive(thiserror::Error, Debug)]
pub enum Error {
    // ...

    #[error("the request timed out")]
    Timeout,

    #[error("failed to connect to endpoint")] 
    Connection,

    #[error("got a Reqwest error: {source}")] 
    Reqwest {
        #[source]
        source: reqwest::Error,
    },

    // ...
}
```

We can implement a `From` trait to convert the `reqwest::Error` into our own `Error` type:

```rust
impl From<reqwest::Error> for Error {
    fn from(error: reqwest::Error) -> Self {
        if error.is_timeout() {
            Error::Timeout
        } else if error.is_connect() {
            Error::Connection
        } else {
            // For other Reqwest errors we do not bother to distinguish them
            Error::Reqwest { source: error }
        }
    }
}
```

Then, we can convert the Reqwest error into our own `Error` type like so:

```rs
// Send the GET request
let response = match client.get(MODELS_API_ENDPOINT).send().await {
    Ok(response) => response,

    // Convert Reqwest error into our own error
    Err(error) => {
        return Err(Error::from(error));
    }
};
```

### Errors for Status


When the arm ` Ok(response)` is matched, it only means that we have successfully sent the request.
The response may not be successful.
For example the status code of the response may be 401, 500, etc.

To turn a response into an *errorable* (if that is even a word) result based on its status code, we may use the `error_for_status` method.

The previous snippet may be refactored as follows:

```rs
// Send the GET request
let response = match client.get(MODELS_API_ENDPOINT).send().await {
    // Turn the response into an errorable result
    Ok(response) => match response.error_for_status() {
        // Successful response
        Ok(response) => response,

        // We need to refactor the From<reqwest::Error> trait
        // to convert to more custom error variants 
        // associated with the status codes
        Err(error) => {
            return Err(Error::from(error));
        }
    },

    // Convert Reqwest error into our own error
    Err(error) => {
        return Err(Error::from(error));
    }
};
```

Referring to possible [OpenAI Error Codes](https://platform.openai.com/docs/guides/error-codes), we need to add more error variants:

```rust
// error.rs

/// The error type of this library.
#[derive(thiserror::Error, Debug)]
pub enum Error {
    // ...

    #[error("failed to authenticate with the provided OpenAI API")]
    Authentication,

    #[error("you are accessing the API from an unsupported country, region, or territory")]
    UnsupportedRegion,

    #[error(
        "you are sending requests too quickly, or run out of credits or hit your maximum monthly spend"
    )]
    ExceedRateLimitOrQuota,

    #[error("issue on OpenAI servers")]
    Server,

    #[error("OpenAI servers are experiencing high traffic")]
    Overloaded,

    // Other errors with status codes we do not care to specify
    #[error("unknown status code {status_code}: {source}")] 
    UnknownStatusCode {
        status_code: reqwest::StatusCode,

        #[source]
        source: reqwest::Error,
    },

    // ...
}
```

Meanwhile, we also need to refactor the `from` implementation to convert the `reqwest::Error` into our own `Error` type:

```rust
impl From<reqwest::Error> for Error {
    fn from(error: reqwest::Error) -> Self {
        // Check status code
        match error.status() {
            None => {
                if error.is_timeout() {
                    Error::Timeout
                } else if error.is_connect() {
                    Error::Connection
                } else {
                    Error::Reqwest { source: error }
                }
            }
            
            // Handle error status codes
            Some(status_code) =>
                match status_code {
                    // 401
                    reqwest::StatusCode::UNAUTHORIZED => Error::Authentication,

                    // 403
                    reqwest::StatusCode::FORBIDDEN => Error::UnsupportedRegion,

                    // 429
                    reqwest::StatusCode::TOO_MANY_REQUESTS => Error::ExceedRateLimitOrQuota,

                    // 500
                    reqwest::StatusCode::INTERNAL_SERVER_ERROR => Error::Server,

                    // 503
                    reqwest::StatusCode::SERVICE_UNAVAILABLE => Error::Overloaded,

                    // Other
                    _ => Error::UnknownStatusCode { status_code, source: error },
                }
        }
    }
}
```

## Deserializing JSON Responses

After receiving the response, I decided to first deserialize the response to a `serde_json::Value` struct since the received JSON object is not immediately the data model we want to return.

```rust
// Deserialize the response to a JSON value
let response = match response.json::<serde_json::Value>().await {
    Ok(response) => response,

    // Need to define a new error
    Err(error) => {
        return Err(todo!());
    }
}
```

### Nested Errors

Of course, we need to again handle the possible error. 
Instead of defining a direct enum variant of `Error` as what we did before, 
to make the error more structured, we can first define a variant `ModelsApi` of `Error` and a new enum `ModelsApiError` specially for the models API errors.

```rust
// error.rs

/// The error type of this library.
#[derive(thiserror::Error, Debug)]
pub enum Error {
    
    // ...

    #[error("failed to request the models API: {0}")] 
    ModelsApi(ModelsApiError),

    // ...
}

#[derive(Debug, thiserror::Error)]
pub enum ModelsApiError {
    ...
}
```

Handle the error:

```rust
// error.rs

#[derive(Debug, thiserror::Error)]
pub enum ChatApiError {
    #[error("failed to parse to chat completion: {source}")] ParseToChatCompletion {
        #[source]
        source: reqwest::Error,
    },

    // ...
}

// models/api_calls/list.rs

// Deserialize the response
let response = match response.json::<serde_json::Value>().await {
    Ok(response) => response,
    Err(error) => {
        return Err(Error::ModelsApi(ModelsApiError::ParseToJson { source: error }));
    }
};
```

### Parsing `serde_json::Value`

Next, as we have observed from Postman response, we are interested in the `data` property of the response.
Get it from `serde_json::Value` struct:

```rust
// Get the data property
let data = match response.get("data") {
    // Convert to a owned data
    Some(data) => data.clone(),

    // Handle error
    None => {
        return Err(Error::ModelsApi(ModelsApiError::MissingDataProperty));
    }
};
```

Finally, we can convert the `data` to `Vec<Model>` using `serde_json::from_value` (here is where the `Deserialize` trait for `Model` comes into play).:

```rust
// Parse to models
let models: Vec<Model> = match serde_json::from_value(data) {
    Ok(models) => models,
    Err(error) => {
        return Err(Error::ModelsApi(ModelsApiError::ParseToModels { source: error }));
    }
};
```


## Retrieving a Model

The implementation of the `retrieve_model` method is similar.

Only a few points need to be noted:
- Specify the route of the endpoint: `https://api.openai.com/v1/models/{model}`
- Interpret the 404 status code specifically as model not found
- We can deserialize the response to `Model` directly

The full code snippet is as follows:

```rust
/// Retrieves a model instance, providing basic information about the model such as the owner and permissioning.
pub async fn retrieve_model<S: AsRef<str>>(client: &OpenAIClient, model_name: S) -> Result<Model> {
    // Send the request
    let response = match
        // Specify the route
        client.get(format!("{}/{}", MODELS_API_ENDPOINT, model_name.as_ref())).send().await
    {
        Ok(response) =>
            match response.error_for_status() {
                Ok(response) => response,
                Err(error) => {
                    // Interpret 404 as model not found
                    if let Some(reqwest::StatusCode::NOT_FOUND) = error.status() {
                        return Err(
                            Error::ModelsApi(
                                ModelsApiError::ModelNotFound(model_name.as_ref().to_string())
                            )
                        );
                    }

                    return Err(Error::from(error));
                }
            }
        Err(error) => {
            return Err(Error::from(error));
        }
    };

    // Deserialize the response to Model directly
    let model = match response.json::<Model>().await {
        Ok(model) => model,
        Err(error) => {
            return Err(Error::ModelsApi(ModelsApiError::ParseToModel { source: error }));
        }
    };

    Ok(model)
}
```