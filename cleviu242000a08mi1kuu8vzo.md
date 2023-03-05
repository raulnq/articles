---
title: "Reducing package size in .NET with Runtime Package Store"
datePublished: Sun Mar 05 2023 15:01:25 GMT+0000 (Coordinated Universal Time)
cuid: cleviu242000a08mi1kuu8vzo
slug: reducing-package-size-in-net-with-runtime-package-store
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1678020910122/438531ec-d429-47af-a371-bcaa142fbc75.png
tags: net

---

When deploying a .NET application, we typically encounter two issues.

* Including dependencies used by the application increases the size of the deployment package.
    
* By default, the packages are not compiled into native code.
    

To resolve those issues, there is an "unknown " feature that was introduced with .NET Core 2.0 named [Runtime Package Store](https://learn.microsoft.com/en-us/dotnet/core/deploying/runtime-store). This feature allows us to package and deploy our application against a set of packages that already exist in our environment, thereby reducing the size of the deployment package and improving the performance of the application. By utilizing the Runtime Package Store, we can ensure that all the dependencies used by the application are compiled into native code, resulting in a faster and more efficient deployment process (it seems that this is only valid up to .NET 5, we'll soon see why).

To implement the most basic usage scenario of the Runtime Package Store, we can do a "pre-deploy" of the package store into a server and point all the applications deployed against it. The first step is to identify the packages required by your application and create a manifest (a regular `.csproj` file). Create the `packages.csproj` file with the following content:

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net6.0</TargetFramework>
  </PropertyGroup>
  <ItemGroup>
	<PackageReference Include="Newtonsoft.Json" Version="13.0.2" />
  </ItemGroup>
</Project>
```

Run the following command to provision the Runtime Package Store:

```bash
dotnet store --manifest packages.csproj --skip-optimization --runtime win10-x64 --output ./store
```

The `dotnet store` command can use the following parameters:

* `--framework:` The specific target framework (in our case was defined in the file). Possible values are: `net7.0`, `net6.0`, `net5.0`, `netstandard2.1`, `netcoreapp3.1`, `net48`.
    
* `--manifest`: Specifies the path to the manifest (`*.csproj` file).
    
* `--runtime:` The target platforms where the application runs. For instance, `linux-x64`, `win10-x64`, or `osx.10.12-x64` (full list [here](https://learn.microsoft.com/en-us/dotnet/core/rid-catalog)).
    
* `--output`: Specifies the path to the Runtime Package Store. By default, the output of the command is under the `.dotnet/store` subdirectory of the user's profile.
    
* `--framework-version`: This option enables you to select a specific framework version.
    
* `--skip-optimization`: Skip the compilation into native code. In our case, we use `skip-optimization` because the Crossgen tool was available until `net5.0` ([https://github.com/dotnet/sdk/issues/24752](https://github.com/dotnet/sdk/issues/24752)).
    

The result of the command is a folder structure like this:

```bash
|-- store
|   |-- x64
|   |   |-- net6.0
|   |   |   |-- newtonsoft.json
|   |   |   `-- artifact.xml
|   |    `--
|    `--
`--
```

In the root of the target platform, we can find an `artifact.xml` file with the list of all the packages in our store:

```xml
<StoreArtifacts>
  <Package Id="Newtonsoft.Json" Version="13.0.2" />
</StoreArtifacts>
```

Our Runtime Package Store is ready. Time to create the application that will use it:

```bash
dotnet new console -n HelloWorldSerializer
dotnet new sln -n DotnetStoreSandbox
dotnet sln add --in-root .\HelloWorldSerializer
dotnet add .\HelloWorldSerializer package Newtonsoft.Json --version 13.0.2
```

Update the `Program.cs` file as follows:

```csharp
using Newtonsoft.Json;

var json = JsonConvert.SerializeObject(new Message() { Value = "Hello world Runtime Package Store" });

Console.WriteLine(json);

Console.ReadLine();

public class Message
{
    public string Value { get; set; } = null!;
}
```

Run `dotnet publish .\HelloWorldSerializer\ --configuration Release --runtime win10-x64 --no-self-contained --output publish` to see the regular output (the `Newtonsoft.Json.dll` is present):

```bash
|-- publish
|   |-- HelloWorldSerializer.deps.json
|   |-- HelloWorldSerializer.dll
|   |-- HelloWorldSerializer.exe
|   |-- HelloWorldSerializer.pdb
|   |-- HelloWorldSerializer.runtimeconfig.json
|   `-- Newtonsoft.Json.dll
`-- 
```

Run `dotnet publish .\HelloWordSerializer\ --configuration Release --runtime win10-x64 --no-self-contained --manifest .\store\x64\net6.0\artifact.xml --output publish` :

```bash
|-- publish
|   |-- HelloWorldSerializer.deps.json
|   |-- HelloWorldSerializer.dll
|   |-- HelloWorldSerializer.exe
|   |-- HelloWorldSerializer.pdb
|   `-- HelloWorldSerializer.runtimeconfig.json
`-- 
```

The `Newtonsoft.Json.dll` has been gone. Run the `HelloWorldSerializer.exe`, and we will see the following error:

```bash
Unhandled exception. System.IO.FileNotFoundException: Could not load file or assembly 'Newtonsoft.Json, Version=13.0.0.0, Culture=neutral, PublicKeyToken=30ad4fe6b2a6aeed'. The system cannot find the file specified.
File name: 'Newtonsoft.Json, Version=13.0.0.0, Culture=neutral, PublicKeyToken=30ad4fe6b2a6aeed'
   at Program.<Main>$(String[] args)
```

The problem is that the application does not know where is located our Runtime Package Store. Set the `DOTNET_SHARED_STORE` environment variable with the corresponding location, `$env:DOTNET_SHARED_STORE="C:\Source\dotnet-store\store"`. Rerun the application (under the same session in which the environment variable was set):

```json
{"Value":"Hello world Runtime Package Store"}
```

Now, we see the right output on the screen. As you can see there is a lot of potential for what we can do with Runtime Package Store as this other [post](https://www.fearofoblivion.com/using-dotnet-runtime-package-stores-to-optimize-docker-images) shows or be used as the base of other tools like the [AWS Lambda layers with .NET Core](https://aws.amazon.com/es/blogs/developer/aws-lambda-layers-with-net-core/). Thanks, and happy coding.