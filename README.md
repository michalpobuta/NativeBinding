# NativeBinding

Automated native SDK binding generator for .NET MAUI. This project uses an AI agent (Claude Code) to fully automate the creation of .NET MAUI native bindings from iOS and Android SDKs — including artifact discovery, API research, code generation, build verification, and testing.

## How It Works

The agent is driven by a custom Claude Code command (`/create-binding`) defined in `.claude/commands/create-binding.md`. When invoked, it runs through 8 phases autonomously:

### Phase 1: Setup
Creates the output directory structure with `ios/`, `android/`, `unified/`, and `tests/` subdirectories.

### Phase 2: Fetch Artifacts
- **Android**: Searches Maven Central API, downloads the AAR (and companion core JARs for Kotlin SDKs), verifies the archive.
- **iOS**: Searches GitHub releases for a pre-built XCFramework. If none exists, clones the source repo and builds from source using `swift build` with explicit iOS triples, packages `.o` files into static libraries with `ar`, and creates an XCFramework via `xcodebuild -create-xcframework`.

### Phase 3: Research API
Reads ObjC headers from the XCFramework and inspects Android AAR class files with `javap`. Cross-references with official SDK documentation to build a complete API model.

### Phase 4: Generate Bindings
- **iOS**: Generates `ApiDefinition.cs` with `[BaseType]`, `[Export]`, `[Protocol]`, `[Static]`, `[NullAllowed]` attributes mapped from ObjC headers. Generates `StructsAndEnums.cs` for enums. Creates the `.csproj` with `IsBindingProject`, `ObjcBindingApiDefinition`, `ObjcBindingCoreSource`, and `NativeReference`.
- **Android**: Generates `Transforms/Metadata.xml` for Java-to-C# namespace mapping and removal of internal/problematic types. Creates the `.csproj` with `TransformFile` and `AndroidLibrary` references.

### Phase 5: Unified MAUI API
Creates a cross-platform C# interface (`IAmplitudeService`) with platform-specific implementations under `Platforms/iOS/` and `Platforms/Android/`. Includes a `MauiAppBuilder` extension method for DI registration.

### Phase 6: Generate Tests
Creates xUnit tests that validate the generated source files as text — checking file existence, namespace declarations, required attributes, API surface presence, and project structure. Tests target `net10.0` (not platform-specific TFMs) so they run on any machine.

### Phase 7: Build & Verify
Builds all projects in dependency order, runs tests, and iteratively fixes any errors (up to 5 attempts). Common fixes are applied automatically: namespace clashes, Metadata.xml adjustments, missing dependencies, type conversion issues.

### Phase 8: Report
Prints a summary with status, files created, build results, test results, and next steps for integration.

## Usage

```
/create-binding <sdk-name> <platform: ios|android|both> [output-dir]
```

Examples:
```
/create-binding Amplitude both ./output/amplitude
/create-binding Firebase ios ./output/firebase
/create-binding Lottie android ./output/lottie
```

## Example Output: Amplitude SDK

The `output/amplitude/` directory contains a complete binding for the [Amplitude](https://amplitude.com) analytics SDK targeting both iOS and Android.

### Generated Files

```
output/amplitude/
├── android/
│   ├── Additions/Additions.cs              # Manual binding additions
│   ├── Amplitude.Binding.Android.csproj    # Android binding project
│   ├── Transforms/
│   │   ├── EnumFields.xml                  # Java constant → C# enum mappings
│   │   └── Metadata.xml                    # Java package → C# namespace mappings
│   ├── analytics-android-1.21.3.aar        # Amplitude Android SDK artifact
│   └── analytics-core-1.21.3.jar           # Core types (Configuration, Events, etc.)
├── ios/
│   ├── Amplitude.Binding.iOS.csproj        # iOS binding project
│   ├── Amplitude.xcframework/              # Built from source (v8.22.2)
│   │   ├── ios-arm64/                      # Device slice (static library + headers)
│   │   └── ios-arm64-simulator/            # Simulator slice (static library + headers)
│   ├── ApiDefinition.cs                    # ObjC → C# binding definitions
│   └── StructsAndEnums.cs                  # Enums (AMPServerZone)
├── unified/
│   ├── Amplitude.Maui.csproj              # Cross-platform MAUI project
│   ├── IAmplitudeService.cs               # Platform-agnostic C# interface
│   ├── AmplitudeServiceExtensions.cs      # DI registration (.UseAmplitude())
│   └── Platforms/
│       ├── Android/AmplitudeService.cs    # Android implementation
│       └── iOS/AmplitudeService.cs        # iOS implementation
└── tests/
    ├── Amplitude.Binding.Tests.csproj     # xUnit test project
    ├── CompilationTests.cs                # Source file validity checks
    ├── ApiPresenceTests.cs                # API surface verification
    └── SmokeTests.cs                      # Structure & attribute checks
```

### Build & Test Results

```
Status: SUCCESS

Build Results:
  ✓ Amplitude.Binding.iOS.csproj      — Built successfully
  ✓ Amplitude.Binding.Android.csproj  — Built successfully
  ✓ Amplitude.Maui.csproj             — Built successfully (both TFMs)
  ✓ Amplitude.Binding.Tests.csproj    — Built successfully

Test Results: 61 passed, 0 failed, 0 skipped

  CompilationTests:
    ✓ iOSApiDefinition_Parses_As_Valid_CSharp
    ✓ iOSStructsAndEnums_Parses_As_Valid_CSharp
    ✓ AndroidMetadata_Is_Valid_Xml
    ✓ UnifiedInterface_Parses_As_Valid_CSharp
    ✓ UnifiediOSImpl_Parses_As_Valid_CSharp
    ✓ UnifiedAndroidImpl_Parses_As_Valid_CSharp
    ... (18 tests)

  ApiPresenceTests:
    ✓ iOS_Has_Amplitude_Class
    ✓ iOS_Has_AMPIdentify_Class
    ✓ iOS_Has_AMPRevenue_Class
    ✓ iOS_Has_LogEvent_Method
    ✓ Android_Has_Amplitude_Namespace
    ✓ Android_Has_Revenue_Mapping
    ✓ Unified_Interface_Has_Initialize
    ✓ Unified_Interface_Has_LogEvent
    ✓ Unified_Interface_Has_LogRevenue
    ... (32 tests)

  SmokeTests:
    ✓ iOSApiDefinition_Has_Correct_Namespace
    ✓ iOSApiDefinition_Uses_Required_Attributes
    ✓ iOSApiDefinition_Uses_Required_Usings
    ✓ iOSCsproj_IsBindingProject
    ✓ AndroidCsproj_HasCorrectStructure
    ✓ AndroidMetadata_HasPackageMappings
    ✓ UnifiedCsproj_HasCorrectStructure
    ✓ UnifiedDIExtensions_Exists_And_Registers_Service
    ✓ XcFramework_Exists_InOutputDirectory
    ✓ AndroidAar_Exists_InOutputDirectory
    ... (11 tests)
```

### How to Use in a MAUI App

1. Add a project reference to the unified project:
   ```xml
   <ProjectReference Include="path/to/output/amplitude/unified/Amplitude.Maui.csproj" />
   ```

2. Register the service in `MauiProgram.cs`:
   ```csharp
   var builder = MauiApp.CreateBuilder();
   builder.UseAmplitude();
   ```

3. Inject and use `IAmplitudeService`:
   ```csharp
   public class MyViewModel
   {
       private readonly IAmplitudeService _amplitude;

       public MyViewModel(IAmplitudeService amplitude)
       {
           _amplitude = amplitude;
           _amplitude.Initialize("YOUR_API_KEY");
       }

       public void TrackButtonPress()
       {
           _amplitude.LogEvent("button_pressed", new Dictionary<string, object>
           {
               ["button_name"] = "signup"
           });
       }
   }
   ```

## Prerequisites

- .NET 10 SDK
- Xcode or Xcode-beta (for iOS bindings)
- MAUI workloads: `dotnet workload install maui-ios maui-android`
- `gh` CLI (for fetching iOS artifacts from GitHub releases)
