# quickjs-zig

Compile quickjs using the Zig build system.

## Usage

### Add to your dependencies

```
zig fetch --save git+https://github.com/sea-grass/quickjs-zig#main
```

### Reference the artifacts in your build.zig

```
// Rest of build elided for simplicity
pub fn build(b: *std.Build) void {
  // Reference the dependency from your build.zig.zon
  const quickjs_dep = b.dependency("quickjs");
  
  // Use the helper defined further below to build your c module
  const c_mod = CModule.build(b, .{
    .quickjs = quickjs_dep,
    .target = target,
    .optimize = optimize,
  });

  // Add an import to your c module
  exe_mod.addImport("c", c_mod);
}

/// A helper to build a c module in the build system without using
/// `cInclude`.
const CModule = struct {
    const header_bytes = (
        \\#include <quickjs.h>
        \\#include <quickjs-libc.h>
    );

    pub const Options = struct {
        quickjs: *std.Build.Dependency,
        target: std.Build.ResolvedTarget,
        optimize: std.builtin.OptimizeMode,
    };

    pub fn build(b: *std.Build, opts: Options) *std.Build.Module {
        const c = translateC(b, opts);
        const mod = createModule(c, opts);
        return mod;
    }

    fn translateC(b: *std.Build, opts: Options) *std.Build.Step.TranslateC {
        const header_file = b.addWriteFiles().add("c.h", header_bytes);

        const c = b.addTranslateC(.{
            .root_source_file = header_file,
            .target = opts.target,
            .optimize = opts.optimize,
            .link_libc = true,
        });
        
        // Add quickjs to the include path
        c.addIncludePath(opts.quickjs.artifact("quickjs").getEmittedIncludeTree());

        return c;
    }

    fn createModule(c: *std.Build.Step.TranslateC, opts: Options) *std.Build.Module {
        const mod = c.createModule();
        
        // Link the quickjs library
        mod.linkLibrary(opts.quickjs.artifact("quickjs"));
        
        return mod;
    }
};
```

### Use quickjs

In your zig file, import your c module and use the QuickJS library.

