---
layout: post
title: Zig - Linking Dear ImGui
date:   2025-04-30 10:00:00 +0100
banner: /assets/img/2025-04/c_programming_language_striked_by_lightning.png
categories: devlog zig
---

I have come to find that apparantly, linking Dear ImGui isn't as straight forward as I thought.
While working on my new rendering engine [Z8](https://github.com/MrScriptX/z8), I had to reimplement Dear ImGui.
I have been using it in C++ for a while now, and I thought it would be easy to just link it in Zig.
And well, it wasn't too hard really, but I had to figure out a few things along the way.
So here are the sweet little tips I got from this experience.

## Building Dear ImGui

First of all, you need to build the Dear ImGui bindings for C.

Instead of using the `cimgui` bindings, we are going to use the official Dear ImGui C bindings: [dear bindings](https://github.com/dearimgui/dear_bindings).

First, clone imgui repository and the dear bindings repository.:

```bash
git clone https://github.com/ocornut/imgui.git
git clone https://github.com/dearimgui/dear_bindings.git
```

Now we can generate the bindings by running the following command:

```bash
$ cd dear_bindings
$ python -m venv .venv
$ source .venv/bin/activate
$ pip install -r requirements.txt
$ ./BuildAllBindings.sh
$ deactivate
```

For Windows users, you can use the `BuildAllBindings.bat` script instead.

```bat
$ cd dear_bindings
$ python -m venv .venv
$ .venv\Scripts\activate.bat
$ pip install -r requirements.txt
$ BuildAllBindings.bat
$ deactivate
```

This will generate the bindings in the `generated` folder.

## Linking Dear ImGui

Now that we have the bindings, we can link them in our Zig project.
Copy the `generated` folder to your Zig project folder.

I personally structure my project like this:

- `project_root`
  - `libs`
    - `cimgui` (Dear ImGui module for Zig)
      - `src`
        - `root.zig` (the root file of the module)
        - `build.zig` (the build file of the module)
  - `common`
    - `imgui` (the Dear ImGui source files)
  - `src`
    - `main.zig` (the main file of the project)

As you can see, I don't directly import the Dear ImGui source files, but I create a module for it.
Then I import the module in my main file.

You can reproduce this structure or use another one, but just make sure the paths are correct,
and that you possess at least `root.zig` and `build.zig` files for Dear ImGui module.

In the new `build.zig`, we will add the following code:

```zig
const std = @import("std");
const Build = std.Build;

pub fn build(b: *Build, target: Build.ResolvedTarget, optimize: std.builtin.OptimizeMode) *Build.Module {
    // we create a new module for the imgui library
    const module = b.addModule("stb", .{
        .root_source_file = b.path("libs/cimgui/src/root.zig"), // this is the root file of the module
        .target = target,
        .optimize = optimize,
    });

    module.addIncludePath(.{ .cwd_relative = "common/imgui" });
    module.addCSourceFile(.{ .file = b.path("common/imgui/imgui.cpp"), .flags = &.{ "" } });
    module.addCSourceFile(.{ .file = b.path("common/imgui/imgui_widgets.cpp"), .flags = &.{ "" } });
    module.addCSourceFile(.{ .file = b.path("common/imgui/imgui_tables.cpp"), .flags = &.{ "" } });
    module.addCSourceFile(.{ .file = b.path("common/imgui/imgui_draw.cpp"), .flags = &.{ "" } });
    module.addCSourceFile(.{ .file = b.path("common/imgui/imgui_demo.cpp"), .flags = &.{ "" } });
    module.addCSourceFile(.{ .file = b.path("common/imgui/dcimgui.cpp"), .flags = &.{ "" } });
    module.addCSourceFile(.{ .file = b.path("common/imgui/dcimgui_internal.cpp"), .flags = &.{ "" } });

    return module; // return the module so we can use it in our main file
}
```

For the backends, you have to link their respective library too and add the include paths.
For example, if you are using SDL3 and Vulkan, you can add the following :

```zig
// Add SDL3 and Vulkan backends
module.addCSourceFile(.{ .file = b.path("common/imgui/imgui_impl_sdl3.cpp"), .flags = &.{ "" } });
module.addCSourceFile(.{ .file = b.path("common/imgui/imgui_impl_vulkan.cpp"), .flags = &.{ "" } });

// add the c bindings for SDL3 and Vulkan
module.addCSourceFile(.{ .file = b.path("common/imgui/dcimgui_impl_sdl3.cpp"), .flags = &.{ "" } });
module.addCSourceFile(.{ .file = b.path("common/imgui/dcimgui_impl_vulkan.cpp"), .flags = &.{ "" } });

// Add SDL3 include path
module.addIncludePath(.{ .cwd_relative = "common/SDL3/include" });

// Add Vulkan include path
const env_map = std.process.getEnvMap(b.allocator) catch @panic("Out of memory !");
if (env_map.get("VK_SDK_PATH")) |path| { // find VK_SDK_PATH in the environment variables
    module.addIncludePath(.{ .cwd_relative = std.fmt.allocPrint(b.allocator, "{s}/include", .{path}) catch @panic("OOM") });
}
else {
    @panic("VK_SDK_PATH not found ! Please install Vulkan SDK.");
}
```

Ok, now that we have a build function to create a module for imgui, we can go to the `root.zig` file.
In this file, we are going to export the functions we need from the imgui library.

```zig
pub usingnamespace @cImport({
    @cInclude("dcimgui.h");
    @cInclude("dcimgui_impl_sdl3.h");
    @cInclude("dcimgui_impl_vulkan.h");
    // ...add other includes here if needed
});
```

We can even export wrapper functions for the Dear ImGui functions we need.
```zig
pub fn SliderInt(label: []const u8, v: *i32, v_min: i32, v_max: i32) bool {
    return c.ImGui_SliderInt(@ptrCast(label), v, v_min, v_max);
}

// import the C functions again but under the c namespace to use them in Zig
// if you expose all the functions you need, you can even drop the previous import.
const c = @cImport({
    @cInclude("dcimgui.h");
    @cInclude("dcimgui_impl_sdl3.h");
    @cInclude("dcimgui_impl_vulkan.h");
});
```

Back to our main `build.zig` file, we can now add the imgui module to our build.
```zig
// you should already have this at the top of your file
const target = b.standardTargetOptions(.{});
const optimize = b.standardOptimizeOption(.{});

const exe = b.addExecutable(.{
    .name = "project",
    .root_source_file = b.path("src/main.zig"),
    .target = target,
    .optimize = optimize,
    .link_libc = true,
});

// add the imgui module to the executable
const cimgui = @import("libs/cimgui/build.zig").build(b, target, optimize);
exe.root_module.addImport("imgui", cimgui); // this is so we can import our module

// link std cpp library
exe.linkLibCpp();
```

## Using our brand new module

Now that we have our module, we can use it in our main file.

```zig
const std = @import("std");
const imgui = @import("imgui");

pub fn main() void {
    // use c functions from imgui
    const context = imgui.ImGui_CreateContext(null);
    if (context == null) {
        @panic("Failed to create ImGui context!");
    }
    defer imgui.ImGui_DestroyContext(context);

    // ... other code

    // or use the wrapper function we created
    var value: i32 = 0;
    if (imgui.SliderInt("My Slider", &value, 0, 100)) {
        std.debug.print("Slider value: {}\n", .{value});
    }
}
```

## Conclusion

That's really all there is to it.
If you have any questions, feel free to ask in the comments or anywhere I post ([YouTube](https://www.youtube.com/@R3DC0DE), Reddit, etc...).

Hopefully this guide did help you out a bit.

Be blessed and happy coding !
