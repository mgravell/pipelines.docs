# pipelines.docs

This repository is intended as a user-contributed documentation repository for the .NET "pipelines" work, currently in progress. If you have additions or changes: PR me!

- [Getting started](GettingStarted)

## What are pipelines?

Pipelines are the proposed and experimental IO API for .NET - *broadly* similar to `Stream`, but intended to target a range of common problems and scenarios that make it very hard to handle *high end* performance and scalability requirements. Most *direct* usage of these APIs is probably limited to developers involved in network protocols or serialization libraries, with many more developers benefitting *indirectly*.

The core "pipelines" code is in [`corefxlab`](https://github.com/dotnet/corefxlab/), in particular the [`System.Buffers.Primitives`](https://github.com/dotnet/corefxlab/tree/master/src/System.Buffers.Primitives) and [`System.IO.Pipelines`](https://github.com/dotnet/corefxlab/tree/master/src/System.IO.Pipelines) libraries. There are a range of other "pipelines" libraries in `corefxlab`: for the most part they should be *ignored* - they are experiments playing with the API, not the proposed API helpers. Example connectors (for connecting to sockets etc) are available in the [`KestrelHttpServer`](https://github.com/aspnet/KestrelHttpServer) repository; for example [`Kestrel.Transport.Sockets`](https://github.com/aspnet/KestrelHttpServer/tree/dev/src/Kestrel.Transport.Sockets)

## What is the current status of pipelines?

The current API is usable but under constant review; issues are discussed [here](https://github.com/dotnet/corefxlab/issues). The Kestrel http server makes use of pipelines to drive a feature-complete high-performance http server.

*External* fully complete **but experimental** implementations have been completed for a range of protocols including:

- TLS
- RESP (Redis binary protocol)
- protobuf
- WebSocket (against a much earlier version of the API; not currently updated)

These external tests are intended to test and stretch a range of common use-cases.

