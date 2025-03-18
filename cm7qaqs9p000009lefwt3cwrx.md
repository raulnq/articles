---
title: "How to Build Multi-Architecture Container Images for .NET Applications"
datePublished: Sat Mar 01 2025 14:27:32 GMT+0000 (Coordinated Universal Time)
cuid: cm7qaqs9p000009lefwt3cwrx
slug: how-to-build-multi-architecture-container-images-for-net-applications
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1740765654536/e97bab82-d82c-4877-8d64-3272f3c2e465.png
tags: docker, dotnet, arm, podman, x86

---

Containers are increasingly popular due to their ability to package software as images that can run anywhere. This concept is exciting but poses challenges when dealing with multiple architectures. Fortunately, features like multi-architecture support in Docker and cross-platform compilation in .NET applications help us overcome these challenges with a few additional steps.

## Pre-requisites

* Install [**Docker Desktop**](https://docs.docker.com/desktop/install/windows-install/)
    

## Create Dockerfile

We will use the `webapi` template to create our application. Run the following commands to set up the solution:

```powershell
dotnet new webapi -n MyWebApi
dotnet new sln -n Sandbox
dotnet sln add --in-root MyWebApi
```

Create a `Dockerfile` in the project directory with the following content:

```dockerfile
FROM --platform=$TARGETPLATFORM mcr.microsoft.com/dotnet/aspnet:8.0 AS runtime
USER app
WORKDIR /app
EXPOSE 8080
EXPOSE 8081

FROM --platform=$BUILDPLATFORM mcr.microsoft.com/dotnet/sdk:8.0 AS build
ARG BUILD_CONFIGURATION=Release
ARG BUILDPLATFORM
ARG TARGETARCH
ARG TARGETOS
WORKDIR /src
COPY ["MyWebApi/MyWebApi.csproj", "MyWebApi/"]
RUN dotnet restore "./MyWebApi/MyWebApi.csproj" -a $TARGETARCH 
COPY . .
RUN echo "Building on $BUILDPLATFORM, targeting $TARGETOS/$TARGETARCH"

WORKDIR "/src/MyWebApi"

FROM build AS publish
ARG BUILD_CONFIGURATION=Release
RUN dotnet publish -a $TARGETARCH --no-restore "./MyWebApi.csproj" -c $BUILD_CONFIGURATION -o /app/publish /p:UseAppHost=false

FROM runtime AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "MyWebApi.dll"]
```

Docker provides arguments to specify the build and target platforms:

* The build arguments are related to the host where the builder runs (`BUILDPLATFORM`, `BUILDOS`, `BUILDARCH`, and `BUILDVARIANT`).
    
* The target arguments include the values set with the `--platform` flag in the `build` command (`TARGETPLATFORM`, `TARGETOS`, `TARGETARCH`, and `TARGETVARIANT`).
    

These arguments are available in the global scope of the `Dockerfile` but not within the build stages. To use these arguments inside a stage, declare them as arguments, and Docker will automatically set their values. Here's how we can use them:

* The `FROM` command uses the `--platform` flag to specify the image's platform, such as `linux/amd64`, `linux/arm64`, or `windows/amd64`. In the `Dockerfile` above, the image used for building the application includes the `--platform=$BUILDPLATFORM` flag to be aligned with the host's platform and avoid any issues. The image used for running the application includes the `--platform=$TARGETPLATFORM` flag to match the desired platform.
    
* Additionally, the `dotnet publish` and `dotnet restore` commands use the `-a $TARGETARCH` option to define the target architecture, such as `arm64` or `amd64`.
    

## Build Images

To build an image for each target platform, execute the following command at the solution level:

```powershell
docker build --platform linux/amd64 --tag raulnq/mywebapi:amd64 --file .\MyWebApi\Dockerfile .
docker build --platform linux/arm64 --tag raulnq/mywebapi:arm64 --file .\MyWebApi\Dockerfile .
```

The `--platform` flag specifies the target platform for the build. As previously mentioned, it only passes the value to the `Dockerfile`, which performs the work. By default, the value is set to the host's platform where the build runs, formatted as `os/arch` or `os/arch/variant`.

## Create Manifest

A manifest is a JSON file that describes a set of images for different platforms, collectively known as a manifest list. It allows multiple images to be associated with a single reference. When a container is deployed using this manifest, the correct image is selected based on the host's platform. To create the manifest, use the following command:

```powershell
 docker manifest create raulnq/webapi:1.0 raulnq/mywebapi:amd64 raulnq/mywebapi:arm64
```

We can inspect the manifest's content using the command `docker manifest inspect raulnq/webapi:1.0`:

```json
{
    "schemaVersion": 2,
    "mediaType": "application/vnd.oci.image.index.v1+json",
    "manifests": [
        {
            "mediaType": "application/vnd.oci.image.manifest.v1+json",
            "size": 1522,
            "digest": "sha256:43ab8996af783470032926fb6c63b25313174d50b360a78f9e6f70344a706df3",
            "platform": {
                "architecture": "amd64",
                "os": "linux"
            }
        },
        {
            "mediaType": "application/vnd.oci.image.manifest.v1+json",
            "size": 1523,
            "digest": "sha256:a095526a10b3c981fbe17bb7f2a2cb1e236989463bcfe3eb0b5df654c351e0b4",
            "platform": {
                "architecture": "arm64",
                "os": "linux"
            }
        }
    ]
}
```

This allows us to verify the size and digest for each image manifest included in the manifest.

## Push manifest

As the final step, we can push the manifest to our preferred container registry using the following command:

```powershell
docker manifest push raulnq/webapi:1.0
```

It is worth mentioning that all the commands showed today are fully compatible with [Podman](https://podman.io/). Thank you, and happy coding.