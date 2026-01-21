# main.rs - Server Entry Point

## File Location
`home-mixer/main.rs`

## Purpose
This file starts the Home Mixer server. It's the entry point - when you want to run the recommendation system, this is what gets executed. It sets up networking, configures compression, and starts listening for requests.

## Line-by-Line Explanation

### Imports (Lines 1-14)

```rust
use clap::Parser;
```
**Line 1**: Import the command-line argument parser.
- `clap` = Command Line Argument Parser library
- Lets the program accept configuration via command-line flags

```rust
use log::info;
```
**Line 2**: Import the info logging function.
- Used to write informational messages to logs

```rust
use std::time::Duration;
```
**Line 3**: Import Duration for time measurements.
- Used for timeout configuration

```rust
use tonic::codec::CompressionEncoding;
```
**Line 5**: Import compression options for network communication.
- Allows sending/receiving compressed data

```rust
use tonic::service::RoutesBuilder;
```
**Line 6**: Import route builder for the gRPC server.
- gRPC = Google Remote Procedure Call, a protocol for services to communicate

```rust
use tonic_reflection::server::Builder;
```
**Line 7**: Import reflection service builder.
- Allows clients to discover what methods the server supports

```rust
use xai_home_mixer_proto as pb;
```
**Line 9**: Import the protocol buffer definitions.
- Protocol buffers define the structure of messages sent over the network
- `pb` is a short alias for these definitions

```rust
use xai_http_server::{CancellationToken, GrpcConfig, HttpServer};
```
**Line 10**: Import server utilities.
- `CancellationToken` = Used to signal when to shut down
- `GrpcConfig` = Configuration for the gRPC server
- `HttpServer` = The main server that handles requests

```rust
use xai_home_mixer::HomeMixerServer;
```
**Line 12**: Import the Home Mixer service implementation.

```rust
use xai_home_mixer::params;
```
**Line 13**: Import parameter constants.

### Command-Line Arguments (Lines 15-26)

```rust
#[derive(Parser, Debug)]
#[command(about = "HomeMixer gRPC Server")]
struct Args {
```
**Lines 15-17**: Define command-line arguments.
- `#[derive(Parser, Debug)]` = Automatically generate argument parsing code
- `#[command(about = "...")]` = Description shown in help text

```rust
    #[arg(long)]
    grpc_port: u16,
```
**Lines 18-19**: Port for gRPC communication.
- `#[arg(long)]` = Use `--grpc_port` syntax
- `u16` = Unsigned 16-bit integer (port numbers are 0-65535)

```rust
    #[arg(long)]
    metrics_port: u16,
```
**Lines 20-21**: Port for metrics/monitoring.

```rust
    #[arg(long)]
    reload_interval_minutes: u64,
```
**Lines 22-23**: How often to reload configuration (in minutes).

```rust
    #[arg(long)]
    chunk_size: usize,
```
**Lines 24-25**: Size of data chunks for processing.

```rust
}
```
**Line 26**: End of Args struct.

### Main Function (Lines 28-78)

```rust
#[xai_stats_macro::main(name = "home-mixer")]
#[tokio::main]
async fn main() -> anyhow::Result<()> {
```
**Lines 28-30**: The main function definition.
- `#[xai_stats_macro::main(...)]` = Custom macro for statistics collection
- `#[tokio::main]` = Run this function in an async runtime
- `async fn main()` = The main function is asynchronous
- `-> anyhow::Result<()>` = Returns either success or an error

```rust
    let args = Args::parse();
```
**Line 31**: Parse command-line arguments into the Args struct.

```rust
    xai_init_utils::init().log();
    xai_init_utils::init().rustls();
```
**Lines 32-33**: Initialize logging and TLS (secure connections).
- `log()` = Set up logging system
- `rustls()` = Set up TLS encryption

```rust
    info!(
        "Starting server with gRPC port: {}, metrics port: {}, reload interval: {} minutes, chunk size: {}",
        args.grpc_port, args.metrics_port, args.reload_interval_minutes, args.chunk_size,
    );
```
**Lines 34-37**: Log the server configuration at startup.

```rust
    // Create the service implementation
    let service = HomeMixerServer::new().await;
```
**Lines 39-40**: Create the Home Mixer service.
- This initializes all the pipeline components
- `await` = Wait for async initialization to complete

```rust
    // Keep a reference to stats_receiver before service is moved
    let reflection_service = Builder::configure()
        .register_encoded_file_descriptor_set(pb::FILE_DESCRIPTOR_SET)
        .build_v1()?;
```
**Lines 41-44**: Build the reflection service.
- Registers the protocol buffer definitions
- Allows clients to introspect available methods

```rust
    let mut grpc_routes = RoutesBuilder::default();
```
**Line 46**: Create a routes builder for gRPC.

```rust
    grpc_routes.add_service(
        pb::scored_posts_service_server::ScoredPostsServiceServer::new(service)
            .max_decoding_message_size(params::MAX_GRPC_MESSAGE_SIZE)
            .max_encoding_message_size(params::MAX_GRPC_MESSAGE_SIZE)
            .accept_compressed(CompressionEncoding::Gzip)
            .accept_compressed(CompressionEncoding::Zstd)
            .send_compressed(CompressionEncoding::Gzip)
            .send_compressed(CompressionEncoding::Zstd),
    );
```
**Lines 48-56**: Configure the main gRPC service.
- `ScoredPostsServiceServer::new(service)` = Wrap our service in a gRPC server
- `max_decoding_message_size` = Maximum size of incoming messages
- `max_encoding_message_size` = Maximum size of outgoing messages
- `accept_compressed(Gzip)` = Accept Gzip-compressed requests
- `accept_compressed(Zstd)` = Accept Zstd-compressed requests
- `send_compressed(...)` = Send responses compressed

```rust
    grpc_routes.add_service(reflection_service);
```
**Line 58**: Add the reflection service to routes.

```rust
    let grpc_config = GrpcConfig::new(args.grpc_port, grpc_routes.routes());
```
**Line 60**: Create gRPC configuration with port and routes.

```rust
    let http_router = axum::Router::default();
```
**Line 62**: Create an HTTP router (currently empty/default).
- Could be used for health checks or other HTTP endpoints

```rust
    let mut server = HttpServer::new(
        args.metrics_port,
        http_router,
        Some(grpc_config),
        CancellationToken::new(),
        Duration::from_secs(20),
    )
    .await?;
```
**Lines 64-71**: Create and configure the server.
- `args.metrics_port` = Port for metrics/health endpoints
- `http_router` = HTTP routes
- `Some(grpc_config)` = gRPC configuration
- `CancellationToken::new()` = Token for graceful shutdown
- `Duration::from_secs(20)` = Timeout settings
- `.await?` = Wait for creation, propagate any error

```rust
    server.set_readiness(true);
    info!("Server ready");
```
**Lines 73-74**: Mark server as ready and log it.
- Kubernetes and other systems check readiness before sending traffic

```rust
    server.wait_for_termination().await;
```
**Line 75**: Wait until server receives shutdown signal.
- This blocks until someone stops the server (Ctrl+C, SIGTERM, etc.)

```rust
    info!("Server shutdown complete");
    Ok(())
}
```
**Lines 76-78**: Log shutdown and return success.

## How the Server Works

```
Server Start
     │
     ├─1. Parse command-line arguments (ports, etc.)
     │
     ├─2. Initialize logging and TLS
     │
     ├─3. Create HomeMixerServer
     │    └── Initialize all pipeline components
     │        └── Connect to Phoenix, Thunder, etc.
     │
     ├─4. Configure gRPC service
     │    └── Set message sizes, compression
     │
     ├─5. Start listening on ports
     │
     ├─6. Mark as ready
     │
     └─7. Wait for shutdown signal
          │
          └── Handle requests in the meantime
```

## Key Takeaways

1. **Entry point for the service** - This is what starts when you run home-mixer
2. **Async runtime with Tokio** - Uses async/await for efficient I/O
3. **gRPC server** - Communicates via gRPC (not REST/HTTP)
4. **Compression enabled** - Supports Gzip and Zstd for efficiency
5. **Graceful shutdown** - Waits for termination signal
6. **Metrics port separate** - Health/metrics on different port from main service
