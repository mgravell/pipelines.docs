## pipelines.docs

This repository is intended as a user-contributed documentation repository for the .NET "pipelines" work, currently in progress. If you have additions or changes: PR me!

# What do you need to play with pipelines?

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
