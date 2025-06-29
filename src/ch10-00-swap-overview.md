# Encrypted Swap

Encrypted swap is a solution for small-footprint secure microcontrollers that must rely upon external RAM chips. The idea is to have the fast, on-chip RAM within the secure perimeter serve as the working set of data, but have this backed up with a "swapfile" consisting of slower, off-chip RAM that is also encrypted.

The swap implementation is modelled after the kind of swap space found in other operating systems like Linux. The kernel can over-commit pages of virtual memory, and data is only allocated when a program actually attempts to access the over-committed pages. When the on-chip memory gets full, the swapper will guess as to what pages are not being used and copy them to encrypted swap. The kernel is then free to mark those virtual memory pages as invalid, and re-allocate them to actively used data.

Terminology:

- `wired` refers to pages that are resident in RAM and cannot be swapped out
- `resident` refers to a page in RAM
- `free RAM` refers unallocated internal RAM
- `swapped` refers to a page in swap
- `free swap` refers to unallocated pages in external swap memory
- `reserved` refers to memory that has been over-committed, i.e., exists in virtual memory space but has no physical allocation
- `allocated` refers to memory that has been allocated, but could be in either `resident` or `swapped` states

## Review of Virtual Memory Implementation Without Swap

In Xous, every process has a page table, including the kernel. The `satp` field of a process record stores the page address of the root of a process page table in `satp.PPN`. `satp.ASID` contains the PID.

The kernel is mapped into every process virtual memory space, in the top 16MiB of memory. Thus, the kernel is unique in that it is the only process that can access its physical pages alongside another process.

When each process is created, their page tables are populated with entries that hard-wire the program's code. The stack and heap are also fully allocated, but over-provisioned. Only the first page of each space is backed with physical memory; the rest is demand-paged. Thus, when a program starts, its maximum stack and heap extents are defined by the loader. These extents can be modified at runtime with a kernel syscall, but a program will OOM-fail even if there is physical memory available if its virtual stack and heap allocations are exhausted.

To review, each PTE has the following flags:

- V `valid`: page contents are valid
- R `read`: page can be read
- W `write`: page can be written
- X `execute`: page is executable
- U `user`: page is accessible in user mode
- G `global`: page is in all address spaces
- A `accessed`: page has been accessed (read, execute, or write) (Vex does not implement)
- D `dirty`: page has been written since last time it was cleared (Vex does not implement)
- S `shared`: (part of RWS set) shared page - used by Xous to track if a page has been lent to another process
- P `previously writable`: (part of RWS set) previously writable (not used)

From the standpoint of memory management, a page can only have the following states:

- [Allocated](https://github.com/betrusted-io/xous-core/blob/f389b41ccf3f31d4565b6840403af522ffc16890/kernel/src/arch/riscv/mem.rs#L894-L896): `V` set
- [Fault](https://github.com/betrusted-io/xous-core/blob/f389b41ccf3f31d4565b6840403af522ffc16890/kernel/src/arch/riscv/mem.rs#L901-L903): No flags are set, or `S` is set and `V` is not set
- Reserved: `V` is *not* set, and at least one other flag is set except for `S`

## Encryption Method

Swap is encrypted with an AEAD that is either AES-GCM-SIV or ChachaPoly (the choice will be determined based on benchmarked performance, and can even be changed on the fly since the encrypted information is ephemeral on every boot). The swap encryption key is generated from a hardware TRNG on boot. It is critical that this TRNG function correctly and be truly random early at boot.

The 16-byte AEAD MAC codes for every page are stored in a global appendix in untrusted RAM; this is fine, as the MAC is considered to be ciphertext, as all security derives from the key, but the swap space is reduced by this overhead. Detaching the MAC is done purely as a convenience to simplify page offset mapping computations; the detachment is not meant to imply the MAC is somehow cryptographically separable from the ciphertext block, as would be the case in e.g. a detachable signature scheme.

The nonce for the AEAD is derived as follows:

`nonce[96] = {1'b0|swap_count[31]|pid[8]|p_page[20]|4'b0|v_page[20]|4'b0}`

For performance reasons, no AAD is used.

This gives the following security properties:
- In all cases, pages cannot be replayed between reboots, as the key is generated on each boot
- Tampered pages in external memory are cryptographically likely to be detected (due to the 128-bit MAC code)
- Identical plaintext between two processes will not map to the same ciphertext
- Identical plaintext within a process will not map to the same ciphertext
- Ciphertext between two processes cannot be swapped to do code injection with ciphertext
- Ciphertext of one page within a process cannot be copied to a new page location in the same process and decrypt correctly
- Identical plaintext located in the same physical location and virtual location for the same process with the same swap count will create interchangeable ciphertext
- There is no domain separation. This improves performance slightly (because we don't have to process AAD), but we don't get domain separation. I don't think this is terribly important because the key should be randomly generated each boot, but to improve resistance against potential cross-domain attacks, the swap source data on disk uses a non-null AAD, under the theory that at load time we can better afford the computational overhead of AAD, versus trying to stick it into the core loop of the swapper.

The swap count of a page is a cryptographically small number (nominally 31 bits) that is used to track which page has been least-recently used (to manage evictions). The implementation will be coded to allow a larger number if necessary, but there is a trade-off between the size of this number and the amount of heap space needed to track every virtual page and its swap space; as the swap grows large, the overhead can start to overwhelm the amount of memory available in a small footprint microcontroller.

The last property means that, for example, it is possible to infer something about the heap or stack of a running process that has been swapped out. In particular, we can detect if a region of stack or heap memory has been modified & swapped, and then restored to the original data & swapped, once the swap count is saturated.

This can be used for the following malicious activities:
  - Build a side channel to exfiltrate data via swap
  - Force a running process into a previous stack or heap configuration by recording and restoring pages after forcing the swap count to roll over

Mitigation of this vulnerability relies upon these factors:
  - It takes a long time to push the swap count to 31 bits. This is the equivalent of encrypting and decrypting about 17 terabytes of data using the embedded controller. This would take over 11 years if the microcontroller can run AES-GCM-SIV bidirectionally at a rate of 100MiB/s. An Apple M1 Pro [can achieve 590MiB/s](https://engineering.linecorp.com/en/blog/AES-GCM-SIV-optimization) unidirectionally, so this is an optimistic estimate for an embedded controller running at a few hundred MHz. A similar analysis can be done for ChachaPoly. Note that the swap count is 31 bits because the MSB is used to track if the page is allocated or free.
  - Instead of saturating, the swap count must roll-over. This means that the LRU algorithm will suffer a performance degradation after a "long time" (see previous bullet for bounds on that), but by avoiding saturation it means an attacker has a single window of opportunity to re-use a page before having to roll over again (or they must keep a log of all the pages). This isn't a cryptographically strong defense, but it practically complicates any attack with little cost in implementation.
  - The swap count can be increased to up to 40 bits in size (any larger would overflow the nonce size considering the other data concatenated into the nonce), if further strength is desired, at a considerable price on 32-bit microcontrollers.

## Swap Implementation

Swap is a kernel feature that is enabled with the flag `swap`.

Swap is intended to be implemented using an off-chip SPI RAM device. While it is most performant if it is a memory mapped RAM, it does not need to be; the `swapper` shall be coded with a HAL that can also handle SPI devices that are accessible only through a register interface.

### Image Creation

`swap` configured builds cannot assume that all the static code can fit inside the secure FLASH within a microcontroller.
Thus, the image creator must take regions marked as `IniS` and locate them in a "detached blob" from the main `xous.img`.

The security properties of the two images are thus:
  - `xous.img` is assumed to be written into a secure on-chip FLASH region, and is unencrypted by default.
  - `swap.img` is assumed to be written to an off-chip SPI FLASH memory, and is untrusted by default.

A simple bulk signature check on `swap.img`, like that used on `xous.img`, is not going to cut it in an adversarial
environment, because of the TOCTOU inherent in doing a hash-and-check and then bulk-copy over a slow bus like SPI.
Thus, the following two-phase scheme is introduced for distributing `swap.img`.

The AEAD shall be either ChachaPoly or AES-256, depending upon which is more performant (to be updated). We use a "detached-MAC" scheme only because it makes mapping block offsets in the ciphertext stream to block offsets in the plaintext stream logically easier. There's no cryptographic meaning in detaching the MAC.

1. In phase 1, `swap.img` is encrypted using an AEAD to a "well-known-key" of `0`, where each block in FLASH encrypts a page of data, and the MAC are stored in an appendix to the `swap.img`. The first page is an unprotected directory that defines the expected offset of all the MAC relative to the encrypted blocks in the image file, and contains the 64-bit nonce seed + AAD. The problem of whether to accept an update is outside the scope of this spec: it's assumed that if an update is delivered, it's updated with some signature tied to a certificate in a transparency log that is confirmed by the user at update time. This does mean there is a potential TOCTOU of the bulk update data versus signing at update time, but we assume that the update is done as an atomic operation by the user in a "safe" environment, and that an adversary cannot force an update of the `swap.img` that meets the requirements of phase 2 without user consent.
2. In phase 2, `swap.img` is re-encrypted to a locally generated key, which is based on a key stored only in the device and derived with the help of a user-supplied password. This prevents adversaries from forcing an update to `swap.img` without a user's explicit consent. NOTE: the key shall *not* be re-used between `swap.img` updates; it should be re-generated on every update. This does mean that the signature on the `xous.img` shall change on every update, but this is assumed to be happening already on an update.

In order to support this scheme, the `Swap` kernel argument contains a `key` field. The image creator sets this to a 256-bit "all zero" key initially for distribution and initial device image creation. Once the device is provisioned with a root key, the provisioning routine shall also update the kernel arguments (which are stored inside the secure FLASH region) with the new key, and re-encrypt the `swap` detached blob in SPI FLASH to the unique device key before rebooting.

If followed correctly, a device is distributed "naive" and malleable, but once the keying ceremony is done, it should be hard to intercept and/or modify blocks inside `swap.img`, since the block-by-block read-in and authentication check provides a strong guarantee of consistency even in the face of an adversary that can freely MITM the SPI bus.

This is different from the detached-signature with unencrypted body taken for on-chip FLASH, which is a faster, easier method, but only works if the path to FLASH memory is assumed to be trusted.

The nonce for the `swap.img` AEAD is 96 bits, where the lower 32 bits track the block offset, and the upper 64 bits are the lower 64 bits of the git commit corresponding to the `swap.img`. This 64-bit git commit is stored as a nonce seed in the unprotected header page along with the offset of the MAC + AAD data. The incorporation of the git commit helps protect against bugs in the implementation of the locally generated key. The locally generated key should not be re-used between updates, but tying the nonce to the git commit should harden against chosen ciphertext attacks in the case that the generated key happens to be re-used.

The AAD shall be the ASCII string 'swap'. I don't think it's strictly necessary, but might as well have domain separation.

Thus the swap image has the following layout:
- `0x0:0xFFF`: unencrypted header containing the version field, partial nonce, mac offset, and the AAD.
- `0x1000:0x1FFF`: Encrypted XArgs description of the swap blob, padded to a page
- `0x2000:...`: Successive `IniS` images.

Note that this format introduces two offsets in the swap data:

1. Offset from disk start to start of encrypted images, equal to 0x1000.
2. Offset from encrypted image start to `IniS` start (e.g., space for the XArgs block)

Note that if the XArgs block overflows its page of space, the loader may not handle this gracefully. There is an assert
in the image creator to catch this issue.

### Boot Setup: Loader

The loader gets new responsibilities when `swap` is enabled:
- The loader needs to be aware of both the location and size of the trusted internal unencrypted RAM (resident memory), and the external encrypted RAM (swap memory).
- The resident memory is tracked using the existing "Runtime Page Tracker" (RPT) mechanism.
- Additional structures are created and mapped into PID2's memory space:
   - `0xE000_0000`: The "Swap Page Tables" (SPT) root page table. This is like the swap's "satp". It contains an array of `satp`-like pages that form the root pointer for each process' virtual swap address space. Follow-on pages for 2nd-level page tables are also mapped into the `E000_0000` range, but the exact location and amount varies depending on the amount of swap needed.
   - `0xE100_0000`: The swap arguments. This contains all of the arguments necessary to initialize the swap manager in userspace itself, and is a memory-map of the `SwapSpec` struct
   - `0xE100_1000`: A copy of the RPT, except with `wired` memory marked with a PID of 0 (pages marked with the kernel's PID, 1, are free memory; the kernel code itself is marked 0 and `wired`). The size is fixed, and is proportional to the total size of internal (`resident`) RAM.
- All of these structures must be mapped into PID2's memory space by the loader
- The loader is responsible for querying the TRNG on every boot to generate the session key for encrypting off-chip RAM.
- A new image tag type is created `inis`, to indicate data that should start in encrypted swap.
- A kernel argument with tag `swap` is created. It contains the userspace address for PID2 (the swapper) of the SPT, SMT, and RPT structures.

The SPT has the same structure as system page tables. However, SPT entries are only allocated on-demand for processes that have swap; it is not a full copy of every page in the system page table.

#### INIF handling

Regions marked as `xip` (i.e, marked as `inif` type) are assumed to be FLASH that is contained within the secure perimeter of the microcontroller. These regions are mapped directly and unencrypted into the kernel memory space.

These boot-critical processes must be `xip`, and are never swapped out:

- kernel
- swapper
- ticktimer

More regions can be marked as `xip`; this is preferable, because if the code is resident in on-chip FLASH, it isn't taking up working-set RAM and things will run faster.

The initial data set of `inif` processes are considered to be `wired`; however, their reserved memory regions can be swapped after allocation.

#### INIE handling

Regions marked as `inie` are copied to the working set RAM and executed out of RAM. It is assumed this tag is mostly unused in microcontrollers with a small internal working set, but the behavior is not modified because it is a valid and useful tag for devices with large, trusted RAMs.

#### INIS handling

Regions marked as `inis` are copied into encrypted swap on boot. The kernel page table state starts out with the correct `R`/`W`/`X`/`U` values, but `V` is not set, and `P` is set. Entries are created in the SPT to inform the tracker where to find the pages.

An kernel argument of type `swap` is provided, which is a base and bounds to the SPT/SMT region. This is meant to be passed to the `swapper` process when it registers to the kernel.

The image creation routine and kernel arguments need to be extended to support `inis` regions located in off-chip SPI FLASH. The off-chip data is not encrypted, but it is signature checked with a dedicated signature block. Note that the off-chip SPI FLASH does not need to be memory mapped: the loader may read the memory through a register interface.

### Kernel ABI

Systems with `swap` enabled must have a process located at `PID` 2 that is the `swapper`. The kernel will only recognize `swap` extension syscalls originating from `PID` 2. The kernel page fault handler must also be extended to handle swapped pages by invoking the swapper to recover the contents.

The following kernel syscall extensions are recognized when the `swap` feature is activated:

- `RegisterSwapper`
- `SwapOp`

`RegisterSwapper` establishes the legitimacy of the swapper process; `SwapOp` is a wrapper around a swapper ABI that can change and evolve.

The swapper shall not allow any shared-state locks on data structures required to satisfy a swap request. Such a lock will lead to a system hang with no error message, since what happens is the `swapper` will busy-wait eternally as preemption has been disabled.

The general flow of swap handling is as follows:

- The userspace swapper handler is registered. This must happen as early in the boot process as possible.
- An event generates a swap exception. Sources of events include page faults on a swappable page, out of memory, and garbage collection events.
- Events from within the kernel must issue a `swap_reentrant_syscall()` call, instead of a regular syscall.
- The swapper handles the events, and returns.

#### `swap_reentrant_syscall`

`swap_reentrant_syscall()` allows the kernel to use the `SwapOp` API via a syscall interface. This is important because anything can run out of memory, and anything can encounter a swapped page, including the kernel in its most tender moments (such as an OOM while allocating new page table entries).

Thus in order to make the swap implementation sane, we need to be able to make system calls from inside the kernel and from inside interrupt handlers.

Xous has an "exception handler" context for system calls. It is not set up for nested operations. In implementing swap, we could either introduce nesting by allocating a separate stack space for nested calls and return vectors for the same, or we could just fake it by backing up the stack before doing a re-entrant call.

While the former strategy sounds elegant, it would require patching every syscall path with something that reads a piece of state to determine which level of the nesting stack you're on. In particular, the assembly stub that is responsible for setting up the exceptions would need to be reworked to do this. This was deemed to be more invasive and more bug-prone than the alternative.

So, in this implementation, `swap_reentrant_sycall` has an assembly stub which runs just before entering a re-entrant syscall, and also just after. The routine reads the current stack pointer, copies the exception handler's entire stack to a backup location, does the syscall, and then on return, restores the stack's contents. This minimizes code changes to other code paths (reducing analytical complexity) in exchange for an operation that is extremely risky but analytically tractable.

#### RegisterSwapper Syscall

The `swapper` registers with the kernel on a TOFU basis. Note that all the data necessary to setup the swapper is placed in the swapper's memory space by the loader, so the kernel does not need to marshall this.

The arguments to `RegisterSwapper` are as follows:

- `handler` is the virtual address of the entry point for the blocking swap handler routine in the swapper's memory space
- `state` is the virtual address of a pointer to the shared state between the swapper userspace and the swapper blocking handler

All communications between the kernel and the swapper happen through the `handler` entry point.

#### SwapOp Syscall

The `SwapOp` syscall encodes a private ABI between the swapper and the kernel. The idea is that this ABI can rapidly evolve without having to update the syscall table, which would require an update to the Xous package itself. The `SwapOp` syscall has arguments consisting of the op itself, which is coded as the numerical representation of `SwapAbi` (below), and up to 6 generic `usize` arguments that have a meaning depending on the `SwapAbi` code.

The `SwapAbi` may change rapidly, so please refer to the code for the latest details, but below gives you an idea of what is inside the ABI.

```rust
pub enum SwapAbi {
    Invalid = 0,
    ClearMemoryNow = 1,
    GetFreePages = 2,
    RetrievePage = 3,
    HardOom = 4,
    StealPage = 5,
    ReleaseMemory = 6,
}
```

`ClearMemoryNow` is the call used when the system has run out of physical memory. Only the swapper is allowed to invoke this call; other processes may proxy a request for memory through the swapper's userspace server. This call stops all interrupts, and immediately enters the userspace handler to nominate pages to evict in hopes of recovery.

`GetFreePages` returns the number of physical pages currently unallocated.

`RetrievePage` is used to retrieve a page that has been previously swapped out into physical memory. It must be called with the target PID and virtual address of the page to be retrieved, along with a physical address of where to put the retrieved page.

`HardOom` is the kernel interface to `ClearMemoryNow`. It is what is used to demand a page when another page operation requires it, e.g. you're in the middle of a lend, and the target process needs an L1 page table page, but you're out of free pages; `HardOom` will try to free up some pages to allow an operation like this to proceed. Unlike `ClearMemoryNow`, this call can originate from any process ID, but only from within the kernel.

`StealPage` instructs the kernel to mark a specified page in the victim process' memory space as swapped, and return its contents for storage in swap. The userspace swapper is responsible for swap strategy, and this is one half of the call that it uses to execute the strategy.

`ReleaseMemory` instructs the kernel to mark a specified physical page in the swapper's memory space as no longer used, returning it to the free memory pool. This is called after a successful `StealPage` call followed up by archival of the swapped page to encrypted swap.

Most of the interesting code for the swapper is split between the kernel stub (inside kernel/src/swap.rs) and the userspace service (services/xous-swapper). The userspace service itself is split into the blocking handler and the regular preemptable handler, with most of the action happening in the blocking handler. The blocking handler is named such because it happens in an interrupt-like context: no pre-emption of any type is allowed.

There are also a couple of modifications to the architecture-specific implementations (kernel/src/arch/riscv/irq.rs and kernel/src/arch/riscv/mem.rs) to shim the swap into core memory routines. Keep in mind at least half the effort for swap is in the loader, which is responsible for setting up and initializing all the relevant data structures so that swap is even possible.

#### Flags and States

When `swap` is enabled, the flags have the following meaning:

- V `valid`: page contents are valid
- R `read`: page can be read
- W `write`: page can be written
- X `execute`: page is executable
- U `user`: page is accessible in user mode
- G `global`: page is in all address spaces
- A `accessed`: page has been accessed (read, execute, or write) (Vex does not implement)
- D `dirty`: page has been written since last time it was cleared (Vex does not implement)
- S `shared`: (part of RWS set) shared page - used by Xous to track if a page has been lent to another process
- P `swapped`: (part of RWS set) page is in swap

From the standpoint of memory management, a page can only have the following states:

- `Allocated`: `V` set, `P` may not be set
- `Fault`: No flags are set, or `S` is set and `V` is not set
- `Swapped`: `P` is set. `V` is *not* set. `S` may also not be set. Upon access to this page, the kernel allocates a resident page and calls `ReadFromSwap` to fill it. The page will move to the `Allocated` state on conclusion.
- `Reserved`: `V` is *not* set, `P` is *not* set, and at least one other flag is set except for `S`. A kernel allocates a resident page and zeros it. The page will move to the `Allocated` state on conclusion.

Pages go from `Allocated` to `Swapped` based on the `swapper` observing that the kernel is low on memory, and calling a series of `StealPage`/`ReleaseMemory` calls to free up memory. The kernel is allowed to run out of memory and invoke the swapper to free memory for its operations through the use of re-entrant syscalls.

Pages go from `Allocated` to `Reserved` when a process unmaps memory.

The kernel can OOM and recover by pushing pages to swap. When the `swapper` runs out of swap space, `write_to_swap_inner()` inside the userspace handler will panic with an out of swap message, and the system will halt.
