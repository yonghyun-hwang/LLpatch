LLpatch: LLVM-based Kernel Livepatch Generation
===============================================

LLpatch generates, from a source patch, a kernel loadable module or binary
package, that can update Linux kernel without rebooting a machine. It uses
[LLVM](https://llvm.org/) IR (Intermediate Representation) to generate
Linux kernel livepatch. Livepatch generation by LLpatch is not affected by
machine architecture, types of linkers, and kernel versions. In addition,
it does not require a whole kernel build with special compiler options,
which makes livepatch generation much faster while keeping original kernel
repository unchanged.

Supported Architectures
-----------------------

The livepatch generation by LLpatch is not affected by machine
architecture. As long as the '[clang](https://clang.llvm.org/)' is
available for the architecture and Linux kernel can be compiled with the
'clang', LLpatch should work. For now, LLpatch is well tested on the
following architectures.

- x86-64
- arm64

How to Build LLpatch
--------------------

LLpatch is implemented in C++ and
[bash](https://www.gnu.org/software/bash/) scripts. The followings are
prerequisites to build and run LLpatch.

#### Prerequisites for C++ 

1. clang++

The 'Makefile' for LLpatch uses 'clang++' to compile the source codes. The
source codes for LLpatch uses c++17 standard. Hence, please install clang
11 or later to build LLpatch.

- Ubuntu: clang-11
- ...

2. llvm-11

LLpatch generates and parses LLVM IR for the livepatch generation. To this
end, LLpatch requires LLVM library. Current LLpatch is implemented with
llvm-11 library. This is a package name for llvm-11.

- Ubuntu: llvm-11-dev
- ...

3. ELF library

At the final step for livepatch generation, LLpatch creates special ELF
sections in the kernel loadable module, kernel livepatch. The sections are
used by Linux kernel to apply the livepatch properly. To parse and manipulate
ELF sections, ELF library is used.

- Ubuntu: libelf-dev
- ...

#### Prerequisites for `llpatch`

The `llpatch` uses couple of commands to orchestrate the generation process for
kernel livepatch. The following packages are required.

- Ubuntu: binutils, binutils-aarch64-linux-gnu, diffstat, gawk, git, grep, kmod, make, patch, sed
- ...

#### Build LLpatch

LLpatch provides a simple 'Makefile' to build. Please open the 'Makefile'
to check the path to the LLVM and ELF libraries. If the paths are correct
in the 'Makefile', please issue the following command.

```bash
# print out help message on how we want to build
$ make

# clean up
$ make distclean

# build LLpatch
$ make CONFIG=optimize all -j `nproc`
```

Recommendations/Assumptions on .patch File
------------------------------------------

LLpatch makes few recommendations/assumptions on .patch file, input file,
for livepatch generation. 

1. Changes in header files are highly discouraged.
   - This is intentional not to allow big livepatch. It has been observed
     that large livepatch is hard to be applied under heavy workload.
    
2. No changes made in kbuild, makefiles for Linux kernel.

3. Changes in either a single kernel module or vmlinux
   - This is intentional design choice to simplify livepatch generation logic in
     LLpatch.
   - If it's required to generate livepatch that patches several kernel modules 
     and/or vmlinux, LLpatch provides '--multi' option to do that.
   - Again, large kernel livepatch is highly discouraged.

4. No fuzzy matching for applying .patch file
   - To reduce the complexity in distilling changed code by .patch file, no
     fuzzy matching is allowed for .patch file.
   - If 'llpatch' denies .patch file, use 'update-patch' command to avoid
     fuzzy matching.

Preparation for Kernel Livepatch Generation
-------------------------------------------

LLpatch assumes that Linux kernel is already built. If a fresh kernel repository is just
created, please build Linux kernel first.

Once Linux kernel is built, next step is to ensure that .patch file, input to LLpatch, can
be applied properly onto the kernel repository. For this, please run the following command
for dry-run.

```bash
# assuming that you are under root of kernel repository.
$ patch -N -p1 -d `pwd` --dry-run -i "${PATCH_FILE}"
```

If dry-run sees fuzzy-matching, use the following command to make the patch well-aligned
with the kernel repository.

```bash
$ update-patch --help
Usage: update-patch [IN_FILE] [OUT_FILE]
Update .patch file, [IN_FILE] to [OUT_FILE]

File: patch file

$ update-patch "${PATCH_FILE}" "${UPDATED_PATCH_FILE}"
```

How to Generate Kernel Livepatch by LLpatch
-------------------------------------------

LLpatch requires two inputs to generate kernel livepatch. The first input
is a path to the root of "pre-built" kernel repository. The second input is
a .patch file. With the given inputs, livepatch generation is very simple. Just
one command, 'llpatch'. The following shows list of command-line options
to create a kernel livepatch.

```bash
# prints out help message
$ ${path_to_llpatch}/llpatch --help
Usage: llpatch [OPTIONS] [FILE]
Build a package for kernel livepatch.

File: .patch file to use for the livepatch package build.
      The patch is applied temporarily to kernel.

Options:
  --arch       CPU architecture for livepatch. arm64 and x86_64 are supported.
               Default is x86_64.
  -h, --help   This help message.
  -k, --kdir   Path to kernel repository. If not specified, $CWD is used.
  -o, --odir   Path to output directory. If not specified, '/pkgs' is used.

Developer Options:      (for internal testing, not production use)
  --multi               Allow changes in multiple kmods and/or vmlinux
                        This is highly discouraged option to avoid "large" livepatch
  --skip-pkg-build      Produce livepatch-<diffname>.ko rather than livepatch package
  --slow-path           use kbuild to build llvm ir files
  --debug-dir           Dir for debugging with all intermediate files for klp generation


# generates kernel livepatch for x86_64 platform and creates binary package, 
# tarball, under ${kdir}/pkgs. ${kdir} is a path to a root of "pre-built" 
# kernel repository. If -k option is not given, CWD (Current Working Directory)
# is assumed to be ${kdir}
$ ${path_to_llpatch}/llpatch --kdir=${kdir} ${patch_file}


# generates kernel livepatch for arm64 platform and creates binary package, 
# tarball, under ${klp_dir}
$ ${path_to_llpatch}/llpatch --arch=arm64 --odir=${klp_dir} ${patch_file}
```
#### Semi-Automatic Livepatch Generation (Advanced)

Upstream kernel livepatch provides a way to implement custom callback
functions, (post|pre)-(patch|unpatch). The callbacks can be called before/after
the livepatch gets applied/reverted. If some actions should take place
before/after patching/unpatching, the callback can be implemented and
passed to Linux kernel. Please see the following steps for the callback
implementation.

```bash
# create livepatch wrapper, livepatch.c, under ${OUT_DIR}
$ llpatch --skip-pkg-build --odir=${OUT_DIR} ${PATCH_FILE}

# once the wrapper is generated, edit ${OUT_DIR}/(${KMOD}|vmlinux)/livepatch.c
# to implement the callbacks.

# with the callbacks implemented in livepatch.c, next step is to build livepatch
$ cd ${OUT_DIR}/(${KMOD}|vmlinux)
$ make CLANG=1 LLVM=1

# create RELA sections for kernel livepatch.
$ livepatch fixup --rela ${OUT_DIR}/(${KMOD}|vmlinux)/${LIVEPACH}.ko

# create package for the livepatch.
# KERNEL_RELEASE="$(strings "init/version.o" | awk '/^Linux version/ { print $3 }')"
$ create-package ${OUT_DIR}/(${KMOD}|vmlinux)/${LIVEPATCH}.ko \
    --buildinfo=${OUT_DIR}/(${KMOD}|vmlinux)/buildinfo \
    --patch=${OUT_DIR}/(${KMOD}|vmlinux)/klp_test_meminfo.patch  \
    --output=kernelpatch-${PATCH_FILE_NAME}-${KERNEL_RELEASE}.${CPU_ARCH}.msvp.tar.xz
```

#### Semi-Automatic Livepatch Generation for Multiple Kernel Modules (Advanced)

'llpatch' provides a way to manually generate kernel livepatch patching vmlinux and/or
several kernel modules. For the livepatch generation, two steps are required.

Step1: generate kernel livepatches for the .patch files with '--skip-pkg-build' option.

Step2: merge the generated kernel livepatches.

The following shows list of commands and their options for the livepatch
generation.

```bash

# Step1: generate kernel livepatches. Assuming that each patches modifies a single
#        ${KMOD} or vmlinux.
$ ${path_to_llpatch}/llpatch --skip-pkg-build ${patch_file_1} -o ${livepatch_1}
$ ${path_to_llpatch}/llpatch --skip-pkg-build ${patch_file_2} -o ${livepatch_2}
...
$ ${path_to_llpatch}/llpatch --skip-pkg-build ${patch_file_n} -o ${livepatch_n}

# Step2: merge kernel livepatches and put package binary under ${livepatch}
$ ${path_to_llpatch}/llpatch-merge -n ${name_of_livepatch} -o ${livepatch} \
      ${livepatch_1}/${KMOD1} ${livepatch_2}/${KMOD2} ... ${livepatch_n}/${KMODn}
```

#### Debug Kernel Livepatch Generation (Advanced)

LLpatch can generate lots of useful files to help debugging the livepatch
generation. To this end, LLpatch implements --debug-dir option.

```bash
$ llpatch --debug-dir ${DEBUG_DIR} ${PATCH_FILE}
```

With the --debug-dir option, LLpatch generates couple of files and
directories to help understanding on livepatch generation. See the following.

```bash
${DEBUG_DIR}/
├── llpatch.cmds
├── obj_patched_map.txt
├── ${PATH_TO_PATCHED_SRCS}
│   ├──  ${PATCHED_C_FILE}.c__klp_diff.ll
│   ├──  ${PATCHED_C_FILE}__original.c
│   ├──  ${PATCHED_C_FILE}__original.c__aligned
│   ├──  ${PATCHED_C_FILE}__original.ll
│   ├──  ${PATCHED_C_FILE}__patched.c
│   ├──  ${PATCHED_C_FILE}__patched.c__aligned
│   └──  ${PATCHED_C_FILE}__patched.ll
└── ${KMOD}|vmlinux
    ├──  buildinfo
    ├──  klp_patch.o
    ├──  klp_patch.o.bak
    ├──  ${PATCH_FILE}
    ├──  livepatch.c
    ├──  livepatch.lds
    └──  Makefile
```

'llpatch.cmds' contains list of commands that 'llpatch' ran for livepatch
generation. 'obj_patched_map.txt' shows what kernel modules changes which
.c files. When it comes to ${FILE}__original.c, it is a original kernel
code while ${FILE}__patched.c is a file w/ .patch file
applied. ${FILE}*__aligned is an output file of 'livepatch align' command
for ${FILE}.c. ${FILE}.c__klp_diff.ll contains LLVM codes that distills
difference between original and patched code.

Notes
-----

Please be advised that kernel livepatch could not handle all corner cases automatically
when patching linux kernel. (e.g., livepatched function A and B have dependencies between
them) So, please test your kernel livepatch before deploying it to real production
environment. Please be warned for possible kernel crash.