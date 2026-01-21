# source.rs - Where Posts Come From

## File Location
`candidate-pipeline/source.rs`

## Purpose
This file defines what a "Source" is. A Source is responsible for fetching the initial list of posts that might appear in your feed. Think of it as the starting point - before any filtering or ranking happens, something needs to actually get the posts.

## Line-by-Line Explanation

```rust
use std::any::{Any, type_name_of_val};
```
**Line 1**: Import tools from Rust's standard library.
- `std::any` = A module for working with types at runtime
- `Any` = A special type that can represent any other type
- `type_name_of_val` = A function that gets the name of a value's type as text

```rust
use tonic::async_trait;
```
**Line 2**: Import the `async_trait` tool from the tonic library.
- `tonic` = A library for building network services
- `async_trait` = A helper that lets us define traits (blueprints) with async functions

```rust
use crate::util;
```
**Line 4**: Import the `util` module from this same crate (package).
- `crate` = Refers to the current package
- `util` = Helper functions defined elsewhere in this package

```rust
#[async_trait]
```
**Line 6**: A decorator that enables async functions in the trait below.
- Normally, Rust traits can't have `async` functions directly
- This decorator from `tonic` makes it possible

```rust
pub trait Source<Q, C>: Any + Send + Sync
```
**Line 7**: Define a public trait (blueprint) called `Source`.
- `pub trait` = A public blueprint that other code can implement
- `Source` = The name of this blueprint
- `<Q, C>` = Generic type parameters (Q = Query type, C = Candidate type)
- `: Any + Send + Sync` = Requirements for anything implementing this trait:
  - `Any` = Can be inspected at runtime
  - `Send` = Can be sent between threads safely
  - `Sync` = Can be shared between threads safely

```rust
where
    Q: Clone + Send + Sync + 'static,
    C: Clone + Send + Sync + 'static,
```
**Lines 8-10**: Additional requirements for the type parameters.
- `where` = Introduces constraints on the generic types
- `Q: Clone + Send + Sync + 'static` = The Query type must:
  - `Clone` = Be copyable
  - `Send` = Be safe to send between threads
  - `Sync` = Be safe to share between threads
  - `'static` = Live for the entire program duration (no borrowed references)
- `C:` = Same requirements for the Candidate type

```rust
{
```
**Line 11**: Start of the trait's method definitions.

```rust
    /// Decide if this source should run for the given query
    fn enable(&self, _query: &Q) -> bool {
        true
    }
```
**Lines 12-15**: Define the `enable` method.
- `///` = Documentation comment
- `fn enable` = A function named "enable"
- `&self` = Takes a reference to the Source itself
- `_query: &Q` = Takes a reference to a Query (the underscore means "I don't use this")
- `-> bool` = Returns a boolean (true or false)
- `{ true }` = Default implementation always returns true (source is enabled)

This method lets a Source decide whether it should run for a particular request. By default, all Sources are enabled, but specific implementations can override this to only run in certain situations.

```rust
    async fn get_candidates(&self, query: &Q) -> Result<Vec<C>, String>;
```
**Line 17**: Define the main method that fetches candidates.
- `async fn` = This function can wait for things (like network requests)
- `get_candidates` = The function name
- `&self` = Takes a reference to the Source itself
- `query: &Q` = Takes a reference to the Query (user info)
- `-> Result<Vec<C>, String>` = Returns either:
  - Success: A vector (list) of Candidates
  - Failure: An error message as a String

This is the core method - it does the actual work of finding posts. Notice there's no default implementation (no `{ }` with code), so every Source MUST implement this method.

```rust
    fn name(&self) -> &'static str {
        util::short_type_name(type_name_of_val(self))
    }
```
**Lines 19-21**: Define the `name` method.
- `fn name` = A function named "name"
- `&self` = Takes a reference to the Source itself
- `-> &'static str` = Returns a string that lives forever (a string literal)
- The implementation uses helper functions to get the type's name automatically

This method returns the name of the Source for logging purposes. The default implementation automatically extracts the type name, so implementations don't need to specify their own name.

```rust
}
```
**Line 22**: End of the trait definition.

## What This Code Does in Practice

When the pipeline runs, it:

1. Goes through each Source it has
2. Calls `enable()` to check if the Source should run for this request
3. If enabled, calls `get_candidates()` to fetch posts
4. Combines all the posts from all enabled Sources

## Example of How a Real Source Would Work

While this file only defines the blueprint, a real Source implementation might look like:

```rust
struct ThunderSource {
    client: ThunderClient,  // Connection to the Thunder database
}

impl Source<Query, Candidate> for ThunderSource {
    async fn get_candidates(&self, query: &Query) -> Result<Vec<Candidate>, String> {
        // Call Thunder to get recent posts from followed users
        self.client.get_posts_for_user(query.user_id).await
    }
}
```

## Key Takeaways

1. **Source is a blueprint** - It defines what any Source must be able to do
2. **Sources fetch the initial posts** - Before filtering, scoring, or ranking
3. **Multiple Sources can run in parallel** - The pipeline can get posts from different places at once
4. **enable() provides flexibility** - Different Sources can run in different situations
5. **get_candidates() is the core work** - Every Source must implement this
