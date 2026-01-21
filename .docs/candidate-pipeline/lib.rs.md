# lib.rs - Module Entry Point

## File Location
`candidate-pipeline/lib.rs`

## Purpose
This file is the entry point for the candidate-pipeline module. In Rust, `lib.rs` acts like a table of contents - it declares what parts of the module exist and makes them available to other code.

## Line-by-Line Explanation

```rust
pub mod candidate_pipeline;
```
**Line 1**: Make the `candidate_pipeline` file available publicly.
- `pub` = public (other code can use it)
- `mod` = module (a file or folder containing code)
- This tells Rust: "There's a file called `candidate_pipeline.rs` and I want other code to be able to use it."

```rust
pub mod filter;
```
**Line 2**: Make the `filter` file available publicly. This contains the definition of what a Filter should do.

```rust
pub mod hydrator;
```
**Line 3**: Make the `hydrator` file available publicly. Hydrators add information to posts.

```rust
pub mod query_hydrator;
```
**Line 4**: Make the `query_hydrator` file available publicly. These add information about the user making the request.

```rust
pub mod scorer;
```
**Line 5**: Make the `scorer` file available publicly. Scorers calculate how interesting each post is.

```rust
pub mod selector;
```
**Line 6**: Make the `selector` file available publicly. Selectors pick the best posts from the scored list.

```rust
pub mod side_effect;
```
**Line 7**: Make the `side_effect` file available publicly. Side effects are background tasks that don't affect the main result.

```rust
pub mod source;
```
**Line 8**: Make the `source` file available publicly. Sources are where posts come from.

```rust
pub mod util;
```
**Line 9**: Make the `util` file available publicly. This contains helper functions used by other files (not shown in the repository, but referenced).

## Why This Matters

Without this file, none of the other code in this folder would be accessible. It's like having a book with no table of contents - the chapters exist, but no one knows how to find them.

The order of the lines doesn't matter for functionality, but they're organized logically:
1. Main pipeline logic first
2. Then the components in roughly the order they run
3. Utilities last

## What Each Module Does (Summary)

| Module | Role in Pipeline |
|--------|------------------|
| `candidate_pipeline` | Orchestrates everything - the main controller |
| `filter` | Decides which posts to keep or remove |
| `hydrator` | Adds extra data to posts |
| `query_hydrator` | Adds extra data about the user |
| `scorer` | Assigns numeric scores to posts |
| `selector` | Chooses which posts make the final list |
| `side_effect` | Runs background tasks (logging, analytics) |
| `source` | Retrieves initial list of posts |
| `util` | Helper functions |
