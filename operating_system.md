# ECE469: Operating System Engieering

This is a course focus on Operating System, Adapted from MIT6.828, using JOS as primary lab materials.

## Note

#### PC's physical address space.

    +------------------+  <- 0xFFFFFFFF (4GB)
    |      32-bit      |
    |  memory mapped   |
    |     devices      |
    |                  |
    /\/\/\/\/\/\/\/\/\/\

    /\/\/\/\/\/\/\/\/\/\
    |                  |
    |      Unused      |
    |                  |
    +------------------+  <- depends on amount of RAM
    |                  |
    |                  |
    | Extended Memory  |
    |                  |
    |                  |
    +------------------+  <- 0x00100000 (1MB)
    |     BIOS ROM     |
    +------------------+  <- 0x000F0000 (960KB)
    |  16-bit devices, |
    |  expansion ROMs  |
    +------------------+  <- 0x000C0000 (768KB)
    |   VGA Display    |
    +------------------+  <- 0x000A0000 (640KB)
    |                  |
    |    Low Memory    |
    |                  |
    +------------------+  <- 0x00000000


### BIOS
The BIOS will handle __basic IO initialization__, and pass the process to the bootloader. 

It used __16 bit real mode__, with __[CS:IP]__ format for addressing. Ends with magic byte of __0x55AA__.

### Bootloader
- 32bit protected mode
- Use virtual memory, index using GDT (global descriptor table)

#### ELF

The ELF (Excutable and Linkable Format) is defined at `inc/elf.h`. It contains loading information. 

- ELF __Section Header__, which contains program layout, checked with `objdump -h obj/kern/kernel`. Example: `.rodata` means Read-only data; `.data`means data section holds the program’s initialized data; `.bss`means section reserved for uninitialized global variable...

```
Kernel Binary Layout:
┌─────────────────────┐
│      .text          │  (code)
├─────────────────────┤
│      .data          │  (data)
├─────────────────────┤
│      .stab          │  ← __STAB_BEGIN__ points here
│  ┌───────────────┐  │
│  │ Stab entry 0  │  │  Index 0
│  │ Stab entry 1  │  │  Index 1
│  │ Stab entry 2  │  │  Index 2
│  │     ...       │  │
│  │ Stab entry N  │  │  Index N
│  └───────────────┘  │
│                     │  ← __STAB_END__ points here
├─────────────────────┤
│     .stabstr        │  ← __STABSTR_BEGIN__ points here
│  "kern/init.c\0"    │
│  "i386_init\0"      │
│  "cons_init\0"      │
│       ...           │
│                     │  ← __STABSTR_END__ points here
└─────────────────────┘
```
- `VMA`: linked address. Execution assumes section is here
- `LMA`: load address. where section is placed when loaded
- ELF __Program Header__, which describes how loader should work for the program, check with `objdump -p obj/kern/kernel`. Example:
```
obj/kern/kernel:     file format elf32-i386

Program Header:
    LOAD off    0x00001000 vaddr 0xf0100000 paddr 0x00100000 align 2**12
         filesz 0x00007f46 memsz 0x00007f46 flags r-x
    LOAD off    0x00009000 vaddr 0xf0108000 paddr 0x00108000 align 2**12
         filesz 0x0000b6e1 memsz 0x0000b6e1 flags rw-
   STACK off    0x00000000 vaddr 0x00000000 paddr 0x00000000 align 2**4
         filesz 0x00000000 memsz 0x00000000 flags rwx
```


### Kernel
The bootloader will handle the process to the kernel.

#### Stack

The `stack` is initialized in `kern/entry.S`, as well as `ebp` and `eip`.

- `kern/entrypgdir.c` map virtual memory in the kernel, once paging enabled in `kern/entry.S`
- `ebp`: base pointer, the position when function enters
- `eip`: instruction pointer, return address of a function call
 

#### STAB
 __Stab__ is a format to descript a program to a debugger. 

 After running `objdump -h obj/boot/boot.out`, we see:
 ```
 2 .stab         000006c0  00000000  00000000  0000029c  2**2
                  CONTENTS, READONLY, DEBUGGING
  3 .stabstr      00000466  00000000  00000000  0000095c  2**0
                  CONTENTS, READONLY, DEBUGGING
 ```
 shows that the stab has ELF information during the boot. 

 In kernel:
 ```
2 .stab         00003bc5  f0102238  00102238  00003238  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  3 .stabstr      000016f7  f0105dfd  00105dfd  00006dfd  2**0
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
 ```

 We can observe kernel program information by running `objdump -G obj/kern/kernel | head -n 15` (first 15 lines):
 ```
 obj/kern/kernel:     file format elf32-i386

Contents of .stab section:

Symnum n_type n_othr n_desc n_value  n_strx String

-1     HdrSym 0      1418   000017c8 1     
0      SO     0      0      f0100000 1      {standard input}
1      SOL    0      0      f010000c 18     kern/entry.S
2      SLINE  0      44     f010000c 0      
3      SLINE  0      57     f0100015 0      
4      SLINE  0      58     f010001a 0      
5      SLINE  0      60     f010001d 0      
6      SLINE  0      61     f0100020 0
 ```
  The stabs layout is described above. More definitions are provided in `inc/elf.h`.
  - `n_desc` may vary based on the types, but it means description of that type.
  - `n_value` normally is address of where that is at, but may vary as well.
  - `n_strx` is the offset of string description, like function name or source filename.

### OS initialization process //TODO:

- BIOS Phase
    - add later

- Bootloader Phase
    - boot/boot.S + boot/main.c
    - Switch processor mode
    - Read 8 disk sectors to get ELF header data
    - Load segments from disk to mem, which contains actual kernel code + data
    - Jump to kernel entry, 0x10000C





## Side Note
- `.out` file: linked executable
- gdb: use `symbol-file` to let gdb know the function lines, calls, etc
- gdb: use `x/10x addr` to examine 10 words from adder
- `CR0`: control register 0; `CR0_PG`: paging enable; `CR0_PE`: enable protected mode; `CR0_WP`: write protect, cannot write to RO pages.

### Lab1 Exercise Answer

#### EX6
They are different. I think may because at bootloader, the kernel isn't being loaded to LMA, because the 8 sectors needs to be loaded in the for loop in `boot/main.c`

#### EX7
After paging enabled, 0xF0100000 also has information at LMA. I think the problem will be at relocated tag, at line 74 in `kern/entry.S`, because it will relocate execution to VMA