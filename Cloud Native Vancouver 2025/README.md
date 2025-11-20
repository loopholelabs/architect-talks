# Scale to Zero and Live Migration of Stateful Apps on Kubernetes

## Summary

Checkpoint/restore technology captures the complete state of a running app and allows it to be restored later without losing memory contents, open file descriptors, or other app state. This talk will give a quick overview of what C/R is useful for, how it can be used for scaling apps to zero and migrations, how it is implemented in Linux, and how you can use it in your Kubernetes cluster.

## Overview

Having C/R support means that apps with long initialization times can skip startup costs by restoring from a checkpoint. Python apps can skip reading source files, Spring Boot can amortize its initialization, interpreters are always in their initialized state, and LLM weights don't need to be read from disk first.

C/R also enables scaling idle apps to zero so they release their resources. Cluster autoscalers can then remove unused nodes and add them back when demand increases, without losing app state.

It also can be used to do live migrations, where stateful apps can be moved between nodes without downtime. This makes a lot of things much easier: App state isn't lost when nodes are drained, they can be live migrated off a node before maintenance is done, and binpacking becomes much easier since moving an app to a node with available space can be done without losing app state.

A few technologies allow for C/R on Linux. CRIU is the most popular one; it serializes kernel resources into protocol buffers and restores them by transmuting a virtual process back into the original, which integrates well with OCI runtimes. gVisor virtualizes syscalls like a microkernel, providing more control but with a performance penalty since syscalls run in userspace. Firecracker provides VM snapshot/restore with high reliability but requires KVM, limiting it to bare-metal. PVM may eventually support non-bare-metal/non-nested virtualization environments.

As an example, we'll demo how Architect implements checkpoint/restore for Kubernetes using this tech. Runtime classes allow you to opt in to using checkpoint/restore, while annotations and labels control scaling and migration behavior. Scale to zero is triggered by listening for network traffic via XDP and the existing CNI, and migrations are triggered by a pod being deleted and re-created with the same pod template hash. Karpenter and cluster autoscalers can then right-size the cluster automatically. We'll demo some examples like a Go service, Valkey/Redis, Postgres, and Minecraft servers being migrated during the talk.

## Speaker Notes

- Introduction
  - Felicitas Pojtinger
  - Head of R&D at Loophole Labs
  - Loophole Labs does live migration of various workloads for lots of different use cases
  - I work on the process migration part and I've previously implemented the first network migration and VM migration implementation at Loophole Labs
  - Personal: https://felicitas.pojtinger.com/, https://github.com/pojntfx/, https://mastodon.social/@pojntfx, https://www.linkedin.com/in/pojntfx/ and https://bsky.app/profile/felicitas.pojtinger.com
  - Company: https://loopholelabs.io/, https://www.linkedin.com/company/loophole-labs, https://github.com/loopholelabs and https://x.com/LoopholeLabs
- What is C/R?
  - Captures the complete state of a running app into a file
  - Allows restoring from that file at a later time without losing anything
    - Memory contents
    - CPU registers
    - Open file descriptors
    - Devices (e.g. an accelerator like a GPU)
- What will the talk be about?
  - Usecases for C/R
    - Amortizing startup time
    - Scaling apps to zero when they are not needed
    - (Live) migrations
  - How C/R is implemented
    - CRIU
    - gVisor
    - In KVM via Firecracker/Cloud Hypervisor etc.
  - Using C/R in Kubernetes with Architect
- Usecases for C/R
  - Amortizing startup time
    - Apps with long initialization times can skip startup costs by restoring from a checkpoint
    - Python: Skip reading source files one by one into memory
    - Spring Boot: No need to warm up the JVM and do other initialization
    - Interpreters: No need to initialize memory or load things from disk
    - LLM inference: Weights don't need to be read from disk separately or interpreters initialized
  - Scaling apps to zero
    - A lot of apps don't always need to be running
    - Websites that serve a regional market where people go to sleep
    - Servers than fan out during peak times and then run with far fewer resources
    - Sometimes, just stopping and restarting servers is the right answer, but a lot of apps take a while to start so that is too expensive
    - With checkpoint/restore, the restart becomes instant as it resumes right from where it left off
    - When an app is checkpointed, it's resources get scaled down to zero
    - Kubernetes then sees its cluster-wide resource usage go down
    - Nodes get automatically released back to the cloud provider's node pool
    - When the app restores from a checkpoint, the resources get scaled back to what they were before
    - Kubernetes then sees its cluster-wide resource usage go up
    - Nodes get automatically added back from the cloud provider's node pool to the cluster
  - Live migrations
    - Usually, moving between infrastructure means that you lose state
    - There is a level of inherent "cost" to moving apps between infrastructure as a result
    - With C/R, we can snapshot apps on the source of such a move, and then just restore on the destination
    - That means the inherent cost is gone - you can move at any time
    - It's now much easier to take a physical node ofline for maintenance by migrating workloads off it and then just move them back afterwards
    - Binpacking also becomes much easier since apps can just be migrated to another available spot on another node that just happens to fit the app
- Checkpoint/Restore Implementations for Linux
  - CRIU
    - Checkpoint/Restore in Userspace
    - Contains of (mainlined) patches to the kernel that expose kernel resources to userspace
    - A userspace program then goes ahead and uses those kernel interfaces to checkpoint and restore
    - Checkpoint
      1. CRIU seizes all threads in the target process tree (you pass in a PID to CRIU), which freezes them in place/stops time so that things don't drift
      1. Injects code snippets into target process ("infection") to dump register state and reading hard-to-access memory regions
      1. Reads `/proc/pid/maps` to find out what memory the process allocates
      1. Uses that information to dump memory contents from `/proc/pid/mem` etc. to disk, along with permissions, flags, and information about which processes share memory with which other process
      1. Goes through all file descriptors (files, sockets, pipes and special files) and dumps positions, flags, paths, connection state, buffers etc. to disk
      1. Gets namespace memberships (PID, network, mount, user etc.), CPU register state and credentials (capabilities, signal handlers etc.) and dumps them to disk
      1. Reads kernel state from `/proc` and `/sys` on the process and dumps it into the file
      1. After checkpointing, process is either left running (unfreezes the process tree/detaches from it with ptrace) or killed
    - Restore
      1. Recreates the process tree as a skeleton with fork (so that parents and children have the right relationship)
      1. Locks, then sets `/proc/sys/kernel/ns_last_pod` to desired `PID - 1` to restore processes with the original PID when possible (see https://www.criu.org/index.php?title=Pid_restore&mobileaction=toggle_view_desktop)
      1. Recreates or joins (enters with `setns`) namespaces that were set (PID, network, mount, user etc.) just like they were before the checkpoint
      1. Reopens files at original paths with the flags and positions, recreates pipes, sockets and TCP connections
      1. Restores in-flight data in pipes and buffers by manipulating kernel state
      1. Remaps memory regions (and mmap'ed files) at the original addresses with the right permissions, then fills memory from the memory images
      1. Restores CPU registers including the instruction pointer (processes resume exactly where they left off)
      1. Restores credentials, signal handlers etc.
      1. Unpauses the restored process by jumping into the restored instruction pointer, which resumes it exactly
    - CRIU is great because it doesn't incur any overhead to the process at runtime - it's just a process like any other until you start snapshotting
    - Checkpoints are somewhat portable across hosts
    - It is hard to checkpoint an _arbitrary_ application, or a very complex application, e.g. one that uses complex forms of shared memory or `io_uring`
    - Needs careful instrumentation and workarounds to work
  - gVisor
    - Unlike containers, gVisor provides "real" sandboxing support
    - Virtualizes syscalls, filesystems and networking for much better security than containers
    - How it starts processes
      1. Reads the OCI runtime spec (`config.json`) just like any other OCI runtime would
      1. Creates a new process for the sandbox and puts it into a network namespace, configures resources with cgroups
      1. Starts gofer process (a filesystem process) with very limited capabilities (no `CAP_SYS_ADMIN` etc.), which limits FS access (if Sentry is compromised, the gofer still limits how much the attacker can access)
      1. Starts Sentry (the sandbox component) in the sandbox process we've started earlier
      1. Uses either KVM, ptrace or systrap to intercept syscalls
      1. Sets up a gVisor kernel (task scheduler, signal handlers, VFS etc.)
      1. Mounts emulated filesystems via the gofer (e.g. /proc, /sys and so on)
      1. Sets up overlayfs and mounts other volumes
      1. Sets up netstack (Go implementation of the kernel's network stack in userspace)
      1. Sets up Linux seccomp filters on the sandbox (Sentry) - Sentry can only do a few bare minimum syscalls (read/write) and not do any other ones (exec etc.)
      1. Loads the container's binary (PID 1) and sets up memory mappings, the stack, interpreter etc.
      1. Creates an init task in the kernel (with memory, file descriptors, FS context, credentials and signal handlers)
      1. Unblocks init task on startup
    - How a sandboxed process running in the sandbox accesses resources
      - For syscalls, the platform (KVM, ptrace, systrap) catches it and calls the relevant handler in Sentry, which then emulates the Linux kernel behavior and returns the result to the sandboxed process
      - For filesystem access, the VFS layer receives a request, the Gofer FS implementation sends an RPC to the gofer process, which then does the actual host syscalls and returns the result to the sandboxed process via the VFS later
      - For networking, socket syscalls go into netstack (the userspace network implementation), which then processes packets in a goroutine and writes them to veth devices via `AF_PACKET`
    - Checkpoint
      1. Get a request via the sandbox control socket (sent by `runsc checkpoint` etc.) in the sandbox controller
      1. Pauses all tasks by sending an internal `SIGSTOP` signal and waiting until all task goroutines have stopped
      1. Stops all timers so that time doesn't drift and flushes all async operations, waits for gofers and netstack to finish
      1. Serialize gVisor kernel state like PIDs, CPU register states, credentials, IPC objects and namespaces
      1. Serialize memory addresses and file descriptors, much like how CRIU does it, except using the virtualized resources, not the actual Linux kernel resources
      1. Serialize filesystem state from the VFS, e.g. mount tables, host paths that correspond to the gofer-virtualized paths. The pseudo-filesystems (`/proc`, `/sys` etc.) are regenerated from on restore from the rest of the gVisor state
      1. Serialize network state with netstack (TCP connection state, UNIX sockets etc.)
      1. Serialize virtual device state (PTYs etc.)
      1. Write into checkpoint image files based on protobuf schema
      1. After checkpointing, process is either left running (remove stop flags, which unpauses the process) or killed & the sandbox torn down
    - Checkpointing in gVisor is a bit different from CRIU
      - No need for PID manipulation - gVisor controls the PID namespace, so no need to modify the kernel
      - No "infection"/injecting code into the actual process
      - Sentry has internal structs for kernel resources, which are much easier to serialize than kernel state
      - The network suspend/restore implementation is more complete since it doesn't rely on the kernel TCP implementation
    - Restore
      1. Sandbox process, Sentry and gofer get started just like with the initial start, with minimal state
      1. Checkpoint image files get read into memory, where the state decode then parses the format and unmarshals into the relevant serializable gVisor components
      1. Kernel state gets restored by iterating through tasks and creating goroutines for each, using the same internal PID as on checkpoint
      1. CPU registers, signal handlers, credentials and FS context get set back in the gVisor resources
      1. Memory managers get recreated in the same virtual address ranges as the checkpoint with the same flags, and memory pages get read from the checkpoint files
      1. mmap-ed regions backed by an actual file get re-mapped to the same path on the host
      1. The filesystem state gets reconstructed from the checkpoint and the gofer filesystems get reconnected, tmpfs files get restored from checkpoint, `/proc` and `/sys` get re-generated from the now restored gVisor state
      1. File descriptor tables get restored for each task, by gofer re-opening them on the host or reading them from the checkpoint data for tmpfs
      1. Network state gets restored by creating a new netstack instance and creating NICs and routing tables with the stored data from the checkpoint
      1. TCP sockets, UNIX sockets etc. get restored in netstack
      1. Devices (PTY etc.) get restored by recreating them based on the checkpoint data
      1. To resume, the kernel clears the restore flag (similar to the stop flag), which then iterates through each task, sets registers with the instruction pointer, and jumps to the instruction pointer (depending on the platform, by executing and monitoring it with seccomp for systap, and in KVM by making the guest CPU do that) to restore the task exactly where it left off
  - Firecracker
    - Starts lightweight VMs, not processes ("microVM")
    - Full KVM-based isolation for great security properties
    - Comes with some overhead as a result of this since you'll be running a nested kernel, but that kernel is very minimal and uses next to no resources - you can run thousands of such VMs on a single node
    - CPUIDs can be set with CPU templates so that multiple different hosts can be reduced to their lowest common denominator
    - Needs some sort of hardware acceleration, so it only works with bare metal (unless PVM is used)
    - How it starts a microVM
      1. Receives a REST API call that configures the internal VM configuration
      1. Opens `/dev/kvm` and creates a VM file descriptor via `KVM_CREATE_VM`
      1. Allocates guest memory via mmap and registers the new memory regions via the KVM devices
      1. Creates vCPUs for each CPU via the KVM devices
      1. Loads the Linux kernel into guest memory
      1. Initializes virtio devices (disks/block devices, net, entropy etc.) and registers them on the VM bus
      1. Configures the system for boot (setting up ACPI (x86)/Device Tree (aarch)) and spawns a (paused) thread for each vCPU
      1. Applies seccomp filters to the VMMs and vCPU threads for better sandboxing (similar to how the Sentry is sandboxed in gVisor), then unpauses the vCPU threads and KVM begins executing the guest kernel
    - How a microVM accesses host resources
      - Code runs on the physical CPU (sandboxed via VT-d/VT-x)
      - (Block) device I/O happens via virtio devices, which then gets handled on the host via interrupts
      - For networking, TAP devices are usually used on the host, which can then be put into network namespaces etc. for sandboxing
    - Checkpoint
      1. Receives a REST API call that pauses all vCPUs, which causes all vCPU threads to exit from `KVM_RUN`
      1. Sends a `SaveState` event to each vCPU thread, which dumps CPU state (e.g. registers, MSRs, LAPIC state, TSC frequency, CPUID leaves etc.)
      1. Dumps VM-level states (guest memory etc.) and virtio device state from the host (serializes queues and configuration, e.g. MAC address or TAP device names)
      1. Serializes device state into a snapshot file and guest memory into a separate file
      1. After checkpointing, VM is either left running (by resuming the vCPU threads) or stopped completely
    - Checkpointing here is different from both CRIU and gVisor
      - Firecracker controls the entire guest CPU and devices, so it can checkpoint a much simpler and well-defined interface
      - No need for TCP restore per se since the whole network device is checkpointed
      - No need for file descriptors to be dumped or syscalls to be paused/resumed in the guest, all of that state is just stored in the guest memory
      - No need for a virtual file system like with gVisor since the well-defined virtio devices exist
      - Memory can be lazy-loaded since it is just an `mmap`ed region and doesn't have to be read by the kernel
    - Restore
      1. Receives REST API call that opens the snapshot state file and deserializes it
      1. Loads guest memory with `mmap` (so that it can be lazy-loaded), either from a file (for non-anonymous memory) or initialized with zeros
      1. Reads vCPU counts, memory sizes, CPU templates etc. and configures KVM like at startup, just using the values from the checkpoint files
      1. Restores vCPUs by setting TSC frequencies and registers for each vCPU from the checkpoint
      1. Restores the Firecracker implementations of the virtio devices from the checkpoints by deserializing them, (re-)opening the TAP devices, setting the MAC addresses and recreates the queues
      1. Like with the original startup process, it applies seccomp filters to the VMMs and vCPU threads for better sandboxing (similar to how the Sentry is sandboxed in gVisor), then unpauses the vCPU threads and KVM begins executing the guest kernel
  - PVM
    - Usually, you need to have hardware virtualization support in order for you to start VMs (AMD/Intel's VT-d/VT-x)
    - You can use emulation, but that makes things very slow/high-latency
    - Nested virtualization also exists, but there are very heavy performance penalties with that approach and some security concerns (almost no cloud provider enables it)
    - PVM fixes this, it implements a KVM vendor implementation (like VT-d) that works similarly to how we sandbox processes in user space with a new method called shadow paging
    - How PVM works
      - Code execution of the guest kernel happens in Ring 3 (just like a regular process), the guest kernel vs. host kernel distinction is done via software flags and shadown page tables
      - Syscalls from guest userspace to the guest kernel trap to the in-kernel switcher component, which switches CR3 (page table pointer, from guest user to guest kernel page tables) and GSBASE (per-CPU/thread-local data pointer), then returns directly to the guest kernel's syscall handler via SYSRETQ without hypervisor involvement
      - Hypercalls from the guest kernel use the SYSCALL instruction (in supervisor mode), which traps to the hypervisor and are handled as either PVM-specific hypercalls (page table switching, TLB flushes, MSR access) or standard KVM hypercalls
      - For memory access, PVM maintains a shadow page table which contains the physical addresses of where things are on the host, changes to the guest page table get intercepted and make PVM update the shadow tables
      - Exceptions/interrupts trap to the hypervisor, which either handles them for shadow MMU/emulation (page fault etc.) or injects them back to the guest via its event delivery mechanism
    - It is very performant in CPU-heavy workloads, but has some impacts on performance in fork-heavy workloads (think Linux kernel compliations where `make` calls fork a lot of times)
    - Not a mainline patch yet, but a patchset for Linux 6.12 (LTS) is available
    - We maintain a few Firecracker patches that support C/R with Firecracker and PVM: https://github.com/loopholelabs/firecracker/tree/main-live-migration-pvm
    - Sources: https://github.com/virt-pvm/linux, https://github.com/virt-pvm/misc
    - It implements a lot of the KVM vendor implementation interface, enough for the Firecracker C/R implementation to basically just work™!
  - Network migration via XDP
    - Usually, during a network migration, connections break
    - We use XDP on both ingress and egress to intercept traffic
    - We use XDP in driver mode, which means the eBPF programs run inside the network interface driver in the kernel on the host machine CPU (not on the NIC itself - that would be offload mode, which only works on an extremely small subset of cards with XDP-supporting DPUs)
    - The efficiency gains come from skipping the normal kernel network stack steps, allowing us to intercept packets very early in the processing pipeline
    - In our testing we were able to get up to 200 Gb/s: https://loopholelabs.io/blog/xdp-for-egress-traffic
    - Once we're in the data path:
      1. We can intercept traffic
      1. Buffer it
      1. Redirect it etc. which allows us to pause traffic before a checkpoint
      1. Do the checkpoint
      1. Resume on a different (in the case of a migration) or the same (in the case of a scale to zero operation) node,
      1. Unpausing traffic while flushing buffers, effectively migrating a connection without causing any downtime
    - Even if we're moving between nodes we can migrate the connections as long as we're in the data path somewhere between the user and the server, which thanks to eBPF & XDP is quite scalable
    - No opt-in is required from applications - anything, whether it's TCP or UDP, can be migrated this way
- Demo of C/R on Kubernetes with Architect
  - Signing up via the console
  - Installing it via Helm
  - Adding the runtime class and annotations to the example application
  - Scaling the application down to zero and scaling it back up again
  - Migrating the application between two nodes by deleting the pod and having the deployment
  - Demo: Video of the Minecraft demo at KubeCon NA 2023 and 2024
