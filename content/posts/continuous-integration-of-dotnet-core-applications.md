---
title: "Continuous integration of .NET Core applications"
date: 2017-03-21
tags:
  - csharp
  - dotnet
  - continuous-integration
url: /2017/03/21/continuous-integration-of-dotnet-core-applications/
---

On March 7, 2017, the [.NET Core SDK](https://www.microsoft.com/net/download/core) was released. It consists of two parts: the [.NET Core runtime](https://docs.microsoft.com/en-us/dotnet/articles/core/) and the [.NET CLI](https://docs.microsoft.com/en-us/dotnet/articles/core/tools/). The runtime allows you to run .NET Core applications, whereas the CLI is a command-line interface that allows you to develop .NET Core applications.

In this blog post, we'll see how we can use the .NET CLI to build a small .NET Core console application. We then show how easy it is to setup a continuous integration pipeline using AppVeyor, Travis and CircleCI.

Note: we will assume that we already have an existing account for AppVeyor, Travis and CircleCI.

## Installing the .NET Core SDK

Our first step is to install the .NET Core SDK. This is surprisingly simple. Just go to the [download](https://www.microsoft.com/net/download/core) page and follow the on-screen instructions.

Once the installation has completed, open a command prompt and run:

```bash
dotnet --version
```

If the installation was successful, the .NET CLI's version number will be displayed, which for me was:

```bash
1.0.1
```

## Creating the application

The application we'll build will be a small console application. Although we could use Visual Studio (2017) to build the application, we'll use the .NET CLI exclusively.

First, we'll create a directory for our application and navigate to that directory:

```bash
mkdir dotnetcore-ci
cd dotnetcore-ci
```

To quickly generate a basic console application, we can use the .NET CLI:

```bash
dotnet new console
```

This will output:

```bash
Content generation time: 47.7522 ms
The template "Console Application" created successfully.
```

Once finished, the CLI will have created two files in the current directory:

- `Program.cs`: the C# source file for our application.
- `dotnetcore-ci.csproj`: a C# project for our application.

These two files is all the .NET CLI needs to build and run our application. Before we can run our application though, we need to restore its dependencies:

```
dotnet restore
```

This command will resolve our project's dependencies:

```bash
Restoring packages for /Users/erikschierboom/Programming/dotnetcore-ci/dotnetcore-ci.csproj...
  Generating MSBuild file /Users/erikschierboom/Programming/dotnetcore-ci/obj/dotnetcore-ci.csproj.nuget.g.props.
  Generating MSBuild file /Users/erikschierboom/Programming/dotnetcore-ci/obj/dotnetcore-ci.csproj.nuget.g.targets.
  Writing lock file to disk. Path: /Users/erikschierboom/Programming/dotnetcore-ci/obj/project.assets.json
  Restore completed in 841.1 ms for /Users/erikschierboom/Programming/dotnetcore-ci/dotnetcore-ci.csproj.

  NuGet Config files used:
      /Users/erikschierboom/.nuget/NuGet/NuGet.Config

  Feeds used:
      https://api.nuget.org/v3/index.json
```

Having restored our project's dependencies, we can now build our application:

```bash
dotnet build
```

This will output some diagnostic information:

```bash
Microsoft (R) Build Engine version 15.1.548.43366
Copyright (C) Microsoft Corporation. All rights reserved.

  dotnetcore-ci -> /Users/erikschierboom/Programming/dotnetcore-ci/bin/Debug/netcoreapp1.1/dotnetcore-ci.dll

Build succeeded.
    0 Warning(s)
    0 Error(s)

Time Elapsed 00:00:02.14
```

We can see that the build was successful, so let's run our application using the .NET CLI:

```bash
dotnet run
```

Our application will now be run (using the .NET Core runtime) and will output a familiar message:

```bash
Hello World!
```

The output may not be that exciting, but what _is_ exciting is that we scaffolded, restored, built _and_ ran our application using just the .NET CLI! No IDE was involved! Additionally, we did all this on our MacBook, showing the fully cross-platform nature of the .NET CLI.

## Building on a continuous integration server

Now that we have our software building locally, let's try to setup a continuous integration pipeline. For that, we need to create a [public repository](https://github.com/ErikSchierboom/dotnetcore-ci) on GitHub for our application. Having pushed our local files to the public repository, we are ready to add continuous integration to our application.

### AppVeyor

The first CI server we'll look at is [AppVeyor](https://www.appveyor.com/), which is a CI server that runs on Windows.

To configure our application, we add an `appveyor.yml` file to our root directory. Its contents are very simple:

```yaml
image: Visual Studio 2017
environment:
  DOTNET_CLI_TELEMETRY_OPTOUT: true
  DOTNET_SKIP_FIRST_TIME_EXPERIENCE: true
before_build:
  - cmd: dotnet restore
```

In the first line, we specify that we want to use the Visual Studio 2017 image, which is the AppVeyor image that has the .NET Core SDK pre-installed.

Next, we set two environment variables:

1. `DOTNET_CLI_TELEMETRY_OPTOUT: true`: don't send any telemetry data.
1. `DOTNET_SKIP_FIRST_TIME_EXPERIENCE: true`: this will prevent the CLI from pre-populating the packages cache.

We then indicate that `dotnet restore` should be executed before the application is built. Note that we don't have to call `dotnet build`, as AppVeyor will automatically do that for us. You can see this when we examine the build output in AppVeyor:

```bash
Build started
git clone -q --branch=master https://github.com/ErikSchierboom/dotnetcore-ci.git C:\projects\dotnetcore-ci
git checkout -qf c6a418e057f44d9604e95429ec273239ef8c01bc

dotnet restore
  Restoring packages for C:\projects\dotnetcore-ci\dotnetcore-ci.csproj...
  Generating MSBuild file
   ...

msbuild "C:\projects\dotnetcore-ci\dotnetcore-ci.csproj" /logger:"C:\Program Files\AppVeyor\BuildAgent\Appveyor.MSBuildLogger.dll"
Microsoft (R) Build Engine version 15.1.548.43366
Copyright (C) Microsoft Corporation. All rights reserved.
Build started 3/20/2017 2:21:46 PM.
Project "C:\projects\dotnetcore-ci\dotnetcore-ci.csproj" on node 1 (default targets).
PrepareForBuild:
  ...
CoreCompile:
  ...
CopyFilesToOutputDirectory:
  ...
Done Building Project "C:\projects\dotnetcore-ci\dotnetcore-ci.csproj" (default targets).
Build succeeded.
    0 Warning(s)
    0 Error(s)
Time Elapsed 00:00:03.27

Discovering tests...OK
Build success
```

And with that, we have our .NET Core application building on AppVeyor using the .NET CLI.

### Travis

Our second CI server is [Travis](https://travis-ci.org/), which runs on Linux.

Just like AppVeyor, we need to add a configuration file to our root directory, this time named `.travis.yml`:

```yaml
language: csharp
dist: trusty
dotnet: 1.0.1
mono: none
env:
  global:
    - DOTNET_SKIP_FIRST_TIME_EXPERIENCE: 1
    - DOTNET_CLI_TELEMETRY_OPTOUT: 1
script:
  - dotnet restore
  - dotnet build
```

This config file has a bit more going on, but nothing complicated. First, we define the language we'll be using, C# in our case. Next, we define the Linux distro it runs on, for which we'll use Ubunty trusty.

We then get to the .NET specific bits, in which we indicate that we want version 1.0.1 of the .NET Core SDK to be installed. We also explicitly specify that we don't require Mono, which would otherwise be automatically installed due to the language specified being `csharp`.

Once again, we specify the two .NET CLI specific environment variables.

Finally, we specify the commands to execute, and this time we do need both the `dotnet restore` and `dotnet build` commands.

The Travis output log will look like this:

```bash
Installing .NET Core
..

git clone --depth=50 --branch=master https://github.com/ErikSchierboom/dotnetcore-ci.git
...

Setting environment variables from .travis.yml
$ export DOTNET_SKIP_FIRST_TIME_EXPERIENCE=1
$ export DOTNET_CLI_TELEMETRY_OPTOUT=1

...

dotnet restore
  Restoring packages for C:\projects\dotnetcore-ci\dotnetcore-ci.csproj...
  Generating MSBuild file
   ...

dotnet build
Copyright (C) Microsoft Corporation. All rights reserved.
e/travis/build/ErikSchierboom/dotnetcore-ci/bin/Debug/netcoreapp1.1/dotnetcore-ci.dll
Build succeeded.
    0 Warning(s)
    0 Error(s)
Time Elapsed 00:00:05.60

The command "dotnet build" exited with 0.
Done. Your build exited with 0.
```

The output is very similar to AppVeyor, with only minor differences. Our continuous integration pipeline now verifies that our application builds both on Windows (AppVeyor) and Linux (Travis).

### CircleCI

The last CI server we'll look at is [CircleCI](https://circleci.com), which is also Linux-based.

Once again, configuration is done by adding a file to the root directory, this time named `circle.yml`:

```yaml
version: 2
jobs:
  build:
    working_directory: /temp
    docker:
      - image: microsoft/dotnet:sdk
    environment:
      DOTNET_SKIP_FIRST_TIME_EXPERIENCE: 1
      DOTNET_CLI_TELEMETRY_OPTOUT: 1
    steps:
      - checkout
      - run: dotnet restore
      - run: dotnet build
```

In the first line, we specify that we want to use version 2 of CircleCI, which is currently in beta. With version 2, builds are based on Docker images. That allows us to use the official .NET Core SDK [Docker image](https://store.docker.com/images/6c038a68-be47-4d7e-bfd2-33a6fe75b9ac?tab=description) to build our application.

Once again, we also set the .NET CLI environment variables.

Finally, we specify the steps to execute. First, we checkout our repository and then we run `dotnet restore` and `dotnet build`.

The build output is similar to that of the other two CI servers:

```bash
Build-agent version 0.0.2792-4fbb67c (2017-03-20T15:14:26+0000)
Starting container microsoft/dotnet:sdk
  image not cached, downloading microsoft/dotnet:sdk
sdk: Pulling from microsoft/dotnet
...
Status: Downloaded newer image for microsoft/dotnet:sdk
  using image microsoft/dotnet@sha256:3f87a7b7873ced0110a35aee48591f8b4aea3ccb94bf433a80388886bdbd3076

...

git fetch --force origin master:remotes/origin/master
...
Cloning into '.'...
...
HEAD is now at df83619 Add CircleCI support

dotnet restore
Shell: /bin/bash -eo pipefail

  Restoring packages for /tmp/dotnetcore-ci.csproj...
  Generating MSBuild file /tmp/obj/dotnetcore-ci.csproj.nuget.g.props.
  ...

dotnet build

Microsoft (R) Build Engine version 15.1.548.43366
Copyright (C) Microsoft Corporation. All rights reserved.

  dotnetcore-ci -> /tmp/bin/Debug/netcoreapp1.1/dotnetcore-ci.dll

Build succeeded.
    0 Warning(s)
    0 Error(s)

Time Elapsed 00:00:01.94
```

## Conclusion

Creating, building and running a .NET Core application using the .NET CLI is incredibly easy. It it also quite convenient, as it is cross-platform and you can build your application using the same CLI both locally and remotely.

We used the CLI locally to create a simple .NET Core console application. We then used the CLI remotely to setup three continuous integration servers, AppVeyor, Travis and CircleCI, to build our application.

Configuring the CI servers was very simple: they all required only a single, simple YAML configuration file to be added to the root of our project.

The source code of this test application, which includes the three CI server configuration files, can be found [here](https://github.com/ErikSchierboom/dotnetcore-ci).
