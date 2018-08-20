# Toolchain Setup

## Compiler (GCC)

* `-march`
* `-mtune`
* `-mfloat-abi=(hard|soft)` for ARM
* `-mfpu` for ARM FPU
* `-sysroot`

* `-nostartfiles` disables inclusion of start-up code to include the `_start` symbol.
* `-nodefaultlibs` disables linking against default libraries (e.g., libc).
* `-nostdlib` imples `-nostartfiles` and `-nodefaultlibs` and further disables implicit linking against libgcc. To link against libgcc, `-lgcc` must be passed to the compiler.

libgcc provides low-level support functions when the processor does not support an operation natively and when the code is too expensive to be inlined. The different classes of routines provided includes:
* Complex integer arithemtic
* Soft floating point, decimal, and fixed-point fractional computations
* Exception handling routines (i.e., `_Unwind_*` and `__[de]register_frame_*`)
* Miscellaneous routines (e.g., cache control intrinsics and stack manipulation intrinsics)
libgcc also provides `__main`, which is needed to interface C and C++ code and is invoked prior to `main`. For example, `__main` calls the C++ static initalizers.

The following commands are useful for determining compiler information:
* `gcc -dumpversion` prints the version number of GCC
* `gcc -dumpmachine` prints the toolchain target of the GCC build
* `gcc -print-search-dirs` prints the search path for the "installation", the programs, and the libraries
* `gcc -print-libgcc-file-name` prints the path of libgcc for the default target
* `gcc -print-multi-directory` prints the root directory containing libgcc for the default target
* `gcc -print-multi-lib` prints the mapping between targets and their library search directories
* `gcc -print-multi-os-directory`
* `gcc -print-sysroot`
* `gcc -print-sysroot-headers-suffix`

## Assembler

The GNU assembler does not have many generic options. Most options are target-specific including the `-march` and `-mabi` options common on most targets.

The following commands are useful for determining assembler information:
* `as --target-help` prints help on target-specific options

## Linker

* `-T` sets the linker script
* `--architecture` sets the architecture
* `--format` sets the target
* `--sysroot=`
* `-m` sets the emulation

The following commands are useful for determining linker information:
* `ld --target-help` prints help on emulation-specific options
* `ld --print-output-format` prints the default target output format
* `ld --print-sysroot` prints the sysroot
* `ld --print-map` prints a map file

## Linker Scripts

_Note: The information in this section is taken from [https://sourceware.org/binutils/docs/ld/]._

The linker script specifies how the program will exist at run-time. Each section specified in a linker script has two memory addresses. The load memory address (LMA) specifies where the section will be loaded from (e.g., ROM). The virtual memory address (VMA) specifies where the section will exist after loading (e.g., RAM).

_Question: Will the linker will generate initialization code to ensure that memory regions are properly loaded prior to execution?_

* `ENTRY(symbol)` determines the entry point of the program.
* `SEARCH_DIR(path)` adds a path to the list of search paths. This usually does not need to be augmented.
* `STARTUP(filename)` sets the file to be linked as the start of the output file. This usually does not need to be specified.
* `TARGET(bfdname)` sets how the linker interprets subsequent input files.
* `OUTPUT_FORMAT(bfdname)` sets the target output format.
* `OUTPUT_ARCH(bfdarch)` sets the target architecture.
* `PROVIDE(symbol)` provides a symbol reference if the symbol is not already defined.
* `MEMORY` defines the memory regions and allocations for the resulting binary.
* `REGION_ALIAS(alias, region)` creates aliases for named memory regions. This is a convenience option for defining memory regions by purpose instead of memory medium.
* `SECTIONS` defines how the input sections map to the output sections.
* `KEEP` defines what sections to keep when not referenced by code.

The linker script also supports asserts to sanity check the resulting binary.
* `ASSERT(expression, message)`

Additional functionss may be found at: [https://sourceware.org/binutils/docs/ld/Builtin-Functions.html]. The following functions are useful:
* `ADDR(section)` returns the VMA of the given section.
* `ALIGN(align)` returns the parameter aligned to the next boundary.
* `ALIGNOF(section)` returns the alignment of section in bytes.
* `DEFINED(symbol)` determines whether the symbol is defined.
* `LENGTH(memory)` returns the length of the memory region.
* `LOADADDR(section)` returns the LMA of the given section.
* `LOG2CEIL(exp)` returns the binary log (rounded up).
* `MIN` and `MAX`.
* `ORIGIN(memory)` returns the base address of the memory region.
* `SIZEOF(section)` returns the size of the section in bytes if allocated and an error otherwise.

The linker script supports the setting of variables. The reserved `.` variable is the location counter. It is useful for performing relative address calculations.

Depending on the target output format (e.g., `srec` or `binary`), some information specified in the linker script may be lost. An example of this the entry point of a program.

Linker scripts may be nested to allow for specialization of generic linker scripts.
* `INCLUDE filename`

The following linker script directives only apply to ELF files. 
* `HIDDEN(symbol)` will prevent a symbol from being exported.
* `PROVIDE_HIDDEN(symbol)` provides a symbol reference that is not already defined and will not be exported.
* `VERSION` provides information for versioned symbols.
* `PHDRS` provides information on program header segments. The `SECTIONS` definition in the linker script can place sections in a segment via the `:segment` reference.

The linker can only define addresses, not values. The linker can provide symbols, which represent addresses and can be externed and referenced by program source code. For example, consider the following declaration:

```
start_of_ROM = .ROM;
end_of_ROM = .ROM + sizeof(.ROM);
start_of_FLASH = .FLASH;
```

Then the referring source code to copy one to the other would be:

```
extern char start_of_ROM, end_of_ROM, start_of_FLASH;
memcpy(&start_of_FLASH, &start_of_ROM, &end_of_ROM - &start_of_ROM);
```

or alternatively:

```
extern char start_of_ROM[], end_of_ROM[], start_of_FLASH[];
memcpy(start_of_FLASH, start_of_ROM, end_of_ROM - start_of_ROM);
```

### MEMORY and SECTIONS

The `MEMORY` command defines names, protection attributes, allocation attributes, and address space.

```
MEMORY
{
    name [(attr)] : ORIGIN = address, LENGTH = size
}
REGION_ALIAS(alias, name)
```

The `SECTIONS` command defines names, VMAs, LMAs, alignment, memory regions, and program header information (for ELF).

```
SECTIONS
{
    /* Sections not requiring loading due to already being in the desired location. */
    ROM 0 (NOLOAD) : { ... }
    /* Section consolidation */
    name :
    {
        *(name)
    } > VMA_REGION_NAME AT> LMA_REGION_NAME_IF_DIFFERENT_FROM_VMA_REGION_NAME :phdr [= FILL VALUE AS HEX]
}
```

There is a special `CONSTRUCTORS` command and `__CTOR_LIST__`, `__CTOR_END__`, `__DTOR_LIST__`, and `__DTOR_END__` symbols when dealing with C++.

The `/DISCARD/` section is reserved for excluding input sections from the resulting output binary.

