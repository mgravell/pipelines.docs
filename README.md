# pipelines.docs

This repository is intended as a user-contributed documentation repository for the .NET "pipelines" work, currently in progress. If you have additions or changes: PR me!

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

## What do you need to play with pipelines?

You will need:

- an up to date *preview* runtime and compiler - [nightly is here](https://github.com/dotnet/cli)
- the *preview* myget feeds:

      <add key="dotnet-core" value="https://dotnet.myget.org/F/dotnet-core/api/v3/index.json" protocolVersion="3" />
      <add key="corefxlab" value="https://dotnet.myget.org/F/dotnet-corefxlab/api/v3/index.json" protocolVersion="3" />

- *for your convenience only*, version tags in your project file (so you can update everything in one go); for example (these numbers change often):

       <FrameworkTargetVer>4.5.0-preview1-26108-02</FrameworkTargetVer>
       <CoreFxLabVersion>0.1.0-e180108-4</CoreFxLabVersion>

- preview package references from corefxlab:

      <PackageReference Include="System.IO.Pipelines" Version="$(CoreFxLabVersion)" />
      <PackageReference Include="System.Buffers.Primitives" Version="$(CoreFxLabVersion)" />
        
- preview package references from dotnet-core:

      <PackageReference Include="System.Memory" Version="$(FrameworkTargetVer)" />
      <PackageReference Include="System.Runtime.CompilerServices.Unsafe" Version="$(FrameworkTargetVer)" />
      <PackageReference Include="System.Threading.Tasks.Extensions" Version="$(FrameworkTargetVer)" />
