# Project 1: WrapFS - A Stackable Wrapper File System via LKM

A system programming project implementing a custom stackable wrapper file system (WrapFS) within the Linux kernel. This project operates as an intermediate layer between the Virtual File System (VFS) and the lower-level file system (Ext4), intercepting and modifying system calls and file operations to alter file attributes and directory visibility dynamically.

## 🚀 Key Features

* **Dynamic Ownership Translation (Goal 1):** Modifies the `wrapfs_getattr` function in `inode.c` to intercept file attribute retrievals. If a file's UID and GID are detected as `77777`, the system dynamically alters the metadata to reflect the current process's UID and GID (`current_uid()`, `current_gid()`).


* **Directory Entry Hiding (Goal 2):** Implements custom `wrapfs_readdir` and `wrapfs_filldir` functions in `file.c` to intercept directory iteration. It actively filters directory contents, silently hiding any files ending with the `.sslabsecret` suffix from tools like `ls`.


* **VFS Call Chain Interception:** Successfully maps VFS operations to WrapFS handlers, redirecting calls like `f_op->open` to `wrapfs_open`, `f_op->iterate` to `wrapfs_readdir`, and `i_op->getattr` to `wrapfs_getattr` before passing them down to the Ext4 lower file system.



## 🛠️ Technology Stack

* **Environment:** Linux Kernel v5.15, Ubuntu 22.04.5, Oracle Virtual Box, Windows 11 Host.


* **Languages:** C (Linux Kernel Module).


* **Tools:** `dmesg`, `strace`, Bash (Test scripts), Make.


* **Core Concepts:** Virtual File System (VFS), Inode Operations, File Operations, Context Management (`container_of`).



## 🏗️ System Architecture

The project demonstrates a multi-layered architecture where WrapFS transparently intercepts user-space requests passing through the VFS before they reach the actual lower file system.

```text
[ User Space ] `ls -l` command
      |
[ System Calls ] openat(), getdents64(), statx()
      |
[ Virtual File System (VFS) ] path_openat(), iterate_dir(), vfs_getattr_nosec()
      |
[ WrapFS (This Project) ] wrapfs_open(), wrapfs_readdir(), wrapfs_getattr()
      |   <--- (Attribute spoofing & Directory filtering logic applied here)
      |
[ Lower File System (Ext4) ] ext4_file_open(), ext4_readdir(), ext4_file_getattr()

```

## 🧠 Technical Highlights & Troubleshooting

* **Directory Context Wrapping & Callbacks:** Intercepting directory listings to hide specific files required modifying the kernel's `iterate_dir` behavior without breaking its internal state tracking or pointer arithmetic.


* **Solution:** Engineered a custom `wrapfs_dir_context` wrapper containing a `fake_ctx` mapped to a custom `wrapfs_filldir` actor function. By passing this spoofed context to the lower file system, the custom function intercepts every directory entry. The kernel's `container_of` macro was utilized to safely extract the actual context from the `fake_ctx` pointer, allowing normal entries to pass through while silently returning `0` to drop entries ending with `.sslabsecret`.




* **Dynamic Metadata Spoofing via VFS Interception:** The system needed to dynamically alter file attributes (UID/GID) for specific target files without permanently modifying the actual metadata written to the underlying Ext4 file system.


* **Solution:** Hooked the `getattr` inode operation by implementing `wrapfs_getattr`. The function fetches the true `lower_stat` from the Ext4 layer and inspects the values. If the target conditions are met (UID/GID `77777`), it dynamically overrides the `stat` structure in memory using `current_uid()` and `current_gid()` before returning to user-space, achieving transparent on-the-fly attribute translation.

# Project 2: In-Kernel Virtual-IP Load Balancer via eBPF & TC

A high-performance, in-kernel load balancer implemented using extended Berkeley Packet Filter (eBPF). This project operates entirely within the Linux kernel's Traffic Control (TC) subsystem, performing Network Address Port Translation (NAPT) and session-aware traffic distribution without the overhead of context switching to user-space.

## 🚀 Key Features

* **Transparent VIP Routing (Phase 1):** Intercepts client requests targeting a Virtual IP (VIP) and translates the destination to a physical backend server dynamically. Performs symmetric source address translation on egress to maintain TCP state integrity.
* **TCP Flow-Based Load Balancing (Phase 2):** Distributes incoming traffic across multiple backend servers (`main` and `backup`) using a round-robin scheduling algorithm based on atomic flow counters.
* **Session Affinity via BPF Maps:** Utilizes a `BPF_MAP_TYPE_HASH` to store 5-tuple connection signatures (Source IP/Port, Dest IP/Port, Protocol). This ensures that once a TCP 3-way handshake is initiated, all subsequent packets of that flow are consistently routed to the same backend server.
* **Automated Checksum Recalculation:** Safely modifies L3 (IPv4) and L4 (TCP) headers directly in the kernel datapath, leveraging BPF helper functions to recalculate checksums and prevent packet drops.

## 🛠️ Technology Stack

* **Environment:** Linux Kernel v5.15, Ubuntu 22.04
* **Languages:** C (eBPF programs), Bash (Network setup scripts)
* **Tools:** LLVM/Clang, `bpftool`, `tc` (Traffic Control), `iproute2`, `tcpdump`
* **Core Concepts:** eBPF Maps (Array, Hash), Linux Network Namespaces, TC Ingress/Egress Hooks, Atomic Operations, Low-level Packet Parsing.

## 🏗️ System Architecture

The project simulates a hyper-scale infrastructure using isolated Linux Network Namespaces connected via virtual ethernet (`veth`) pairs and a bridge (`br-db`).

```text
[ ns_client ] (10.0.0.2) 
      |
 (veth-cli) 
      |
 [ TC eBPF Hook @ veth-cli-h ] <--- NAPT & Load Balancing logic applied here
      |
   [ br-db ] (192.168.1.1)
      |
      +---> [ ns_db_main ]   (192.168.1.100:18080)
      +---> [ ns_db_backup ] (192.168.1.101:28080)

```

## 🧠 Technical Highlights & Troubleshooting

* **BPF ISA Compiler Optimization:** During the implementation of the atomic global flow counter, the LLVM 14 compiler emitted an `Invalid usage of the XADD return value` error. This was traced to the limitations of the older BPF Instruction Set Architecture (ISA), which supports atomic addition but does not support fetching the previous value into a register simultaneously in a single instruction context.
**Solution:** Refactored the C code to separate the atomic increment (`__sync_fetch_and_add`) from the value retrieval, successfully generating valid BPF bytecode utilizing pure `BPF_STX | BPF_XADD` instructions.
* **Zero-Padding in eBPF Maps:** Ensured strict memory initialization for the `flow_key` structure (`struct flow_key flow = {};`). This prevented garbage data from leaking into the compiler-generated struct padding bytes, guaranteeing deterministic key matching in the BPF Hash Map for session tracking.
