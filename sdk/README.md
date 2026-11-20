# Parabellum SDK Specification

Compact reference for building Parabellum **Task**, **Driver**, and **Extension** plugins, as well as **App / Host** integrations, using the single unified `parabellum_sdk` module (no direct `engine` import required).

---

## 1. Build & Module Setup

In `build.zig.zon`, declare `parabellum_sdk` as a dependency. In `build.zig`, import the SDK build helpers:

```zig
const sdk = @import("parabellum_sdk");
const sdk_dep = b.dependency("parabellum_sdk", .{ .target = target, .optimize = optimize });
```

### Plugin Build Helpers

| Helper | Output | Module Imported |
|---|---|---|
| `sdk.addTaskPlugin(b, sdk_dep, opts)` | `lib<name>.so` | `parabellum_sdk` |
| `sdk.addDriverPlugin(b, sdk_dep, opts)` | `libdriver_<name>.so` | `parabellum_sdk` |
| `sdk.addExtensionPlugin(b, sdk_dep, opts)` | `libext_<name>.so` | `parabellum_sdk` |
| `sdk.linkPrecompiledTestEngine(compile, opts)` | Links `libparabellum_host` | — |

`PluginOptions`: `{ .name, .root_source_file, .target, .optimize, .link_libc = true }`.

---

## 2. Top-Level Namespaces (`@import("parabellum_sdk")`)

| Namespace | Purpose |
|---|---|
| `sdk.abi` | Unified C-ABI contracts (`abi.common`, `abi.task`, `abi.driver`, `abi.extension`). |
| `sdk.task` | Task Guest SDK: `defineTaskService`, `defineManifest`, `DiContainer`, `TaskContext`, `Logger`, `BoundedStreamReader`, `WindowReader`, `StreamEmitter`, `StreamOutput`, `Channels`, `Rows`. |
| `sdk.driver` | Driver Guest SDK: `Driver` (`Driver.define`, `Driver.Meta`). |
| `sdk.extension` | Extension Guest SDK: `Extension` (`Extension.define`, `Extension.Meta`). |
| `sdk.client` | Protocol-agnostic datasource client: `Client`, `ClientError`. |
| `sdk.job` | Job models: `JobStatus`, `JobRecord`, `JobItem`. |
| `sdk.event` | Daemon event models: `EventKind`, `EventPayload`, `DaemonEvent`, `CustomEvent`, `KeyValue`. |
| `sdk.host` | Host C-ABI & integration runner: `TestEngine`, `RunOptions`, `RunSummary`, `c_api`. |
| `sdk.testing` | In-process testing: `TaskHarness`, `runDriverConformanceSuite`. |

---

## 3. Task Plugins (`sdk.task`)

Tasks are defined as Zig service structs wired through a **task-scoped** `DiContainer` (bound to `TaskContext` so it can resolve host-provided datasources, loggers, and streams from the engine).

### Task-Scoped DI Container

```zig
const std = @import("std");
const sdk = @import("parabellum_sdk");

pub const AppContext = struct {
    logger: sdk.task.Logger,
    api: ?sdk.client.Client = null,

    pub fn init(self: *AppContext, alloc: std.mem.Allocator, ctx: sdk.task.TaskContext) !void {
        _ = alloc;
        self.* = .{
            .logger = sdk.task.Logger.init(ctx.raw),
            .api = sdk.client.Client.fromContext(ctx, "my_api") catch null,
        };
    }

    pub fn deinit(_: *AppContext) void {}
};

pub const AppContainer = sdk.task.DiContainer(AppContext);
```

### Service Registration & Manifest Export

```zig
const sdk = @import("parabellum_sdk");

pub const tasks = [_]sdk.abi.task.TaskDescriptor_ABI{
    sdk.task.defineTaskService(MyService, .{
        .name = "my_task",
        .container = AppContainer,
    }),
};

pub const manifest: sdk.abi.task.ModuleManifest_ABI = sdk.task.defineManifest(
    "my_plugin",
    .{ .major = 0, .minor = 1, .patch = 0 },
    &tasks,
);

comptime {
    @export(&manifest, .{ .name = sdk.abi.task.MANIFEST_SYMBOL_NAME, .linkage = .strong });
}
```

Use `sdk.task.defineManifestWithRequires` when the plugin declares dependencies (`sdk.abi.task.PluginRequirement_ABI`).

### Streaming & Context Primitives

- `sdk.task.TaskContext` / `sdk.task.Logger`: Safe wrappers over `sdk.abi.task.TaskContext_ABI`.
- `sdk.task.BoundedStreamReader` / `sdk.task.WindowReader` / `sdk.task.ClampedSubRegion`: Memory-safe streaming readers.
- `sdk.task.StreamEmitter` / `sdk.task.StreamOutput` / `sdk.task.Channels` / `sdk.task.Rows`: Typed streaming and multi-channel outputs.
- `sdk.client.Client`: Comptime-typed client over datasource repositories (`Client.fromContext(ctx, "datasource_id")`).

---

## 4. Driver Plugins (`sdk.driver`)

Drivers implement datasource/storage backends (`libdriver_<name>.so`):

```zig
const sdk = @import("parabellum_sdk");

pub const manifest = sdk.driver.Driver.define(MyDriver, .{
    .name = "my_driver",
    // DriverMeta fields
});
```

Low-level C-ABI types live in `sdk.abi.driver` (`DriverManifest_ABI`, `StorageRepositoryVTable_ABI`, `ReadSpec_ABI`, `WriteSpec_ABI`, `RowIterator_ABI`).

---

## 5. Extension Plugins (`sdk.extension`)

Kernel extensions hook into engine lifecycle, telemetry, and inbound transports (`libext_<name>.so`):

```zig
const sdk = @import("parabellum_sdk");

pub const manifest = sdk.extension.Extension.define(.{
    .name = "my_extension",
    // ExtensionMeta fields
});
```

Low-level C-ABI types live in `sdk.abi.extension` (`ExtensionManifest_ABI`, `KernelApiVTable_ABI`, `TelemetrySubscriberVTable_ABI`, `InboundTransportVTable_ABI`).

---

## 6. Testing (`sdk.testing` & `sdk.host`)

- `sdk.testing.TaskHarness` — Unit-test tasks in-process with `std.testing.allocator` and zero engine binaries.
- `sdk.testing.runDriverConformanceSuite` — Standard conformance suite for driver plugins.
- `sdk.host.TestEngine` (`RunOptions`, `RunSummary`) — Integration-test workflows against a precompiled `libparabellum_host` (link via `sdk.linkPrecompiledTestEngine`).
