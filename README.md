# DearImguiSharp

![Nuget](https://img.shields.io/nuget/v/DearImguiSharp)

DearImguiSharp is a minimalistic *personal use* .NET wrapper for the immediate mode GUI library, Dear ImGui (https://github.com/ocornut/imgui) built ontop of [cimgui](https://github.com/Extrawurst/cimgui). The goal of Dear ImGui is to allow a developer to build graphical interfaces using a simple immediate-mode style. 

By itself, Dear ImGui does not care what technology you use for rendering; it simply outputs textured triangles. Many example renderers exist such as ones for popular graphics APIs: OGL2, OGL3, D3D9 and D3D11 etc.

At the moment, the library only ships binaries for Windows. This is simply because I lack the motivation to compile, setup CI/CD and test on other Operating Systems. This specific version of the wrapper also ships with the example implementations for Win32, D3D9, 10 and 11 built-in; this is for my personal use.

I built this because I believe that while it generates more optimal, convenient bindings, [ImGui.NET](https://github.com/mellinoe/ImGui.NET) is too fragile. The custom written binding generator is difficult to maintain and very often breaks between source library updates.

## Goals
- Future-proof.
	- Uses tried and tested Mono binding generator [CppSharp](https://github.com/mono/CppSharp).
	- Uses minimal code fixes/changes, in hope API changes wouldn't break things.

- Vanilla
	- No redefinitions of method names etc. 
	- Makes it easy to use original C++ documentation for learning the library.
	
This is a minimal effort wrapper. Just the essentials to get going.

## Usage

Aside from capitalization being modified to match general C# code using CppSharp, the library API should exactly match the original source library.

- All structures reside in the `DearImguiSharp` namespace.
- All functions reside in the `DearImguiSharp.ImGui` class.
- Some custom defined functions reside in `DearImguiSharp.ImGui.Custom`. These are hand-generated for user convenience; but might not be up to date 
- Access to raw P/Invokes is available via `ImGui.__Internal`.

```csharp
using DearImguiSharp;

// Elsewhere in code.
_context = ImGui.CreateContext(null);
ImGui.StyleColorsDark(null);
```

Please note that the bindings don't have default values for some parameters; where a default value exists in the original API, you can pass null, 0 or the default value for the given type.

## Known Issues
- Setters for strings (e.g. `IO.IniFilename`) are not correctly generated and will point to non-static memory.
    - Use the exposed internal interfaces for setting those, e.g. `((ImGuiIO.__Internal*)io.__Instance)->IniFilename = Marshal.StringToHGlobalAnsi("tweakbox.imgui.ini");`

- Const parameters for value types are incorrectly generated by CppSharp.
  - For example, `PushStyleColorVec4` should have `int idx, ImVec2.__Internal val` as parameters and not `int idx, IntPtr val`;
  - The example above and many others I have tried to fix, but there's no guarantee.

## Building from Source
- Clone this repository and its submodules.
- Build [cimgui](https://github.com/cimgui/cimgui) using the instructions provided in its readme.
- Replace headers (`cimgui.h`, `cimgui.cpp`, `cimgui_impl.h`) in `CodeGenerator` project with the ones created using cimgui.
- Build and run `CodeGenerator`. Copy autogenerated `cimgui.dll.cs` to `DearImguiSharp` and compile.


I use my own fork of cimgui preconfigured with Win32, D3D9, 10, 11 support; that said, there is nothing stopping you using your own fork or the official version.

## Testing

I don't have any test project currently available in this repository. I originally built this library to create overlays for existing applications by hooking the DirectX API.

In that vein, you may check out [Reloaded.Imgui.Hook](https://github.com/Sewer56/Reloaded.Imgui.Hook) which utilizes this library. This is what I use to test the library.

## Performance Recommendation(s)
From a performance enthusiast; CppSharp doesn't generate the most efficient code (it's full of reference types!) so let's not make the GC trigger happy.

- Avoid the heap like the plague.
    - Especially with types like `ImVec2` and `ImVec4`. 
	- This means do not use  `var vector = new ImVec2();`.
	- Use the stack! 
- Assume everything is disposable (use `using` statement).
  - Or you'll probably be leaking memory. 
	
How? This library goes out of its way to expose the low level P/Invokes and structures autogenerated by CppSharp (normally private).

Don't do this:

```csharp
using var vector = new ImVec2(); // Heap allocation + Unmanaged Heap Allocation + Dictionary Entry
ImGui.CalcTextSize(vector, text, null, false, -1.0f);
```

This is slightly better (but limits you to stack):

```csharp
var vecInternal = new ImVec2.__Internal();
var vector = new ImVec2(&vecInternal); // Heap allocation
ImGui.CalcTextSize(vector, text, null, false, -1.0f);
```

This is the best:
```csharp
var vecInternal = new ImVec2.__Internal();
ImGui.__Internal.CalcTextSize((IntPtr) (&vecInternal), text, null, false, -1.0f);
```
	
This library is a *minimal effort* library. We try to manually fix the least amount of things so hopefully in the future we can just wrap newer versions of dear imgui without worrying about API changes.

## To Do

- Modify CppSharp generator to not redefine typedef'd void\*\* to IntPtr as this produces incorrect code (no implicit conversion from void\*\* to IntPtr).
- Optimize wrapper by using more efficient bindings. (e.g. Map `ImVec2` to `System.Numerics.Vector2`).

- Support other operating systems:
  - Modify `DearImguiSharp.targets` to add support for other platforms.
  - Implement CI/CD or Github Actions to build `cimgui` for other platforms. 
  - Some kind of reasonable test; maybe use [Veldrid](https://github.com/mellinoe/veldrid).

I plan to address these in the future but it likely wouldn't be any time soon.

This is my first time using CppSharp, so there's still plenty of stuff to still learn.
I'll happily accept pull requests for any of these features or general quality of life improvements.

## Related Projects

- [Dear ImGui](https://github.com/ocornut/imgui): Original Library 
- [cimgui](https://github.com/cimgui/cimgui): C Bindings for Dear Imgui by @sonoro1234
- [ImGui.NET](https://github.com/mellinoe/ImGui.NET): Alternative wrapper using custom code generator by @mellinoe

