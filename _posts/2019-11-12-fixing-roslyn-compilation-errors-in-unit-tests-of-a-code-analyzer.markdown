---
layout: post
title:  "Fixing Roslyn compilation errors in unit tests of a code analyzer"
date:   2019-11-12 08:46:08 +0100
categories: roslyn .net
---

I’m dabbling in creating Roslyn analyzers and soon ran into some problems with Visual Studio’s _Analyzer with Code Fix (.NET Standard)_ project template. In this post I will explain how to find the compilation errors in your unit tests, and how to fix two common compilation errors.

## Checking if you have compilation errors

To check for compilation errors in the code under test, open the `DiagnosticVerifier.GetSortedDiagnosticsFromDocuments` method. In it, replace the `compilationWithAnalyzers` variable with the following code and then observe the `diagnostics` variable while debugging:

{% highlight csharp %}
var compilation = project.GetCompilationAsync().Result;
var diagnostics = compilation.GetDiagnostics();
var compilationWithAnalyzers = compilation.WithAnalyzers(
    ImmutableArray.Create(analyzer));
{% endhighlight %}

## Fixing missing netstandard references

In my case, I ran into the following error:

> error CS0012: The type ‘IDisposable’ is defined in an assembly that is not referenced. You must add a reference to assembly ‘netstandard, Version=2.0.0.0, Culture=neutral, PublicKeyToken=cc7b13ffcd2ddd51’.

Other people report similar issues with e.g. the `Object` type or other basic types. Most of these are caused by the _Analyzer with Code Fix (.NET Standard)_ project template not adding some basic references. To fix these issues, I modified the `DiagnosticVerifier.CreateProject` method by adding the following code:

{% highlight csharp %}
solution = AddRuntimeLibrary(solution, projectId, "netstandard.dll");
solution = AddRuntimeLibrary(solution, projectId, "System.Runtime.dll");
{% endhighlight %}

As `netstandard.dll` has a reference to `System.Runtime.dll` I needed to add both to solve my compilation error. The `AddRuntimeLibrary` is implemented like this:

{% highlight csharp %}
private static Solution AddRuntimeLibrary(Solution solution, 
    ProjectId projectId, string fileName)
{
    var runtimeDirectory = System.Runtime.InteropServices.RuntimeEnvironment
        .GetRuntimeDirectory();
    var dll = Path.Combine(runtimeDirectory, fileName);

    return solution.AddMetadataReference(projectId, 
        MetadataReference.CreateFromFile(dll));
}
{% endhighlight %}

## Fixing missing Main method

It’s quite likely your tests don’t care about the presence of a `Main` method. The project that is created by default within an `AdhocWorkspace` is of type `ConsoleApplication`. This thus leads to the following compilation error:

> error CS5001: Program does not contain a static ‘Main’ method suitable for an entry point

To solve this problem. Modify the project’s `CompilationOptions.OutputKind` to be a `DynamicallyLinkedLibrary` (more commonly known in the .NET world as class library). For this you can use the following piece of code:

{% highlight csharp %}
solution = solution.WithProjectCompilationOptions(projectId,
    solution.GetProject(projectId).CompilationOptions
        .WithOutputKind(OutputKind.DynamicallyLinkedLibrary));
{% endhighlight %}

## Comments

Post your comments [here](https://gist.github.com/davidwalschots/ddb65707744b9191603d8c742495778f).
