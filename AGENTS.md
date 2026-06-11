# QEMU Agent Guidance

This is a QEMU fork focused on experiments and feature development. Below is guidance for AI coding agents to be immediately productive.

## Project Overview

**What is QEMU?**  
QEMU is a generic, open-source machine and userspace emulator/virtualizer. This fork (read: *"I mess with because why not"*) is used for VM memory subsystem experimentation. Key capabilities:
- **System emulation**: Full machine emulation (QEMU executable runs guest OS)
- **User-mode emulation**: Execute foreign architecture binaries on the host
- **Hypervisor integration**: KVM, HVF, Xen support for near-native performance
- **Multi-target**: Supports 17+ CPU architectures (ARM, x86, MIPS, RISC-V, etc.)

The codebase is **~10 million LOC** across C, Python, and Rust, organized into modular subsystems.

## Build System: Meson + Makefile

**Quick Start**  
```bash
mkdir build && cd build
../configure
make -j$(nproc)
```

**Build System Architecture**
- **configure script** (`./configure`): Detects host, selects targets, finds compilers, creates Meson config
- **Meson** (`meson.build` files): Primary build orchestrator; describes modules, sourcesets, dependencies
- **Makefile** and **build/Makefile**: Thin wrapper around Meson; manages build.ninja generation and prerequisites
- **Kconfig** (`*/Kconfig`): Device/board configuration language for per-target feature selection
- **build/**: Build artifacts (binaries, libraries, headers, etc.)

**Key Build Concepts**
- **Sourcesets**: Modular compilation units grouped by subsystem (e.g., `block_ss`, `chardev_ss`)
- **Multi-target builds**: Compile multiple emulators (qemu-system-x86_64, qemu-arm, qemu-img, etc.) from shared code
- **Static libraries**: Subsystems compiled to static libs (libchardev, libblock, libqemuutil.a) then linked
- **CONFIG_* variables**: Gate features per target (enable via configs/devices/*.mak or Kconfig)

**Build Targets**
- System emulators: `qemu-system-$ARCH` (e.g., qemu-system-x86_64)
- Userspace emulators: `qemu-$ARCH` (e.g., qemu-arm)
- Tools: qemu-img, qemu-nbd, qemu-ga (guest agent)
- Tests: unit tests, qtest (device tests), TCG tests, qemu-iotests

See [docs/devel/build-system.rst](docs/devel/build-system.rst) for full details.

## Code Organization

### Top-Level Structure
```
qemu/
├── accel/           → Accelerators: tcg/ kvm/ hvf/ xen/
├── target/          → CPU emulation per architecture (arm/ x86/ mips/ riscv/ etc.)
├── hw/              → Device emulation: hw/pci/ hw/block/ hw/net/ hw/arm/ etc.
├── tcg/             → Tiny Code Generator (dynamic translator)
├── system/          → System-mode QEMU entry point
├── linux-user/      → Linux usermode entry point
├── bsd-user/        → BSD usermode entry point
├── include/         → Public/internal headers
├── qom/             → QEMU Object Model (OOP framework)
├── util/            → Utilities: lists, queues, rcu, threads, etc.
├── io/              → I/O channels, DNS, sockets, TLS
├── chardev/         → Character devices (serial, stdio, TCP, etc.)
├── block/           → Block layer (disk I/O, image formats, snapshots)
├── migration/       → VM state migration & live migration
├── net/             → Network emulation
├── docs/devel/      → Developer documentation (Sphinx-generated)
├── scripts/         → Build scripts, checkpatch.pl, git hooks
└── tests/           → Test suites: qtest/ avocado/ tcg/ qemu-iotests/
```

### Modular Architecture Principles

1. **Device Emulation via QOM (QEMU Object Model)**  
   - Object-oriented C framework for device instantiation and configuration
   - Devices inherit from Type hierarchy (e.g., Object → Device → PCIDevice)
   - Properties, realized callbacks, interrupts are instance state
   - Example: [include/hw/qdev-core.h](include/hw/qdev-core.h)

2. **Conditional Compilation via Sourcesets**  
   - Files added to sourcesets only when their CONFIG_* is true
   - Enables target-specific and feature-specific building
   - Example: `specific_ss.add(when: 'CONFIG_KVM', if_true: kvm_ss)`

3. **Subsystem Boundaries**  
   - **block/**: Disk I/O, image formats (QCOW2, RAW, etc.), snapshots
   - **net/**: Network devices, tap, vhost-user
   - **chardev/**: Serial ports, console I/O, socket backends
   - **hw/**: Boards and device models per arch
   - **target/**: CPU instruction decode, register state, emulation hooks

4. **Per-Target Compilation**  
   - Each target (x86_64-softmmu, arm-linux-user, etc.) has own sourceset
   - Files in `hw/` and `target/` compiled per TARGET_ARCH
   - Shared code in `common_ss`, `system_ss`, `user_ss`

## Memory Subsystem (Primary Focus for This Fork)

**Entry Points for Memory Experimentation**
- **Core Memory API**: [include/exec/memory.h](include/exec/memory.h), [system/memory.c](system/memory.c)
- **CPU TLB & Cache**: [accel/tcg/cputlb.c](accel/tcg/cputlb.c), [accel/tcg/watchpoint.c](accel/tcg/watchpoint.c)
- **Page Handling**: [accel/tcg/translate-all.c](accel/tcg/translate-all.c) (TLB flush, TB invalidation)
- **Address Spaces**: `MemoryRegion`, `AddressSpace` abstractions
- **VM State**: Guest memory layout and DMA configuration

**Common Modifications**
- Trace new memory operations: Add to trace-events, rebuild (see [docs/devel/tracing.rst](docs/devel/tracing.rst))
- Change page size handling: Modify TCG page fault detection in [accel/tcg/cputlb.c](accel/tcg/cputlb.c)
- Experiment with TLB: Adjust [accel/tcg/translate-all.c](accel/tcg/translate-all.c), recompile target
- Add memory callbacks: Use MemoryRegionOps in device code

## Developer Conventions

### Coding Style
Follow [docs/devel/style.rst](docs/devel/style.rst):
- **Indentation**: 4 spaces (never tabs)
- **Line length**: 80 characters (aim for readability, not hard limit)
- **Variable names**: `lower_case_with_underscores`
- **Type names**: `CamelCase`
- **Function prefixes**: `qemu_` for wrapped stdlib; subsystem-specific for public functions
- **Naming shortcuts**: `cs` = CPUState, `env` = CPUArchState, `dev` = DeviceState

**Enforce with**: `./scripts/checkpatch.pl <file>` or `<patch>`

### Module/File Patterns
```c
/* File: target/arm/translate.c — CPU instruction decoder */
typedef struct DisasContext { ... } DisasContext;
static void arm_tr_init_disas_context(...) { ... }
static bool arm_tr_breakpoint_check(...) { ... }

/* Device: hw/arm/versatilepb.c — Board model */
static void versatile_pb_init(...) { ... }
type_init(versatile_pb_machine_init_register_types)

/* Module: block/qcow2.c — Disk image format */
static int qcow2_open(...) { ... }
BlockDriver bdrv_qcow2 = { .format_name = "qcow2", ... };
```

### Common Macros
- `OBJECT_CHECK(type, obj, typename)`: Type-safe cast
- `OBJECT_GET_CLASS(class_type, obj, typename)`: Get class from object
- `object_property_add*(obj, name, ...)`: Property registration
- `qemu_strtol()`, `qemu_mutex_lock()`: Wrapped stdlib functions
- `trace_*()`: Automatic tracing (if event declared in trace-events)

## Testing Framework

**Test Suites**

| Suite | Location | Purpose | Run with |
|-------|----------|---------|----------|
| **Unit Tests** | tests/unit/ | Isolated function tests (QOM, JSON, RCU, etc.) | `make check-unit` |
| **QTest** | tests/qtest/ | Device emulation tests; uses guest-side binary | `make check-qtest` |
| **TCG Tests** | tests/tcg/ | CPU emulation tests per architecture | `make check-tcg` |
| **Avocado** | tests/avocado/ | Integration tests; boots full guest OS | `make check-avocado` |
| **QEMU I/O Tests** | tests/qemu-iotests/ | Block layer, disk image format tests | `cd tests/qemu-iotests && ./check` |
| **Functional** | tests/functional/ | Python-based higher-level tests | `make check-functional` |

**Build & Run**
```bash
make check-help              # List all test targets
make check                   # Run quick unit + qtest + block tests (default)
make check-slow              # Run slow tests (adds TCG tests)
make check-tcg               # TCG translator tests
make check-qtest TARGET=arm  # Device tests for ARM architecture
```

**Writing Tests**
- Unit tests: C code in tests/unit/, compiled to test binary
- QTest: Guest-side binary + host-side test harness (communicate via guest I/O)
- Example: [tests/qtest/allwinner-h3-test.c](tests/qtest/allwinner-h3-test.c)

See [docs/devel/testing.rst](docs/devel/testing.rst) for comprehensive test guide.

## TCG: The Dynamic Translator

**What is TCG?**  
TCG (Tiny Code Generator) translates guest CPU instructions to host machine code at runtime. It's the core of CPU emulation.

**Key Components**
- **Translation Blocks (TB)**: Sequences of guest instructions compiled to host code
- **TCG Ops**: Intermediate representation (IR) between guest ISA and host ISA
- **Direct block chaining**: Optimized jumps between compiled TBs
- **Multi-threaded support**: Each vCPU runs on own thread (for modern systems)

**For Memory Experiments**
- Modify page table walks: [accel/tcg/cputlb.c](accel/tcg/cputlb.c) handles guest TLB + page faults
- Add memory protection: Watchpoint logic in [accel/tcg/watchpoint.c](accel/tcg/watchpoint.c)
- Trace memory access: Use tcg_gen_*() IR ops; see [tcg/tcg-op.c](tcg/tcg-op.c)

See [docs/devel/tcg.rst](docs/devel/tcg.rst) and [docs/devel/multi-thread-tcg.rst](docs/devel/multi-thread-tcg.rst).

## Common Development Tasks

### Task: Add a trace event
1. Declare in relevant `trace-events` file (e.g., [hw/block/trace-events](hw/block/trace-events))
2. Call via `trace_event_name(args)` in code
3. Rebuild: `cd build && make`
4. View traces: `qemu-system-x86_64 -trace enable=event_name -trace file=/tmp/trace.bin`

### Task: Modify device behavior
1. Find device file (e.g., [hw/arm/versatilepb.c](hw/arm/versatilepb.c))
2. Locate `TypeInfo` registration and device properties
3. Update property callbacks or initialization
4. Rebuild target: `make qemu-system-arm`
5. Test: `qemu-system-arm -M versatilepb ...`

### Task: Add a new config option
1. Add to relevant `Kconfig` file (e.g., [hw/arm/Kconfig](hw/arm/Kconfig))
2. Update `configs/devices/*.mak` to enable for target(s)
3. Guard code: `#ifdef CONFIG_MYFEATURE` or `if get_option('myfeature')` in meson.build
4. Rebuild: `make clean && make`

### Task: Debug a guest instruction execution
1. Enable tracing: `./configure --enable-trace-backends=simple`
2. Build: `make qemu-system-arch`
3. Run with debug: `qemu-system-arch -d in_asm,exec,cpu -trace file=/tmp/trace.log ...`
4. View traces in `/tmp/trace.log`

### Task: Write a test for memory behavior
1. Create C file in `tests/qtest/` or `tests/tcg/`
2. Use QTest guest protocol or direct CPU emulation test
3. Add to appropriate `meson.build`
4. Run: `make check-qtest` or `make check-tcg`

## Entry Points & Key Files

**System Emulation**
- Entry: [system/main.c](system/main.c) → `qemu_init()` → `qemu_main_loop()`
- Main loop: [system/main-loop.c](system/main-loop.c)
- Machine registration: Boards defined in `hw/*/` and registered via `type_init()`

**Linux User Emulation**
- Entry: [linux-user/main.c](linux-user/main.c)
- Syscall handling: [linux-user/syscall.c](linux-user/syscall.c)
- Architecture-specific: [linux-user/arch/*/cpu_loop.c](linux-user/arch/)

**Object Model**
- Core: [qom/object.c](qom/object.c), [include/qom/object.h](include/qom/object.h)
- Device layer: [hw/core/qdev.c](hw/core/qdev.c), [include/hw/qdev-core.h](include/hw/qdev-core.h)

**Memory & Execution**
- Memory API: [system/memory.c](system/memory.c), [include/exec/memory.h](include/exec/memory.h)
- CPU execution: [accel/tcg/cpu-exec.c](accel/tcg/cpu-exec.c)
- TLB & caching: [accel/tcg/cputlb.c](accel/tcg/cputlb.c)

## Version & Release Info

Check [VERSION](VERSION) file for current version.  
Patch submission: Use email-based workflow to `qemu-devel@nongnu.org`; see [docs/devel/submitting-a-patch.rst](docs/devel/submitting-a-patch.rst).

## Critical Pitfalls

⚠️ **Common Mistakes**

1. **Forgetting to rebuild after meson.build changes**  
   → Solution: `cd build && rm -rf meson-private build.ninja && make`

2. **Mixing in-tree and out-of-tree builds**  
   → Solution: Always use `mkdir build && cd build && ../configure`

3. **Not testing multi-target builds**  
   → Solution: Run `./configure --target-list=x86_64-softmmu,arm-softmmu,arm-linux-user && make`

4. **Modifying code without understanding sourceset scope**  
   → Solution: Check which sourcesets include your file; see [docs/devel/build-system.rst](docs/devel/build-system.rst)

5. **Forgetting CONFIG_* guards when target-specific**  
   → Solution: Gate code: `if have_system: ... endif` or `when: 'CONFIG_MYFEATURE'`

6. **Assuming single-threaded execution**  
   → Solution: Use atomic ops for shared state; review [docs/devel/multi-thread-tcg.rst](docs/devel/multi-thread-tcg.rst)

## References

- **Full Developer Guide**: [docs/devel/index.rst](docs/devel/index.rst)
- **Build System Deep Dive**: [docs/devel/build-system.rst](docs/devel/build-system.rst)
- **QOM & Device Model**: [docs/devel/qom.rst](docs/devel/qom.rst)
- **TCG Internals**: [docs/devel/tcg.rst](docs/devel/tcg.rst)
- **Coding Style**: [docs/devel/style.rst](docs/devel/style.rst)
- **Testing Guide**: [docs/devel/testing.rst](docs/devel/testing.rst)
- **Trace Events**: [docs/devel/tracing.rst](docs/devel/tracing.rst)

---

**Last updated**: 2026-06-11  
**Scope**: Memory subsystem experimentation and feature development
