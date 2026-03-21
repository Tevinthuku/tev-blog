+++
date = '2026-03-21T07:55:33+03:00'
draft = true
title = 'Api Versioning: Part 2 - Request versioning'
+++

This is just going to be a continuation of the [previous blog](https://tevinthuku.github.io/tev-writes/posts/api-versioning-part-1-response-versioning/)

The key insight is, for requests, the transformer operations happen in reverse.

Users will provide an old model, the library upgrades this model to what the route handler expects.

As a small example, lets assume we had a user creation endpoint. `/users`

In version 0.5 -> It accepts a `name` field

```rust
#[derive(Debug, Serialize, Deserialize, VersionChange)]
#[description = "Clients before v1.0.0 send `name` instead of `full_name`"]
pub struct LegacyCreateUserRequestV0_5 {
    pub name: String,
}
```

in version 1.0 -> It accepts a `full_name` field instead of name, users can then split their 2 names via a whitespace

```rust
#[derive(Debug, Serialize, Deserialize, VersionChange)]
#[description = "Clients before v2.0.0 send `full_name` instead of split name fields"]
pub struct LegacyCreateUserRequestV1 {
    pub full_name: String,
}
```

In version 2.0 -> The API finally performs a proper name split and provides 2 fields `first_name` and `last_name`

```rust
// the latest model
#[derive(Debug, Serialize, Deserialize, VersionChange)]
#[description = "The latest request model expects first and last names separately"]
pub struct CreateUserRequest {
    pub first_name: String,
    pub last_name: String,
}
```

Our request handler will be simple, it will always expect the latest model

```rust
#[post("/users")]
async fn create_user(
    user: VersionedJsonRequest<CreateUserRequest>,
) -> Result<VersionedJsonResponder<CreateUserResponse>> {
    ...
```

The `VersionedJsonRequest` is just similar to the `VersionedJsonResponder` that we covered in the previous post. As I write this, I see we could just use a single struct for both Request & Response, similar to [web::Json](https://docs.rs/actix-web/latest/actix_web/web/struct.Json.html), but I'll refactor this at a different point in time. However, the concept will still remain the same.
The `VersionJsonRequest` extracts the version-id provided and forwards the request body to the latest version.

1 more thing, we now have a demo service that handles this.

[Service-source code](https://github.com/Tevinthuku/version-api/tree/main/version-actix-roundtrip-example)

[https://version-api-11g3.onrender.com](https://version-api-11g3.onrender.com)

How to test this, keep in mind the service is deployed on the free plan, so it needs time to boot up first.

version 0.5.0

```bash
curl -sS \
  -H 'Content-Type: application/json' \
  -H 'X-API-Version: 0.5.0' \
  -d '{"name":"Alice Doe"}' \
  https://version-api-11g3.onrender.com/users
{"name":"Alice Doe","success":true}%
```

version 1.0.0

```bash
curl -sS \
  -H 'Content-Type: application/json' \
  -H 'X-API-Version: 1.0.0' \
  -d '{"full_name":"Alice Doe"}' \
  https://version-api-11g3.onrender.com/users
{"full_name":"Alice Doe","status":"created"}%
```

version 2.0.0

```bash
curl -sS \
  -H 'Content-Type: application/json' \
  -H 'X-API-Version: 2.0.0' \
  -d '{"first_name":"Alice", "last_name": "Doe"}' \
  https://version-api-11g3.onrender.com/users
{"first_name":"Alice", "last_name": "Doe", "status":"created"}%
```

The service is a full roundtrip example where both request and response's versions are respected so user clients do not break.

The last bit I'll work on is error handling. I have a number of todo! comments all over the repo for fixing the error handling. It would be great for me to distinguish between internal errors, eg: with my transformers, and external errors, eg: if a user genuinely provided a wrong DTO for an older version.
Will cover this later though
