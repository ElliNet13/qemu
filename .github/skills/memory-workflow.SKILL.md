---
name: vm-memory-workflow
description: "Workspace skill for finding QEMU VM memory subsystem code, paths, and key functions."
argument-hint: "What memory-related VM code are you working on?"
disable-model-invocation: true
---

# VM Memory Workflow Skill

This skill is a workspace-local reference for QEMU memory-related work. It is meant to help you quickly find memory subsystem code, understand the main API layers, and avoid hunting through the tree every time.

## When to use

- implementing or changing guest RAM allocation
- adding, modifying, or debugging `MemoryRegion` setup
- tracking memory dispatch/read-write paths
- fixing memory hotplug, alias, or address-space topology issues
- finding where RAM backing, dirty tracking, or memory listeners live

## Primary QEMU memory files

- `include/exec/memory.h` — core `MemoryRegion`, `AddressSpace`, and `MemoryRegionOps` definitions
- `qemu/system/memory.c` — main `MemoryRegion` initialization, dispatch, alias, IOMMU, and transaction logic
- `qemu/system/physmem.c` — root system/external physical memory setup and memory map construction
- `qemu/system/memory_mapping.c` — mapping memory regions to host addresses and pointers
- `include/exec/ram_addr.h` — `RAMBlock` helpers, host pointer helpers, dirty-bit helpers, and RAM allocation APIs
- `include/exec/ramblock.h` — RAM block structure and metadata
- `qemu/migration/ram.c` — RAM block lifecycle, dirty tracking, page save/load, and migration integration

## Core memory APIs to jump to

### MemoryRegion creation and setup

- `memory_region_init()` — generic region initialization
- `memory_region_init_io()` — device MMIO regions
- `memory_region_init_ram_nomigrate()` / `memory_region_init_ram_flags_nomigrate()` — normal RAM-backed regions
- `memory_region_init_resizeable_ram()` — resizable RAM
- `memory_region_init_ram_from_file()` / `memory_region_init_ram_from_fd()` — RAM backed by files or fd
- `memory_region_init_ram_ptr()` / `memory_region_init_ram_device_ptr()` — RAM using preallocated host memory
- `memory_region_init_alias()` — alias another region into the same address space
- `memory_region_init_iommu()` — IOMMU-capable region initialization
- `memory_region_init_rom_nomigrate()` / `memory_region_init_rom_device_nomigrate()` — read-only region variants

### MemoryRegion read/write dispatch

- `memory_region_dispatch_read()` — read path for memory regions
- `memory_region_dispatch_write()` — write path for memory regions
- `memory_region_dispatch_read1()` — lower-level dispatch helper
- `memory_region_read_accessor()` / `memory_region_write_accessor()` — accessor wrappers with endianness handling
- `memory_region_access_valid()` — validity checks on read/write access

### Address space and topology

- `address_space_get_flatview()` — get the flattened view for an address space
- `generate_memory_topology()` — build the global memory topology
- `address_space_set_flatview()` — install a new flatview onto an address space
- `address_space_update_topology()` — update region maps after memory transactions
- `memory_region_transaction_begin()` / `memory_region_transaction_commit()` — batch memory-region changes safely

### RAM and dirty tracking

- `memory_region_get_ram_ptr()` — raw host pointer for a RAM-backed region
- `memory_region_get_dirty_log_mask()` — dirty tracking mask for memory regions
- `memory_region_is_ram_device()` / `memory_region_has_guest_memfd()` — RAM and guest-memfd predicates
- `qemu_ram_alloc()` / `qemu_ram_alloc_resizeable()` / `qemu_ram_alloc_from_ptr()` — RAM allocation helpers in `include/exec/ram_addr.h`

## Secondary memory-related locations

- `qemu/accel/kvm/kvm-all.c` — KVM-specific memory pointer handling and host RAM mapping
- `qemu/accel/tcg/cputlb.c` — TCG memory dispatch and translation path
- `qemu/backends/hostmem*.c` — host-backed memory backend implementations
- `qemu/cpu-target.c` — `cpu_memory_rw_debug()` and CPU-side physical memory access support
- `qemu/system/ioport.c` — IO port memory region wiring and region subregion map for I/O
- `qemu/target/i386/kvm/kvm.c` — examples of aliasing memory for SMRAM and other special regions

## Fast search recipes

Use these grep patterns from the workspace root to find the likely code quickly:

- `grep -R "memory_region_init_ram" .` 
- `grep -R "memory_region_dispatch_.*(" .`
- `grep -R "memory_region_init_alias" .`
- `grep -R "memory_region_get_ram_ptr" .`
- `grep -R "address_space_.*flatview" .`
- `grep -R "qemu_ram_alloc" .`
- `grep -R "cpu_physical_memory_read" .`

## Skill usage example

Ask the agent:
- "Show me the QEMU code path for `MemoryRegion` RAM initialization."
- "Where does QEMU build the flattened guest memory topology?"
- "Find the files that handle RAM-backed vs MMIO `MemoryRegion` setup."

## Why this helps

This skill removes the need to remember the exact QEMU memory file locations and call chains. It focuses on the VM memory corpus, so you can jump directly to the memory subsystem rather than scanning unrelated device or CPU code.
