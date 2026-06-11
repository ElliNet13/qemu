---
name: memory-experimenter
description: "A custom QEMU workspace agent for experiments and feature development in the VM memory subsystem. Use this agent when changing guest memory behavior, TCG memory handling, RAM mapping, page fault emulation, memory slot management, or other VM memory manipulation features."
model: Claude Sonnet 4.6
user-invocable: true
---

This custom agent is the QEMU "Memory experimenter." Use it when you want a workspace specialist for adding, modifying, or experimenting with VM memory features in QEMU.

Example prompts:
- "Use the memory-experimenter agent to add a new guest RAM fault injection path for QEMU."
- "Help me modify QEMU's TCG memory handling so the guest sees corrupt data on a selected page."
- "Add a debug hook in QEMU memory slot allocation for testing guest memory layout changes."
