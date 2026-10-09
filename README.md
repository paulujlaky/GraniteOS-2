# GraniteOS 2

A from-scratch **microkernel** rewrite of GraniteOS for ARM64 (and, later,
x86_64).

[![GraniteOS Desktop Demo](_docs/desktop_demo.png)](_docs/desktop_demo.png)

## Requirements

- **Zig 0.15** (tested with 0.15.2)
- **QEMU** with `qemu-system-aarch64`.

## Build And Run 

```sh
zig build # kernel image + user module bundle + persistent disk seed (zig-out/bin)
zig build qemu # boot the full system under QEMU virt (interactive; quit with Ctrl-A x)
zig build qemu-gui # SDL desktop with virtio GPU, input, and sound
zig build qemu-nodisk # boot without a virtio-blk disk (the filesystem reports unavailable)
zig build qemu-bare # boot the kernel alone; halts after initialization
zig build qemu-debug # boot halted with a gdb stub on :1234
zig build test # host unit tests for the kernel core and the user runtime
```

QEMU run steps accept `-Dsmp=<cores>`, `-Dmemory=<MiB>`, and `-Ddisk=<MiB>`
to size the virtual machine and persistent disk image (defaults: 4 cores,
512 MiB RAM, 64 MiB disk). 

## Building Your Own Apps

The desktop pins **Software**, and the launcher exposes it beside its search
field. It reads a small HTTPS repository index

See [`repo/README.md`](repo/README.md) for the read/publish protocol and
deployable software server. This is important for hosting and publishing your own 
third-party apps. To build some on top of GraniteOS 2, see [`sdk/README.md`](sdk/README.md) 
for the source-linked Zig SDK, example application, stripped ELF builder, and publisher.

## Layout

Modules exist as directories and are intentionally thin, and split between Zig code
and Assembly where needed.

```
build.zig                 kernel + user ELFs + bundle/flatten/seedisk tools + QEMU run steps
build/discover.zig        build-time user-module scan and app-catalog generation
tools/flatten.zig         host tool: ELF -> load-faithful flat image
tools/bundle.zig          host tool: user module bundle packer
tools/seedisk.zig         host tool: format and seed the persistent virtio disk
repo/                     repository protocol and reference cloud server
sdk/                      third-party Zig API, example, builder, and publisher
kernel/
  main.zig                post-arch entry; machine discovery; subsystem init; hand-off
  config.zig              compile-time tunables
  error.zig               shared Error set + ABI mapping
  types.zig               arch-free address types
  inspect.zig             kernel inspection surface for user-space status tools
  tests.zig               host unit-test aggregator
  arch/
    arch.zig              the architecture boundary
    host.zig              host-test stand-in for the boundary
    aarch64/
      asm/
        start.S           early boot: EL1, stack, BSS, vectors, MMU enable
        vectors.S         exception vector table
        switch.S          context switch + fresh-thread trampoline
        linker.ld         image layout
      boot.zig            bridge from start.S into Zig
      mmu.zig             seed map, MMU enable, page-table surface
      trap.zig            trap entry: IRQ -> scheduler tick; else diagnose + halt
      cpu.zig             core id, barriers, interrupt mask, halt
      context.zig         thread context: init + switch surface
      gic.zig             GICv3 distributor + per-core redistributors
      timer.zig           ARM generic timer: monotonic time + deadline
      psci.zig            PSCI CPU_ON for secondary-core bring-up
    board/virt.zig        board fallback constants (UART, GIC windows)
  boot/
    dtb.zig               device-tree parse: memory, cores, intctrl windows
    handoff.zig           kernel -> Flint argument packing
    smp.zig               secondary-core entry and scheduler attach
    bundle.zig            Flint-module lookup inside the initrd bundle
  memory/
    frames.zig            buddy physical-frame allocator
    slab.zig              per-type object caches
    region.zig            Region: a run of RAM frames
    address_space.zig     AddressSpace: page tables, map/unmap/activate
  object/
    object.zig            common object header (kind + refcount)
    process.zig           Process: AddressSpace + HandleTable + threads
    thread.zig            Thread: context, state, scheduling
    endpoint.zig          IPC endpoint object
    interrupt.zig         interrupt object + delivery
    notification.zig      notification object for async wakeups
  cap/
    handle.zig            Handle {index, generation}
    handle_table.zig      per-process handle table
  authority/              capability-grant authorities (memory, device, DMA, ...)
  ipc/                    kernel-side message transfer
  sync/                   kernel spinlocks and IPC helpers
  syscall/                syscall dispatch surface
  sched/
    runqueue.zig          intrusive per-core queues
    scheduler.zig         MLFQ + driver class, tick, demote/boost, yield
  debug/
    console.zig           panic-only PL011 UART
    panic.zig             panic diagnostic + halt
user/
  flint/main.zig          boot supervisor; spawns servers, drivers, and Marble/GUI
  marble/main.zig         interactive shell
  lib/
    root.zig              user runtime entry; re-exports submodules
    cap/                  handle indices and grant layouts
    ipc/                  message envelope and protocol constants
    syscall/              syscall wrappers
    runtime/              program entry and init-message handling
    boot/                 bundle reader, ELF loader, DTB parser, app catalog
    io/                   streams, logging, and formatting helpers
    fs/                   filesystem client helpers
    net/                   frame ring, address/checksum helpers, and the TCP socket client
    draw/                 raster, vector, text, and PNG drawing
    gfx/                  desktop chrome, windows, cursor, preferences
    ui/                   widgets, charts, and file-picker UI
    shell/                Marble help/about catalog
    mem/                  user-space memory helpers
    software/             repository records, verification, and install transactions
  programs/common/        bundled CLI utilities (echo, cat, help, status, ...)
  programs/fs/            filesystem commands (ls, write, mkdir, ...)
  programs/location/      cwd helper (location)
  programs/gui/           desktop applications and chrome (welcome, taskbar, ...)
  drivers/console/        PL011 console driver
  drivers/block/          virtio-blk block driver
  drivers/display/        virtio-gpu display driver
  drivers/audio/          virtio-sound PCM output driver
  drivers/net/            virtio-net driver
  servers/naming/         name service
  servers/filesystem/     on-disk filesystem server
  servers/netstack/       ARP/IPv4/ICMP/TCP stack and socket server
  servers/display/        compositor and window manager
  servers/input/          virtio-input multiplexer
```

## License 

The license is the Unlicense. As far as motivation, this is a passion project and for educational purposes, nothing more. With this being said, anyone may (and is encouraged to) use GraniteOS 2 for their own purposes. 
