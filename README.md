# AnalyzerSettings from MSBuild

[![Version](https://img.shields.io/nuget/vpre/AnalyzerSettings.svg)](https://www.nuget.org/packages/AnalyzerSettings)
[![Downloads](https://img.shields.io/nuget/dt/AnalyzerSettings)](https://www.nuget.org/packages/AnalyzerSettings)
[![Build Status](https://dev.azure.com/kzu/oss/_apis/build/status/AnalyzerSettings?branchName=master)](http://build.azdo.io/kzu/oss/28)
[![License](https://img.shields.io/github/license/kzu/AnalyzerSettings.svg)](LICENSE)
[![CI NuGet Feed](https://img.shields.io/badge/nuget-ci_feed-%23004880?logo=NuGet)](https://kzu.blob.core.windows.net/nuget/index.json)


Analyzers typically need some sort of configuration to tweak how they work. 
In order to avoid inventing your own mecanism for settings, the 
[AnalyzerSettings](https://www.nuget.org/packages/AnalyzerSettings) package 
provides the basic targets to allow the settings to be driven by MSBuild 
properties and items, with full support for incremental builds and both 
design-time and compile-time analyzer settings.

Your analyzer targets can simply declare settings to persist as `AnalyzerSetting` 
items, and this package will automatically persist and expose those to your analyzers:

```xml
<!-- In your analyzer package .targets  -->
<ItemGroup>
    <AnalyzerSetting Include="SingleFileCodeGeneration" Value="$([MSBuild]::ValueOrDefault('$(SingleFileCodeGeneration)', 'true'))" />
</ItemGroup>
```

Note how you can even leverage `ValueOrDefault` to easily provide sensible defaults for your 
analyzer setting. Of course, any of the built-in 
[MSBuild property functions](https://docs.microsoft.com/en-us/visualstudio/msbuild/property-functions?view=vs-2019#msbuild-property-functions) can similarly be used.

Consuming the settings is equally straightforward, and requires no dependency with this package 
from your analyzer:

```csharp
public class MyCodeGenAnalyzer : DiagnosticAnalyzer
{
    public override void Initialize(AnalysisContext context)
    {
        // Just to show *all* analyzer contexts are supported via their AnalyzerOptions
        context.RegisterCompilationAction(AnalyzeCompilation);
        context.RegisterSemanticModelAction(AnalyzeSemanticModel);
        context.RegisterOperationAction(AnalyzeOperation/*, params OperationKind[] */);
        context.RegisterSymbolAction(AnalyzeSymbol /*, params SymbolKind[] */);
        context.RegisterSyntaxNodeAction(AnalyzeSyntaxNode/*, params SyntaxKind[] */);
    }

    // Simply converts the additional file that ends in `AnalyzerSettings.ini` into a Dictionary<string, string>
    private Dictionary<string, string> GetAnalyzerSettings(AnalyzerOptions options)
    {
        var settingsFile = options.AdditionalFiles.FirstOrDefault(x => x.Path.EndsWith("AnalyzerSettings.ini", StringComparison.OrdinalIgnoreCase));
        if (settingsFile == null)
            return new Dictionary<string, string>();

        // Added some extra precautions in parsing each line.
        // You could also filter lines for settings starting with your known ones only...
        return settingsFile
            .GetText().Lines[0].ToString()
            .Where(line => !string.IsNullOrEmpty(line))
            .Select(line => line.Split('='))
            .Where(pair => pair.Length == 2)
            .ToDictionary(pair => pair[0].Trim(), pair => pair[1].Trim());
    }

    private void AnalyzeCompilation(CompilationAnalysisContext context)
    {
        var settings = GetAnalyzerSettings(context.Options);
        var useSingleFile = settings.TryGetValue("SingleFileCodeGeneration", out var singleFileString) &&
            bool.TryParse(singleFileString, out var singleFile) && singleFile;
        // ... 
    }

    private void RegisterSemanticModelAction(SemanticModelAnalysisContext context)
    {
        var settings = GetAnalyzerSettings(context.Options);
        // ... 
    }

    private void RegisterOperationAction(OperationAnalysisContext context)
    {
        var settings = GetAnalyzerSettings(context.Options);
        // ... 
    }
    
    private void RegisterSymbolAction(SymbolAnalysisContext context)
    {
        var settings = GetAnalyzerSettings(context.Options);
        // ... 
    }

    private void RegisterSyntaxNodeAction(SyntaxNodeAnalysisContext context)
    {
        var settings = GetAnalyzerSettings(context.Options);
        // ... 
    }    
}
```

Unless disabled by setting the MSBuild property `EnableDefaultOutputPathsAnalyzerSettings` to `false`, all output properties from the [.NET SDK](https://github.com/dotnet/sdk/blob/master/src/Tasks/Microsoft.NET.Build.Tasks/targets/Microsoft.NET.DefaultOutputPaths.targets) are included as analyzer settings since they are generally useful.