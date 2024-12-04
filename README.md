# NuGet DevLoop Sample
A simple example workflow for iterating on nuget packages built in the same solution instead of project references

## The Problem
Sometimes it's advantageous to develop against a Library project as a _NuGet Package_ Reference as opposed to a _Project_ reference.  This is hard to do if the project generating the nuget package is being consumed by a project in the same solution.  NuGet restore locations, global caches, etc all contribute to challenges, especially if you aren't incrementing the version of the package you are building each time a change is made.

## The Approach

### Directory.Build.props

We can change some settings for where packages are restored (to make it easier to clear out this cache), as well as skipping global package caches with some msbuild settings.

Additionally, we can use a target that runs after Pack which deletes the local package restore location each time the project is packaged into a nuget.  This folder needs to be deleted otherwise changes to the library project will be ignored unless the package version of the new package is changed.

You may want to consider moving this into specific projects you are building as nugets and iterating in this way, in case there are Library projects that you are _not_ and do not want this behaviour with.

This file also does a simple `Touch` operation to any of the .csproj files you have listed in the `<ConsumerProjects>` item group so that when you go to debug them, they will be changed, forcing a restore/build of the project again to capture the changes.

```xml
<Project>
  <PropertyGroup>
    <!-- Disable the use of the global NuGet cache -->
    <RestoreNoCache>true</RestoreNoCache>

    <!-- Prevent NuGet from writing to the global HTTP cache -->
    <RestoreDisableParallel>true</RestoreDisableParallel>

    <!-- Specify an alternate, isolated location for restoring packages -->
    <RestorePackagesPath>$(MSBuildThisFileDirectory)/.packages</RestorePackagesPath>

    <!-- Optional: Disable any fallback folders -->
    <RestoreFallbackFolders></RestoreFallbackFolders>
  </PropertyGroup>

  <!-- You can list out the projects you want to have restore packages again for, once the library project/nuget package has changes built -->
  <ItemGroup>
    <ConsumerProjects Include="$(MSBuildThisFileDirectory)/SomeApp/SomeApp.csproj" />
  </ItemGroup>

  <!-- After the Library is packed, delete the local cache folder so the restore happens again with the new bits of the same version -->
  <Target Name="PostPackCleanup" AfterTargets="Pack">
    <RemoveDir Directories="$(MSBuildThisFileDirectory)/.packages" />
    <Touch Files="@(ConsumerProjects)" />
  </Target>
</Project>
```


### NuGet.Config

These config settings may be redundant, but including them to be sure.  We are skipping caching, clearing out having any "Http Cache Folders" specified (so we skip that cache).

Additionally we configure the local `./artifacts` folder as a package source.

> NOTE: You may want to use package source mapping to constrain restores to the local feed where you are publishing to for the purpose of iteration, so they don't conflict restoring from stable ones published to NuGet.Org, depending on the versioning scheme you use for local dev vs publishing.

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
	<config>
		<add key="noCache" value="true" />
		<add key="httpCacheFolder" value="clear" />
	</config>
	<packageSources>
		<!-- ensure only the sources defined below are used -->
		<add key="Artifacts" value="./artifacts/" />
	</packageSources>
</configuration>
```


## Workflow

1. Build the Library project
When you make a change to the Library project, since the project has `<GeneratePackageOnBuild>True</GeneratePackageOnBuild>`, you can simply build the project which will also run the `Pack` target, which should cause the `.packages` folder to be deleted afterwards.

2. Rebuild the Consume Project(s)
To observe the changes in the project you have consuming this NuGet package, you will need to rebuild it, or you can list it in the `<ConsumerProjects>` item group inside of the `Directory.Build.props` file to have the project file automatically touched, such that it will run a restore/build when you start debugging it.


## Testing this project
1. You can test this behaviour by running `SomeApp`.  Observe that it prints out `Version: 1.2.3.4`.
2. Next, in the `SomeLibrary/SomeClass` file, change the version number (eg: to `1.2.3.5`).
3. Now, Build the `SomeLibrary` project
4. Next, run the `SomeApp` project again, and observe it now prints out `Version: 1.2.3.5`