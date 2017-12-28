---
layout: post
title: Building a Fody plugin - pt II
subtitle: Unit testing / nugetizing
---

This post continues on my [prevous Fody post](/2017-10-21-il-weaving/). This time I won't touch on the actual weaving code but rather on how to unit test your created Fody addin and how to package it up.

# Unit testing

The most important test you can do is verify if the assembly has valid IL code after weaving. This can be done with the [PEVerify](https://docs.microsoft.com/en-us/dotnet/framework/tools/peverify-exe-peverify-tool) tool which is installed together with default Visual Studio installs. The main downside of this tool is that it only runs on Windows. Because of this you need to run your (automated) tests on a Windows machine. A cross platform tool to do the same, [ILVerify](https://github.com/dotnet/corert/tree/master/src/ILVerify/), is in the works but not finished yet.

Valid PEVerify output looks like this:

```
>> peverify AssemblyToProcess2.dll

Microsoft (R) .NET Framework PE Verifier.  Version  4.0.30319.0
Copyright (c) Microsoft Corporation.  All rights reserved.

All Classes and Methods in AssemblyToProcess2.dll Verified.
```

Invalid PEVerify output looks like this:

```
>> peverify AssemblyToProcess2.dll

Microsoft (R) .NET Framework PE Verifier.  Version  4.0.30319.0
Copyright (c) Microsoft Corporation.  All rights reserved.

[IL]: Error: [C:\Users\Michiel\Documents\_code\State.Fody\State.Fody.Tests\bin\Release\AssemblyToProcess2.dll : Async::TestField][offset 0x00000026] Endfinally from outside a finally handler
[IL]: Error: [C:\Users\Michiel\Documents\_code\State.Fody\State.Fody.Tests\bin\Release\AssemblyToProcess2.dll : Default::TestField][offset 0x0000001E] Endfinally from outside a finally handler
2 Error(s) Verifying AssemblyToProcess2.dll
```

The way we verify if our weaved assembly is valid, is by comparing the PEVerify output from the original assembly with the output from our weaved assembly. If the output isn't the same an error was introduced during weaving. The output usually has very clear pointers as to what's wrong and where to fix it.
<br />
<br />

Aside from IL validation, the functionality added through weaving should also be tested. In general it's best to compile a list of all possible scenario's your weaving code could have impact on / be used in, and have a test suite which covers all of these. Unsupported scenario's should also be tested as these should fail your build to ensure users know they are in unsupported terrain. As example it could look like this for my State plugin:

Supported scenario's:

- 'normal' method linked to a boolean field or property should add state setters
- 'normal' method linked to a non-existing property should create property and add state setters
- linking to base class properties should work
- inserted try/finally blocks shouldn't break existing try/catch/finally blocks
- all of the above in generic contexts
- all of the above in static contexts
- all of the above in async contexts

Unsupported scenario's which should fail the build:

- request to change instance fields or properties from a static method isn't possible
- linked properties without setters can't be set
- linked properties which aren't booleans can't be set

Functionality which should be tested:

- creation of properties where needed (implied correct detection of existing fields/properties)
- inserted property calls should be executed when executed
- weaving should use the passed configuration correctly

I decided to not work with several test assemblies in my project but instead to have a grouping of valid/invalid C# files in my testing project and compile the test assemblies on the fly at test time. There's no big benefit to go with this approach above compiling linked assembiles aside from learning some new roslyn things :) .

My first approach used the older [CodeDomProvider](https://msdn.microsoft.com/en-us/library/system.codedom.compiler.codedomprovider(v=vs.110).aspx) instead of the newer, roslyn, [CSharpCompilation](https://docs.microsoft.com/en-us/dotnet/api/microsoft.codeanalysis.csharp.csharpcompilation?view=roslyn-dotnet) classes. When using the old provider, it's necessary to compile your assembly in a seperate appdomain in order to successfully test the weaved assembly. If you don't, the initial assembly is retained in memory and not reloaded after weaving it. Managing several appdomains and input/output between them isn't straightforward and as moving to roslyn gives you all the new C# features this is not a hard decision to make.

I copied and commented some simplified code below on how I test some of the above. Full code is on [GitHub](https://github.com/msioen/State.Fody/tree/master/State.Fody.Tests) if interested.

```csharp
  // Test unsupported scenario's
  [Test]
  public void TestInvalidPropertySetter()
  {
    // create assembly with invalid files
    var inputAssemblyPath = CreateAssemblyForFiles("InvalidPropertySetter", "FailingAssemblyFiles", "InvalidPropertySetter.cs");
    // weave generated assembly / check if expected exception is thrown
    var exception = Assert.Throws<WeavingException>(() => TestHelper.WeaveAssembly(inputAssemblyPath));
    Assert.AreEqual(EWeavingError.InvalidPropertySetter, exception.Error);
  }

  // Test supported scenario's
  public void Setup()
  {
    // create assembly from selected files
    var assemblyPath = CreateAssemblyForFiles("Default.cs", "Async.cs", "StateChange.cs");

    // copy our 'input' assembly before weaving
    // => this allows IL comparison checks + manual diffs between both assemblies
    string weavedPath = inputPath.Replace(".dll", "2.dll");
    File.Copy(assemblyPath, weavedPath, true);

    // weave our assembly
    using (var moduleDefinition = ModuleDefinition.ReadModule(assemblyPath))
    {
      // if testing a certain config, the ModuleWeaver.Config property can be set
      var weavingTask = new ModuleWeaver
      {
          ModuleDefinition = moduleDefinition
      };

      // weave and write to file
      weavingTask.Execute();
      moduleDefinition.Write(weavedPath);
    }
  }

  // 'Default' class links to 'IsTesting' property which is not specified in code => it should be added by our code
  [Test]
  public void TestPropertyCreation()
  {
    var instance = GetInstance(assembly, "Default");
    var type = (Type)instance.GetType();
    var properties = type.GetProperties();
    // Default.cs contains one property called IsLoading
    // => we expect there to be two properties if the new one is added successfully
    Assert.AreEqual(2, properties.Length);
    // Actually check if the named property exists
    Assert.IsTrue(properties.FirstOrDefault(x => x.Name == "IsTesting") != null);
  }

  // 'StateChange' class ups an int counter every time it's IsBusy state is set
  // => if a method is successfully weaved the counter should go up by two after method execution
  [Test]
  public void TestStateChange()
  {
    var instance = TestHelper.GetInstance(assembly, "StateChange");
    Assert.AreEqual(0, instance.SetterCounter);
    instance.Test();
    Assert.AreEqual(2, instance.SetterCounter);
  }

  // Get an instance of the passed type to run tests on
  public static dynamic GetInstance(Assembly assembly, string className, params object[] args)
  {
    var type = assembly.GetType(className, true);
    return Activator.CreateInstance(type, args);
  }
```
<br />

# Packaging

Most Fody addins will have two parts. One library which includes all the weaving logic and one library with classes that can be referenced when using your library such as attributes. Both parts could be combined in one library but seperating them makes for a cleaner approach. This way only the code that's really needed is included as a reference, the weaving code is picked up at build but never referenced. In many cases the referenced library can even be dereferenced when weaving is complete as the attributes don't have any more use anymore.

Before creating the nuget package you should know where Fody looks for addins. When you add Fody to your project a xml file, named FodyWeavers.xml, is added as well. Inside of this file you define which weavers should be executed at build time, what configuration they should use and in which order they build.

In the following example we request Fody to first weave with the 'state' addin with the passed properties and then weave with the 'propertychanged' addin:

```xml
<?xml version="1.0" encoding="utf-8"?>
<Weavers>
    <State CountNestedStateChanges="true" />
    <PropertyChanged />
</Weavers>
```

At build time this xml is parsed and then Fody looks for the requested addins. It looks in the following locations:

- Nuget Package directories
- SolutionDir/Tools
- Solution project named 'Weavers'

Nuget-specific, the weaving dll has to be in the root of the package with a specified name pattern: [WeaverName].Fody.dll, eg State.Fody.dll. It's also important to know that this dll should not have 'custom' dependencies as Fody doesn't support compile time loading of references. If you do need a reference to something which can't be resolved, it's possible to workaround this issue by IL merging all dependencies inside of the one dll.

If you have a seperate project with code which should be added as a reference it should be added to the project in the same way you would build a normal nuget package, ie the dll should be added in the appropriate framework folder.

The nuget package is responsible for adding/removing itself from the weavers xml file. This is done through powershell install/uninstall scripts. This might not be necessary anymore as the PropertyChanged repo seems to have removed these files but the [official documentation](https://github.com/Fody/Fody/wiki/DeployingAddinsAsNugets) still has the requirement.

Most Fody addins use the [Pepita](https://github.com/SimonCropp/Pepita) package for easier creation of the nuget package. As such you can find a lot of good examples by looking at the popular Fody addins. With Pepita you can add a build target in your csproj to bundle everything in a nuget package. This could for example look like the following:

```xml
<Target Name="NuGetBuild" AfterTargets="AfterBuild">
  <ItemGroup>
    <FilesToDelete Include="$(SolutionDir)NuGetBuild\**\*.*" />
  </ItemGroup>
  <Delete Files="@(FilesToDelete)" />
  <MakeDir Directories="$(SolutionDir)NuGetBuild" />

  <!-- Copy library dll's which should be referenced in calling projects -->
  <Copy SourceFiles="$(SolutionDir)State\bin\$(Configuration)\net452\State.dll" DestinationFolder="$(SolutionDir)NuGetBuild\lib\net452" />
  <Copy SourceFiles="$(SolutionDir)State\bin\$(Configuration)\net452\State.pdb" DestinationFolder="$(SolutionDir)NuGetBuild\lib\net452" />
  <Copy SourceFiles="$(SolutionDir)State\bin\$(Configuration)\netstandard1.0\State.dll" DestinationFolder="$(SolutionDir)NuGetBuild\lib\netstandard1.0" />
  <Copy SourceFiles="$(SolutionDir)State\bin\$(Configuration)\netstandard1.0\State.pdb" DestinationFolder="$(SolutionDir)NuGetBuild\lib\netstandard1.0" />

  <!-- Copy weaving dll to the root of the package -->
  <Copy SourceFiles="$(SolutionDir)State.Fody\bin\$(Configuration)\net452\State.Fody.dll" DestinationFolder="$(SolutionDir)NuGetBuild" />

  <!-- Copy utility files -->
  <Copy SourceFiles="$(SolutionDir)NuGet\State.Fody.nuspec" DestinationFolder="$(SolutionDir)NuGetBuild" />
  <Copy SourceFiles="$(ProjectDir)install.ps" DestinationFiles="$(SolutionDir)NuGetBuild\tools\install.ps1" />
  <Copy SourceFiles="$(ProjectDir)uninstall.ps" DestinationFiles="$(SolutionDir)NuGetBuild\tools\uninstall.ps1" />

  <!-- Make the nuget package -->
  <PepitaPackage.CreatePackageTask NuGetBuildDirectory="$(SolutionDir)NuGetBuild" MetadataAssembly="$(SolutionDir)State.Fody\bin\$(Configuration)\net452\State.Fody.dll" />
</Target>
```
<br />

# Automate all the things

As I wanted to test out some of the GitHub integrations I decided to automate both testing and packaging as well. There are several continuous integration services to choose from and most are completely free for open source projects. In this case I went with [AppVeyor](https://www.appveyor.com) as this runs on Windows which is necessary for the tests to run completely.

Most of the setup was really simple and straightforward to do. I've written down some of the issues I ran into / some things to consider.

Ensure your environment is setup correctly
: In my case I need to run my tests on Windows which AppVeyor does by default. Aside from that my project uses .NET Core functionality which means the buildbot should run on Visual Studio 2017 to build successfully. If other tools are needed during build time you might need an install script as well to pull additional tools from nuget.
<br />
<br />

Solution needs to be restored
: On AppVeyor nuget packages aren't automatically restored. You need to add a before build script executing 'nuget restore'. If running .NET Core you'll also need to execute 'dotnet restore'.
<br />
<br />

Setup tests
: AppVeyor detects most test suites automatically and will execute them correctly. You might want to use a test script instead however to have more control about how the tests are executed.
<br />
<br />

Nuget deployment
: Deploying your package is really straightforward. You can add the generated .nupkg file as a build artifact. This artifact can then be linked in a NuGet deployment step which is a default deployment provider. Deployment can be set to only run on certain branches or on custom environment variable conditions. If you don't want automatic updates to NuGet you can add a NuGet environment to your project instead. This allows you to very easily deploy a selected version from a previous build artifact.
<br />
<br />

Build issues
: Be aware that you might run into build issues which you don't have locally, for example because of certain installed sdk's. I ran into an issue with the 'System.Collections.Immutable' and 'Microsoft.CodeAnalysis.CSharp' packages. I had to downgrade the package I used to get the automated build to work while this wasn't a problem locally.

<br />
<br />
