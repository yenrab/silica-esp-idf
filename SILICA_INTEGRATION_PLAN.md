# Silica Integration Plan for this ESP-IDF Fork

Status: **Phase 0 done** (placeholder compiler checked in). Everything else is planned, not implemented.

This document is the plan for turning this fork of ESP-IDF into a development tool that builds
firmware written in the Silica language, alongside C, C++ and assembly. The Silica compiler is not
ready for embedded targets yet. The goal is to have every ESP-IDF-side piece built, tested and
documented against a placeholder compiler, so that when the real compiler is ready it drops in
through the normal tool-download mechanism with no build-system changes.

Ground rules:

* This repository never links to, vendors, submodules or hard-codes a path into the Silica
  compiler's development tree. The only coupling is the interface contract in section 3 and,
  later, a `tools/tools.json` entry that downloads a released compiler archive.
* Until then, `tools/silica/bin/silica-compiler` is a placeholder that mimics the compiler's
  file-level contract and generates no code.
* Every phase must leave `idf.py build` working unchanged for projects that contain no Silica.

Execution model:

* The Silica compiler is a **host tool that runs on macOS** (Apple Silicon first) and
  **cross-emits assembly for the Espressif chip** selected by `IDF_TARGET`. It is installed
  into `IDF_TOOLS_PATH` like gcc and clang and is never run on the device.
* The compiler's job ends at assembly text. Assembling, archiving, linking, `ldgen`, flash
  image generation and flashing are all done by this fork with the existing Espressif
  toolchains, exactly as for C sources.
* Everything downstream of the emitted assembly lives in this repository: the build-system
  glue, the runtime component that hosts Silica programs on FreeRTOS, the templates, examples,
  tests and documentation. Only the emitter backends and the runtime's generated code live in
  the compiler (requirements W1 and W2 in section 3).
* Other host operating systems come later, when the compiler is ported. Until then a Silica
  project configured on Linux or Windows must fail with a clear message, not a mysterious one.

---

## 1. Where things stand today

### 1.1 This fork (ESP-IDF)

Nothing Silica-specific existed before this plan. The relevant facts about the build system:

* **Two parallel build systems must be kept in step.** The classic one lives in `tools/cmake/`
  and the newer one in `tools/cmakev2/` (activated by the `IDF_BUILD_V2` environment variable;
  see `tools/cmake/project.cmake:21-99`). Both hard-code the language list `C CXX ASM`
  (`tools/cmake/project.cmake:753` and `:46`) and both branch on `ENABLED_LANGUAGES` when picking
  a linker language (`tools/cmakev2/component.cmake:91-103`, `tools/cmakev2/build.cmake:764-800`).
* `idf_component_register(SRCS ...)` does **not** validate extensions
  (`tools/cmake/component.cmake:273-315`). An unknown extension is passed to `add_library()` and
  silently ignored. The `SRC_DIRS` glob only picks up `*.c *.cpp *.S`
  (`tools/cmake/component.cmake:292`, `tools/cmakev2/compat.cmake:118`).
* Per-language flags are build properties: `C_COMPILE_OPTIONS`, `CXX_COMPILE_OPTIONS`,
  `ASM_COMPILE_OPTIONS`, applied through `add_asm_compile_options()` and friends in
  `tools/cmake/utilities.cmake:415-443` and consumed in `tools/cmake/component.cmake:476-485`.
  The v2 equivalent is `tools/cmakev2/utilities.cmake:657-674`.
* Toolchain selection is by file name: `tools/cmake/targets.cmake:126-171` picks
  `toolchain-[clang-]<target>.cmake`, and `tools/cmake/toolchain.cmake:33-56` sets
  `CMAKE_C_COMPILER` etc. based on whether the file name contains `clang`. Cmakev2 duplicates this
  in `tools/cmakev2/idf.cmake:311-370` and also honours `IDF_CUSTOM_TOOLCHAIN`.
* Every component library is forced to `LINKER_LANGUAGE C` (`tools/cmake/component.cmake:504`).
* Tools are installed by `tools/idf_tools.py` from `tools/tools.json`, validated against
  `tools/tools_schema.json`. The downloader accepts `file://` URLs
  (`tools/idf_tools.py:588-593`). Tool version checks run through
  `tools/cmake/tool_version_check.cmake` and require a `version_cmd` unless the entry sets
  `is_executable: false` or uses `tool_info_file`.
* Firmware entry: FreeRTOS's `main_task` calls a C symbol `void app_main(void)`
  (`components/freertos/app_startup.c:210-213`). No linker-script change is needed for a
  new language, only that symbol.
* CI gates that a new file trips: `tools/ci/executable-list.txt` (any executable under the
  repo), `tools/ci/exclude_check_tools_files.txt` or a `.patterns-*` list in
  `.gitlab/ci/rules.yml` (any file under `tools/`), `tools/ci/check_copyright_config.yaml`
  (license headers, needs a section for a new source extension), `.gitlab/CODEOWNERS`,
  and the README "Supported Targets" table check for any new example
  (`tools/ci/check_build_test_rules.py`).

### 1.2 The Silica compiler (facts the plan depends on, as of September 2026)

These were read from the compiler's own tree and verified by running the binary. They are the
reason the integration cannot be a simple "add a CMake language" job.

| Aspect | Current behaviour |
|---|---|
| Invocation | **No command-line interface.** The binary takes no arguments and reads `silica.config` from the current working directory. `--version` and `--help` do not exist. |
| Input | `silica.config`: one `.silica` path per line, relative to the working directory, dependencies first, `main.silica` last. No search paths. Every module must be listed. |
| Modules | Module name is the file's basename without `.silica`. Directories do not contribute. Basenames must be unique across the whole program. Imports are `use <module>;`. |
| Output | Assembly text only: `foo/bar.silica` produces `foo/bar.sams` and `foo/bar.iface` next to the source. Possibly `__silica_runtime.sams`, `silica.link`, `silica.needs_runtime` at the build root. The compiler never produces objects and never links. |
| Memory protocol | For large unit sets it writes `silica.compile.order` and exits with code **75** between units. The caller must re-invoke it until it exits 0. |
| Backends | One emitter: `apple_silicon_mac` (AArch64, Mach-O assembler syntax). The backend is selected when the **compiler itself** is built (`make TARGET=...`), not at run time. No Xtensa or RISC-V backend exists; there is no `rv32` anywhere in the tree. |
| Runtime | `__silica_runtime.sams` is emitted by the compiler and depends on Darwin pthreads, `os_unfair_lock`, `__ulock_wait`, Mach QoS, `mmap`, `malloc`. Nothing runs without an OS today. |
| FFI | Outbound only (Silica calls C through `dangerous_*` modules and `wrapper_meta` sidecars under `dangerous_exposure_source/`; archives resolved from `dangerous_exposure_source/lib/lib<name>.a`, recorded in `silica.link`). **C cannot call Silica.** |
| Entry symbol | The program entry is emitted as the bare symbol `main`. |
| Symbols | `<module>_<function>` with C ABI; only fixed-width integer and float types at the FFI boundary. |
| Version | No version string. The version is only the number in the binary's file name (`silica-<NNNNNN>-<platform>`, lower is newer). |
| Existing build glue | `project_makefiles/` in the compiler repo: a three-stage make flow (compile all units with the exit-75 loop, assemble each `.sams` with `clang -c -x assembler`, link with clang). |

---

## 2. Design decisions

1. **Source extension:** `.silica`, as the compiler requires.
2. **Whole-set compilation per component, driven by a wrapper, not a CMake language.**
   CMake's language machinery assumes one source in, one object out, with flags on the command
   line. The Silica compiler is configuration-file driven and compiles a dependency-ordered set.
   So each component's Silica sources are compiled by one `add_custom_command` that:
   * writes `silica.config` into `<build>/esp-idf/<component>/silica/`,
   * runs `tools/silica/silica_build.py`, which copies or symlinks the units into that
     directory (preserving relative paths), runs the compiler in a loop until exit code is 0,
     and collects the generated `.sams` files,
   * emits the `.sams` files as `GENERATED` assembly sources of the component library, so the
     existing target assembler (`xtensa-esp-elf-gcc` or `riscv32-esp-elf-gcc`, or clang)
     assembles them exactly as it assembles `.S` files today.

   Consequences: no change to `project(... C CXX ASM)`, no CMake language modules, the
   `LINKER_LANGUAGE C` rule keeps working, and both build systems can share the wrapper.
   A full CMake language (`CMakeDetermineSILICACompiler.cmake` etc.) is listed as an option in
   section 8 if the compiler later grows a per-file, argument-driven mode.
3. **Compiler discovery:** `IDF_SILICA_COMPILER` cache/env variable, defaulting to
   `silica-compiler` on `PATH`, which `idf_tools.py export` puts there once the tool is
   registered in `tools/tools.json`. Target selection is passed to the compiler through the
   `SILICA_TARGET` environment variable (set to `IDF_TARGET`) until the compiler has a flag.
4. **Opt-in:** nothing Silica-related runs unless a component lists a `.silica` file or a
   `SRC_DIRS` directory contains one. There is no `IDF_TOOLCHAIN=silica`; Silica is orthogonal to
   the gcc/clang choice because gcc or clang still assembles and links.
5. **Entry point:** the runtime component (section 4, phase 4) provides
   `void app_main(void)` that calls the Silica entry symbol. This is a runtime concern and is
   part of the interface contract in section 3, not of the build system.
6. **Module-name uniqueness** is enforced per component by the wrapper today. Cross-component
   Silica dependencies are deferred (section 8); until then, a Silica program is one component.

### 2.1 The Silica makefiles and how this plan treats them

The Silica project uses GNU make in two separate places. They are handled differently, and
neither is invoked by this fork.

**Building the compiler itself.** The compiler tree is built with `make -C src`, and
`make TARGET=<backend>` chooses which emitter is compiled into the resulting binary (the
backend is a build-time choice, see section 1.2). ESP-IDF never builds its compilers from
source; gcc and clang arrive as prebuilt archives through `tools/tools.json`, and the Silica
compiler will arrive the same way. The compiler's own makefiles therefore matter only to
whoever produces the release archives (requirement W6 in section 3). They are out of scope
here.

**Building Silica applications.** The compiler repo ships `project_makefiles/`, a drop-in
application build flow with three stages:

| Stage | What the Silica makefiles do | What this plan does instead |
|---|---|---|
| 1. Compile | `topo_silica_config.sh` finds every `.silica`, sorts them by `use` lines with `main.silica` last, writes `silica.config`; `silica_compiler.mk` runs the compiler in a loop until it stops exiting 75 | `tools/silica/silica_build.py` (phase 2) does the same ordering and the same loop, per component, inside the ESP-IDF build directory |
| 2. Assemble | `%.o: %.sams` rule runs `clang -mmacosx-version-min=... -c -x assembler` under a `-j` sub-make | the generated `.sams` files become `GENERATED` assembly sources of the component library, so Ninja runs the target assembler (`xtensa-esp-elf-gcc`, `riscv32-esp-elf-gcc`, or clang) in parallel with everything else |
| 3. Link | clang links all objects plus archives listed in `silica.link` into a host executable with a Mach-O stack-size option | nothing; the objects go into the component archive and through ESP-IDF's linker scripts, `ldgen`, and the ELF-to-binary steps like any C object. Archives named in `silica.link` are mapped to component link dependencies (requirement W5) |

Why the flow is reimplemented rather than reused:

* The makefiles are host-specific. They hard-code clang, a macOS minimum-version flag, a
  Mach-O linker option, and a Homebrew clang path.
* Their end product is a linked host executable, which has no place in an ESP-IDF build.
  Only stage 1 carries over unchanged in spirit.
* Copying them in would couple this repository to the compiler tree, which the ground rules
  forbid.
* ESP-IDF's build uses CMake and Ninja on every supported host, and make is not a guaranteed
  build-time dependency, in particular on Windows.

The three ideas that do carry over, dependency ordering, the exit-75 re-invoke loop, and
collecting generated files, are kept as the contract items C1 to C3 and C7 in section 3, so a
change in the compiler's make flow shows up as a contract change rather than as a hidden
breakage.

**Make-based variant, if preferred.** The custom command in phase 2 could run a small
ESP-IDF-owned makefile (`tools/silica/silica_build.mk`) whose rules mirror the shapes in
`project_makefiles/`, so people who know the Silica flow recognise it. The cost is a GNU make
requirement at build time on every host; ESP-IDF's installer would have to add make to
`tools/tools.json` for Windows, and the wrapper would still be needed to write the per-component
`silica.config` and to hand the results back to CMake. This variant is not the default. If it is
chosen, phase 2 step 1 changes from "Python wrapper" to "Python wrapper plus makefile", and
nothing else in the plan moves.

---

## 3. Interface contract the real compiler must meet

This is the list to hand to the Silica compiler project. The placeholder implements every
"build-system" item now so the ESP-IDF side can be tested. Items marked **(compiler work)** are
prerequisites on the Silica side; the ESP-IDF plumbing cannot make them unnecessary.

Where the compiler work lives: the compiler has one backend directory,
`src_selfhost/emitter/apple_silicon_mac/` (about 33,500 lines), and every target-specific
line of the compiler is inside it, including the actor runtime, which the emitter generates
as assembly text. W1, W2 and the symbol-naming issues are therefore one job: a new
`emitter/<esp_target>/` tree, which the compiler's existing `make TARGET=` mechanism
already knows how to select. C1, C3, C4, C5 and W7 live in the driver and diagnostics code
outside the emitter, and W5 in the frontend FFI checker. Section 5 splits this into work
packages.

Build-system contract (mimicked by the placeholder):

* C1. Reads `silica.config` from the working directory; one `.silica` path per line.
* C2. Writes `<stem>.sams` and `<stem>.iface` next to each unit; assembly must be in GNU
  assembler syntax accepted by `xtensa-esp-elf-as` and `riscv32-esp-elf-as` (ELF directives,
  `#`/`/* */` comments, no Mach-O `_` prefix, no `.subsections_via_symbols`).
* C3. Exit 0 on success; exit 75 with `silica.compile.order` present means "re-invoke me".
  Any other non-zero exit is a hard error with diagnostics on stderr.
* C4. `--version` prints one line containing a parseable version, and `--help` exists.
  (Required by `tools/tools.json` `version_cmd` / `version_regex`; the alternative is
  `is_executable: false` or `tool_info_file`, which the plan uses only for the placeholder.)
* C5. Target selection: honours `SILICA_TARGET=<esp32|esp32s3|esp32c3|esp32c6|...>` (or a
  `--target` flag once one exists) and fails loudly when the target is unsupported.
  Today the backend is chosen when the compiler binary is built, so the first cross-emitting
  release may ship as one binary per chip family instead (for example
  `silica-compiler-xtensa` and `silica-compiler-riscv32`). The wrapper supports both shapes:
  it looks for `silica-compiler-<IDF_TARGET>`, then `silica-compiler-<xtensa|riscv32>`, then
  `silica-compiler` with `SILICA_TARGET` set. Note the compiler's current file naming
  `silica-<N>-<platform>` describes the host only; a cross-emitting binary needs a name that
  carries both host and target.
* C6. Deterministic output: the same sources produce byte-identical `.sams` so Ninja restat
  and ccache-style caching work.
* C7. Emitted runtime assembly (`__silica_runtime.sams`) must be written into the working
  directory, once, and must be assemblable like any other unit.

Compiler work required before firmware can be produced (not mimicked):

* W1. An **Xtensa (esp32, esp32s2, esp32s3)** and a **RISC-V rv32imc/rv32imac (esp32c*/h*/p4)**
  emitter backend, producing ELF assembly with the ESP ABI (ilp32 for RISC-V; windowed ABI
  for Xtensa as used by ESP-IDF).
* W2. A **freestanding runtime on FreeRTOS**: actors, regions, message queues implemented on
  `xTaskCreate`, FreeRTOS queues/semaphores and the ESP-IDF heap (`heap_caps_*`), with the
  Darwin `ulock`/`os_unfair_lock`/QoS dependencies removed. Regions with a `device` space must
  map to DMA-capable or PSRAM heap capabilities.
* W3. **Inbound entry**: the program entry must be reachable as a C-callable, no-argument
  function. The compiler emits it as the bare symbol `main` today, which is acceptable because
  no firmware component defines `main` (the ELF entry is `call_start_cpu0`); the rename to
  `app_main` is done on the ESP-IDF side (phase 4, "Entry point"). An entry-symbol override in
  the compiler is welcome but not required.
* W4. **Inbound FFI** for ISR handlers and FreeRTOS callbacks, or a documented pattern that
  keeps all interrupt work in C and hands events to Silica actors through a queue.
* W5. FFI sidecars must be able to name ESP-IDF component libraries (for example
  `link_library: "esp_timer"`) without a `dangerous_exposure_source/lib/*.a` copy, or the
  wrapper must synthesize that directory from the component's link line.
* W6. A **macos-arm64** compiler build that cross-emits for the Espressif targets, published
  as a release archive with a stable URL so `idf_tools.py` can install it. Other host
  platforms that `tools/tools.json` distinguishes (macos-x86_64, linux-amd64, linux-arm64,
  win64) follow when the compiler is ported; until then the tool entry lists only
  `macos-arm64`, and `idf_tools.py` reports the tool as unavailable on other hosts.
* W7. Error messages with file:line:column so `idf.py` hints and IDE integration work.

---

## 4. Phased plan

Each phase ends with a green `pre-commit run --all-files` and a green
`tools/test_build_system` run for both `IDF_BUILD_V2` unset and set.

### Phase 0: placeholder compiler (done in this change)

* `tools/silica/bin/silica-compiler`: Python placeholder implementing C1 to C4 and C7's file
  layout. Never generates code. `SILICA_PLACEHOLDER_EXIT75=1` forces one exit-75 round so the
  resume loop can be tested.
* `tools/silica/README.md`: what the directory is.
* CI lists: `tools/ci/executable-list.txt` and `tools/ci/exclude_check_tools_files.txt`.

### Phase 1: tool registration

Files: `tools/tools.json`, `tools/tools_schema.json` (only if a new key is needed),
`tools/silica/package_placeholder.py` (new), `docs/en/api-guides/tools/idf-tools.rst`.

1. Add a `silica-compiler` entry to `tools/tools.json`:
   `install: "on_request"`, `supported_targets` all chips, `export_paths: [["silica-compiler", "bin"]]`,
   `version_cmd: ["silica-compiler", "--version"]`, `version_regex: "silica-compiler ([0-9A-Za-z.\\-]+)"`,
   `license: "Apache-2.0"` (confirm against the compiler's LICENSE before release),
   `info_url` pointing at the Silica project page.
2. Versions: the placeholder is packaged by `tools/silica/package_placeholder.py` into
   `silica-compiler-0.0.0-placeholder-any.tar.gz` under the `any` platform key, with a
   computed `sha256` and `size`. For local development the URL is `file://` to the packaged
   archive in `IDF_TOOLS_PATH/dist`; for CI, host the archive on the fork's GitHub release
   page. When the real compiler ships (W6), add a `macos-arm64` version with status
   `recommended` and mark the placeholder `deprecated`; other platform keys are added as the
   compiler is ported.
3. Install entry point. Today `install.sh:32,35` runs `idf_tools.py install --targets=...`
   and then `install-python-env --features=...`; the `--enable-<x>` flags parsed by
   `tools/install_util.py` only select Python-env features, not tools. Add a small extension:
   `install_util.py` recognises `silica` in the argument list and the four install scripts
   (`install.sh`, `install.fish`, `install.bat`, `install.ps1`) then run
   `idf_tools.py install silica-compiler` in addition. Until that lands, the documented
   command is `python tools/idf_tools.py install silica-compiler`.
4. Verify: `idf_tools.py install silica-compiler`, `idf_tools.py export` puts it on `PATH`,
   `idf_tools.py check-tool-supported --tool-name silica-compiler --exec-path ...` passes.

### Phase 2: build system, classic (`tools/cmake`)

Files: `tools/cmake/silica.cmake` (new), `tools/cmake/component.cmake`, `tools/cmake/build.cmake`,
`tools/cmake/utilities.cmake`, `tools/cmake/project.cmake`, `tools/cmake/toolchain_flags.cmake`,
`CMakeLists.txt` (root), `tools/silica/silica_build.py` (new).

1. `tools/silica/silica_build.py`: the wrapper described in section 2. Arguments:
   `--compiler`, `--target`, `--work-dir`, `--component-dir`, `--output-list <file>`,
   `--depfile <file>`, `--entry-symbol <name>` (default `app_main`, see phase 4),
   then the unit paths. It computes the dependency order from `use` lines (port of
   `topo_silica_config.sh` to Python, with `main.silica` last), writes `silica.config`, runs the
   exit-75 loop, verifies every expected `.sams` exists, and writes a depfile so Ninja re-runs
   when any unit changes. Errors: duplicate basenames, cycles, missing outputs, compiler
   diagnostics passed through.
2. `tools/cmake/silica.cmake`: `__silica_add_sources(component_lib sources_var)`.
   Splits `.silica` files out of `sources`, registers the custom command, adds the generated
   `.sams` files back with `set_source_files_properties(... PROPERTIES LANGUAGE ASM GENERATED TRUE)`.
   Called from `idf_component_register` right after `__component_add_sources`
   (`tools/cmake/component.cmake:~470`). Adds `.sams` to the ASM extension list
   (`list(APPEND CMAKE_ASM_SOURCE_FILE_EXTENSIONS sams)` before `project()` in
   `tools/cmake/project.cmake`).
3. `SRC_DIRS` glob: add `*.silica` at `tools/cmake/component.cmake:292`.
4. New build properties `SILICA_COMPILE_OPTIONS` (root `CMakeLists.txt` next to
   `ASM_COMPILE_OPTIONS` at `:384`) and `SILICA_COMPILER` (set in `idf_build_process`,
   `tools/cmake/build.cmake:~620`, from `IDF_SILICA_COMPILER` or `find_program(silica-compiler)`),
   plus `add_silica_compile_options()` in `tools/cmake/utilities.cmake` (passed to the wrapper as
   environment for now, since the compiler has no flags).
5. Response file: add `silicaflags` to `tools/cmake/toolchain_flags.cmake` so
   `idf_toolchain_add_flags(SILICA ...)` works like the other languages.
6. Version check: call `check_expected_tool_version(silica-compiler ${SILICA_COMPILER})` only
   when a component actually contains Silica sources.
7. `project_description.json` (`tools/cmake/project_description.json.in`): add
   `silica_compiler` path so IDE integrations can find it.
8. Also honour `.silica` in `idf_component_get_property(... SRCS)` consumers, `ldgen` (no
   change needed, objects are ordinary ASM objects), and `compile_commands.json` (the wrapper
   appends synthetic entries for each unit so clangd-style tooling can at least open them).

### Phase 3: build system, cmakev2 (`tools/cmakev2`)

Files: `tools/cmakev2/component.cmake`, `tools/cmakev2/compat.cmake`, `tools/cmakev2/build.cmake`,
`tools/cmakev2/utilities.cmake`, `tools/cmakev2/project.cmake`.

1. Include the same `tools/cmake/silica.cmake` (it is build-system agnostic by design) from
   `tools/cmakev2/idf.cmake`.
2. Call `__silica_add_sources` from the v2 component registration path.
3. `*.silica` in the `SRC_DIRS` glob at `tools/cmakev2/compat.cmake:118`.
4. Per-language option genex in `tools/cmakev2/utilities.cmake:657-674` for `SILICA_COMPILE_OPTIONS`
   (applies to the wrapper environment, not to `add_compile_options`).
5. Keep the `ENABLED_LANGUAGES` checks unchanged: ASM stays enabled, so a Silica-only component
   still links.

### Phase 4: runtime component and entry point

Files: `components/silica_rt/` (new: `CMakeLists.txt`, `Kconfig`, `silica_rt.c`,
`include/silica_rt.h`, `idf_component.yml` if published), `components/freertos/app_startup.c`
(no change expected; verify the "Calling app_main()" pytest marker is unaffected).

1. `silica_rt.c`: `void app_main(void)` guarded by `CONFIG_SILICA_RT_PROVIDE_APP_MAIN`
   (default `y` when the component is linked) that calls `silica_main()`. Until W3 lands, the
   placeholder produces no symbol, so the component is built but excluded from the link unless a
   project requests it (`REQUIRES silica_rt`).
2. Stubs for the runtime hooks W2 will need (`silica_rt_spawn`, `silica_rt_send`,
   `silica_rt_region_alloc` mapping to `heap_caps_malloc`), each returning `ESP_ERR_NOT_SUPPORTED`
   and logging once. These define the C-side ABI the runtime port targets; keep them in
   `include/silica_rt.h` with Doxygen comments.
3. Kconfig menu "Silica runtime": provide-app-main, actor stack size, region heap caps.
4. `tools/ci/check_copyright_config.yaml`: add a section for `.silica` files with a
   `new_notice_silica` template (`//` comments per the language) and list `.sams` under
   ignored generated extensions.

**Entry point: how `main.silica` becomes `app_main`.**

The compiler turns any function named `main` into the bare symbol `main`, injects two
runtime-registry init calls at the top of it, and otherwise emits it as an ordinary function:
prologue, body, return value in the first argument register, return. It does not call exit.
Every other function is module-prefixed, so a Silica `fn app_main()` would become
`main_app_main` and would not be treated as the entry; the rename has to happen after the
compiler. Three ways, in order of preference:

1. **Rename in the wrapper (default).** `silica_build.py` already treats `main.silica`
   specially. After assembling it, run `<prefix>objcopy --redefine-sym main=app_main` on the
   object (both Espressif toolchains ship objcopy; clang builds use `llvm-objcopy`), or rewrite
   the `.globl main` / `main:` lines in `main.sams` before assembly. Controlled by the
   build property `SILICA_ENTRY_SYMBOL`, default `app_main`. Works with the placeholder now and
   with the real compiler unchanged.
2. **C shim in `silica_rt`.** `void app_main(void)` calls the Silica `main`. This is what step 1
   above describes. Its extra value is that the shim can run the Silica entry in a dedicated
   FreeRTOS task with its own stack size instead of on the main task. Use it together with
   option 1 by setting `SILICA_ENTRY_SYMBOL=silica_main` so the shim calls `silica_main`.
3. **Compiler flag.** An entry-symbol override in the compiler. Two places in the emitter's
   symbol naming and one in the entry-injection check. Optional (W3).

Changes that are needed whichever option is chosen:

* **Return value.** Silica's `main` returns an atom or an int64 meant as a process exit code.
  `app_main` returns void and the FreeRTOS main task deletes itself when it returns. The shim
  logs the value at info level and ignores it.
* **Stack.** The macOS makefiles link with a 256 MB stack. The ESP-IDF main task stack is
  `CONFIG_ESP_MAIN_TASK_STACK_SIZE`, default 3584 bytes. `silica_rt` adds
  `CONFIG_SILICA_RT_ENTRY_TASK_STACK_SIZE` and runs the entry in its own task (option 2), so
  the main task size stays untouched.
* **Lifetime after return.** On macOS, returning from `main` ends the process and every actor.
  On ESP-IDF, returning from `app_main` leaves other tasks running. The runtime port must
  define one behaviour; keeping actors alive after the entry returns is the firmware-natural
  choice and the plan assumes it.
* **Registry init symbols.** The injected `_silica_pid_registry_init` calls carry the Mach-O
  underscore. The ELF backend emits them without it, and `silica_rt` provides them (W1, W2).
* **Section placement.** The symbol lands in `.text` of the component archive; `ldgen`
  places it in flash like any C function. No linker-script change.
* **Test marker.** The "Calling app_main()" log line in `components/freertos/app_startup.c`
  is untouched, so existing pytest expectations hold.

### Phase 5: `idf.py`, templates and Kconfig

Files: `tools/idf_py_actions/create_ext.py`, `tools/templates/sample_project_silica/` (new),
`tools/idf_py_actions/hints.yml`, `tools/idf_py_actions/core_ext.py`.

1. `tools/templates/sample_project_silica/`: `CMakeLists.txt`, `main/CMakeLists.txt`
   (`idf_component_register(SRCS "main.silica" REQUIRES silica_rt)`), `main/main.silica`
   (the `fn main() -> atom { :ok }` hello world). Mirrors `sample_project_cpp`.
2. `idf.py create-project --silica`, mirroring the existing `--cpp` flag in
   `tools/idf_py_actions/create_ext.py:74-76,162` (`use_cpp` selects `sample_project_cpp`; add
   `use_silica` selecting `sample_project_silica`, with `--cpp` and `--silica` mutually exclusive).
3. `hints.yml`: entries for "silica.config not found", "exit code 75 loop exceeded", "duplicate
   module basename", "silica-compiler not found on PATH (run install.sh --enable-silica)".
4. No change to `SUPPORTED_TARGETS`/`PREVIEW_TARGETS`; Silica is not a target.

### Phase 6: example project

Files: `examples/get-started/hello_world_silica/` (new): `CMakeLists.txt`, `main/CMakeLists.txt`,
`main/main.silica`, `main/helpers.silica`, `README.md` with the "Supported Targets" table,
`sdkconfig.ci`, `pytest_hello_world_silica.py`.

Because the placeholder emits no code, the example's C fallback in `silica_rt` prints
"Silica placeholder: no program compiled" and the pytest expects that line. When W1 to W3 land,
the pytest expectation changes to the real program's output. The example must build for every
target in the table with both gcc and clang (`IDF_TOOLCHAIN=clang`).

### Phase 7: tests

Files: `tools/test_build_system/test_silica.py` (new), `tools/test_build_system/build_test_app/`
(add a `silica_component`), `tools/test_idf_tools/test_idf_tools.py` (tool entry test),
`tools/silica/test/test_silica_build.py` (unit tests for the wrapper: topological order,
`main.silica` last, duplicate basenames, cycle detection, exit-75 loop using
`SILICA_PLACEHOLDER_EXIT75=1`, depfile contents, missing output detection).

Build-system tests, each run with and without `IDF_BUILD_V2`:

* A component with `.silica` in `SRCS` produces `.sams` files under
  `build/esp-idf/<component>/silica/`, the objects appear in the component archive, and the
  compiler is invoked exactly once per component per clean build.
* `SRC_DIRS` picks up `.silica`.
* Touching one unit re-runs the compiler (depfile), touching an unrelated `.c` does not.
* A project with no Silica never invokes the compiler and never requires it on `PATH`.
* `IDF_SILICA_COMPILER` override is honoured; a missing compiler gives the hint text.
* `idf_toolchain_add_flags(SILICA ...)` writes `silicaflags` (classic only).
* CI: add `tools/silica/**/*` to `.patterns-build_system` in `.gitlab/ci/rules.yml` and remove
  the temporary `exclude_check_tools_files.txt` line; add `/tools/silica/` and
  `/components/silica_rt/` to `.gitlab/CODEOWNERS`.

### Phase 8: documentation

Files: `docs/en/api-guides/build-system.rst` (SRCS extension list at `:392` and `:1625`, a new
"Silica sources" section next to the C/C++ ones), `docs/en/api-guides/silica.rst` (new: how to
enable, project layout, module rules, FFI to C components, limitations), `docs/en/api-guides/tools/idf-tools.rst`
(tool entry), `docs/en/api-reference/system/silica_rt.rst` (new, from the header),
`docs/docs_not_updated/` entries for chips that lack a backend, and the root `README.md`
Quick Reference.

### Phase 9: swapping in the real compiler

Checklist, to be executed only when W1 to W7 are satisfied:

1. Add real per-platform versions to the `silica-compiler` entry in `tools/tools.json`
   (status `recommended`), mark the placeholder version `deprecated`, keep the placeholder
   binary in `tools/silica/bin/` for tests.
2. Confirm C1 to C7 against the shipped binary with `tools/silica/test/test_contract.py`
   (a contract test that runs whatever `silica-compiler` is on `PATH` against a fixture
   project and checks the file-level behaviour). Written in phase 7, skipped when only the
   placeholder is present.
3. Wire `silica_rt` to the real runtime (W2), enable `CONFIG_SILICA_RT_PROVIDE_APP_MAIN`.
4. Update `hello_world_silica` expectations, remove the "placeholder" fallback path.
5. Announce the supported chip list in `docs/en/api-guides/silica.rst` and the example README.

---

## 5. Work split and parallel plan

There are two tracks. **Track A** is everything in this fork. **Track B** is everything in
the Silica compiler repository. They meet only at the contract in section 3 and at the
acceptance test A8 below. Neither track waits for the other to start; Track A is completed
against the placeholder, Track B against the contract.

### 5.1 Track A: work outside the compiler (this fork)

Each package names the files it owns. Two agents never edit the same file; when a package
needs a hook in a file owned by another package, it asks that package's owner for the hook.

| ID | Package | Owns | Depends on | Phase |
|---|---|---|---|---|
| A1 | Tool registration: `tools.json` entry, placeholder packaging script, install-script hook, `idf_tools.py` export check | `tools/tools.json`, `tools/tools_schema.json`, `tools/silica/package_placeholder.py`, `install.*`, `tools/install_util.py` | none | 1 |
| A2 | Build wrapper: dependency ordering, `silica.config` writer, exit-75 loop, generated-file collection, depfile, entry-symbol rename, compiler lookup order, non-macOS hint; plus its unit tests | `tools/silica/silica_build.py`, `tools/silica/test/test_silica_build.py` | none (interface frozen in 5.3) | 2, 7 |
| A3 | Classic CMake glue: `silica.cmake`, hooks in `component.cmake`, `build.cmake`, `utilities.cmake`, `project.cmake`, `toolchain_flags.cmake`, root `CMakeLists.txt`, `project_description.json.in`, version check | `tools/cmake/silica.cmake` and the listed hook points in `tools/cmake/*` and root `CMakeLists.txt` | A2 interface | 2 |
| A4 | cmakev2 glue | `tools/cmakev2/*` hook points | A3 skeleton (the `__silica_add_sources` signature) | 3 |
| A5 | Runtime component: `app_main` shim, entry task with its own stack, Kconfig, C runtime API header with stubs, `.silica` copyright section | `components/silica_rt/**`, `tools/ci/check_copyright_config.yaml` | none | 4 |
| A6 | Templates, `idf.py create-project --silica`, hints | `tools/templates/sample_project_silica/**`, `tools/idf_py_actions/create_ext.py`, `tools/idf_py_actions/hints.yml` | none to write; A3 to verify | 5 |
| A7 | Example project with README target table and pytest | `examples/get-started/hello_world_silica/**` | A3, A5 | 6 |
| A8 | Contract and acceptance tests: `test_contract.py` that runs whatever `silica-compiler` is on `PATH` against a fixture and checks C1 to C7; the build-system tests in `tools/test_build_system/test_silica.py` and the `silica_component` fixture | `tools/silica/test/test_contract.py`, `tools/silica/test/fixtures/**`, `tools/test_build_system/test_silica.py`, `tools/test_build_system/build_test_app/silica_component/**` | contract text only for `test_contract.py`; A3 and A4 for the build-system tests | 7 |
| A9 | Documentation | `docs/en/api-guides/silica.rst`, `docs/en/api-reference/system/silica_rt.rst`, edits to `build-system.rst`, `idf-tools.rst`, root `README.md` | drafts from the plan; final pass after A3 to A7 | 8 |
| A10 | CI bookkeeping: `.gitlab/ci/rules.yml` pattern, `CODEOWNERS`, removing the temporary exclude line, pre-commit green | `.gitlab/ci/rules.yml`, `.gitlab/CODEOWNERS`, `tools/ci/exclude_check_tools_files.txt`, `tools/ci/executable-list.txt` | last | 7 |

### 5.2 Track B: work inside the compiler (Silica repository)

Described only to the level needed to size and sequence it. Detail belongs in the compiler's
own design documents.

| ID | Package | Where in the compiler | Depends on | Contract items |
|---|---|---|---|---|
| B1 | Driver: `--version`, `--help`, target selection (`SILICA_TARGET` or `--target`, or per-target binaries), unchanged `silica.config` and exit-75 behaviour | `src_selfhost/main_driver*.silica`, `main_lists.silica`, `emit_target.mk` | none | C1, C3, C4, C5 |
| B2 | Release packaging: a `macos-arm64` archive with `bin/silica-compiler`, stable URL, sha256. Can be done first with the **current** compiler so A1 has a real archive to test against | compiler repo release process | none | W6 |
| B3 | Runtime design decision: emitted-assembly runtime per ISA (today's shape) **or** emit calls into a C runtime that lives in this fork's `silica_rt` (recommended; see section 6). Must be decided before B4 and B5 start | design document | none | W2 |
| B4 | RISC-V backend: `emitter/esp_riscv32/` producing GNU-syntax ELF assembly, ilp32 ABI, rv32imc/rv32imac, ELF symbol naming without the Mach-O underscore, section placement, atom/int/string rodata | new emitter tree | B3 | W1, C2, C6, C7 |
| B5 | Xtensa backend: `emitter/esp_xtensa/`, windowed ABI as used by ESP-IDF, otherwise as B4 | new emitter tree | B3; shares ELF-common pieces with B4 | W1, C2, C6, C7 |
| B6 | Runtime implementation per B3: either the FreeRTOS-based generated runtime inside B4/B5, or the emitter-side call sequences into the `silica_rt` C API from A5 | emitter tree, or the emitter plus A5's header | B3; A5 header if the C-runtime option is chosen | W2, W3 |
| B7 | FFI link manifest: allow `link_library` to name an ESP-IDF component without a `dangerous_exposure_source/lib/*.a` copy | `src_selfhost/ffi/ffi_link_manifest.silica` and the checker | none | W5 |
| B8 | Diagnostics with file:line:column in a fixed format | `src_selfhost/diagnostics/` | none | W7 |
| B9 | Inbound FFI for ISR and FreeRTOS callbacks, or the documented queue-based pattern | frontend and emitter | B6 | W4 |

### 5.3 Interfaces to freeze first

These are written down before the parallel work starts and changed only by agreement:

1. **The compiler contract** (section 3, C1 to C7). Owner: shared. A8's `test_contract.py`
   is the executable form of it; both tracks run it.
2. **The wrapper command line** for `tools/silica/silica_build.py`: `--compiler`,
   `--target`, `--work-dir`, `--component-dir`, `--output-list`, `--depfile`,
   `--entry-symbol`, followed by unit paths; exit codes; the format of the output list.
   Owner: A2. Consumers: A3, A4, A8.
3. **The `__silica_add_sources(component_lib sources_var)` CMake signature** in
   `tools/cmake/silica.cmake`. Owner: A3. Consumer: A4.
4. **The `silica_rt` C API header** (`components/silica_rt/include/silica_rt.h`): entry
   hook name, spawn, send, region allocation, registry init. Owner: A5. Consumers: B6, A7, A9.
5. **The tool name and lookup order** (`silica-compiler-<target>`, then
   `silica-compiler-<xtensa|riscv32>`, then `silica-compiler`). Owner: A1 and B1 jointly.

### 5.4 What runs in parallel

Wave 1, all start at once, no dependencies:

* Track A: A1, A2, A5, A6, A8 (`test_contract.py` and fixtures only), A9 (drafts).
* Track B: B1, B2, B3, B7, B8.

Wave 2, starts as soon as the named prerequisite lands:

* A3 after the A2 interface (not after A2 is finished).
* A4 after the A3 skeleton exists.
* B4 and B5 after the B3 decision; they run in parallel with each other.
* B6 after B3, and after A5's header if the C-runtime option is chosen.

Wave 3:

* A7 after A3 and A5. A8's build-system tests after A3 and A4. A9 final pass after A3 to A7.
* B9 after B6.

Wave 4, joint: phase 9 swap-in, once B1, B2, B4 or B5, and B6 pass `test_contract.py` and the
`hello_world_silica` example runs on hardware.

Suggested agent assignment for this fork, five agents with no shared files:

| Agent | Packages | Notes |
|---|---|---|
| 1 | A2, then A3 | Owns the wrapper and the classic glue; publishes the interfaces in 5.3 items 2 and 3 first |
| 2 | A1, then A4 | Tool registration while waiting for the A3 skeleton |
| 3 | A5, then A7 | Runtime component, then the example that exercises it |
| 4 | A8, then A6 | Contract test first so Track B can run it early; templates after |
| 5 | A9, then A10 | Docs drafted from the plan, CI bookkeeping at the end |

For the compiler repository the natural split is one agent each for B1 with B2, B4, B5,
B7 with B8, and one for B3 followed by B6 and B9. B4 and B5 should agree on the shared
ELF-emission helpers before either starts.

### 5.5 Phase-to-package map

| Phase | Packages |
|---|---|
| 0 placeholder | done |
| 1 tool registration | A1 (B2 supplies a real archive later) |
| 2 classic build system | A2, A3 |
| 3 cmakev2 | A4 |
| 4 runtime component | A5 (B3, B6 for real behaviour) |
| 5 idf.py and templates | A6 |
| 6 example | A7 |
| 7 tests | A8, A10 |
| 8 docs | A9 |
| 9 swap-in | joint, gated on B1, B2, B4 or B5, B6 |

---

## 6. Risks and open questions

* **Assembler dialect (C2).** The current emitter writes Mach-O syntax. GNU ELF syntax for two
  ISAs is new emitter work; if the compiler instead emits object files directly some day, the
  wrapper's "add generated `.sams` as ASM sources" step becomes "add generated `.o` as object
  sources". Keep that switch local to `tools/cmake/silica.cmake`.
* **Whole-set compilation vs. Ninja parallelism.** One compiler run per component serialises
  that component's units. Acceptable now; revisit if the compiler grows per-unit mode.
* **Cross-component Silica modules.** Deferred. Option: a global `SILICA_MODULE_PATHS` build
  property that the wrapper folds into every component's `silica.config`, with `.iface`
  files of already-compiled components passed as inputs once the compiler supports reading
  them without recompiling.
* **Non-macOS hosts.** The real compiler is macOS-only for now. On Linux or Windows the
  wrapper must fail at configure time with a hint that names the host requirement, and the
  build-system tests on those hosts run only against the placeholder. The placeholder itself is
  a Python script with a shebang: fine on POSIX, and on Windows the package must ship a
  `.cmd` shim or the wrapper must invoke it through `sys.executable`. Decide in phase 1.
* **Licence.** Confirm the compiler's licence before adding the `tools.json` entry; the
  schema requires a `license` field.
* **Module basename uniqueness** across a whole firmware image is a language rule, not a build
  rule. It will bite large projects; raise with the language team.
* **Where the runtime lives (decision B3).** Today the emitter generates the actor runtime as
  assembly text, about 4,000 lines for one ISA. Reproducing that for Xtensa and RISC-V doubles
  the backend work and keeps the runtime out of reach of ESP-IDF's debugging tools. The
  recommended alternative is for the ESP backends to emit calls into a C runtime in this
  fork's `silica_rt` component, built on FreeRTOS tasks, queues and `heap_caps_*`. That moves
  most of W2 into Track A, shrinks B4 and B5 to code generation, and makes the runtime
  testable with ordinary ESP-IDF unit tests. The cost is that the emitter's shape changes from
  "generate runtime" to "call runtime", so the decision has to come before B4 and B5 start,
  and the `silica_rt` C API (interface 5.3 item 4) becomes part of the compiler contract.
* **ccache.** The compiler is not a C compiler, so `IDF_CCACHE_ENABLE` does not cover it.
  Determinism (C6) plus the depfile is the substitute.

---

## 7. File index for this plan

New:

* `SILICA_INTEGRATION_PLAN.md` (this file)
* `tools/silica/bin/silica-compiler` (phase 0)
* `tools/silica/README.md` (phase 0)
* `tools/silica/silica_build.py` (phase 2)
* `tools/silica/package_placeholder.py` (phase 1)
* `tools/silica/test/` (phase 7)
* `tools/cmake/silica.cmake` (phase 2)
* `components/silica_rt/` (phase 4)
* `tools/templates/sample_project_silica/` (phase 5)
* `examples/get-started/hello_world_silica/` (phase 6)
* `tools/test_build_system/test_silica.py` (phase 7)
* `docs/en/api-guides/silica.rst`, `docs/en/api-reference/system/silica_rt.rst` (phase 8)

Modified:

* `tools/ci/executable-list.txt`, `tools/ci/exclude_check_tools_files.txt` (phase 0)
* `tools/tools.json`, `tools/tools_schema.json` if needed, `install.sh`, `install.bat`,
  `install.ps1`, `install.fish`, `tools/install_util.py` (phase 1)
* `CMakeLists.txt`, `tools/cmake/{component,build,utilities,project,toolchain_flags}.cmake`,
  `tools/cmake/project_description.json.in` (phase 2)
* `tools/cmakev2/{component,compat,build,utilities,project,idf}.cmake` (phase 3)
* `tools/ci/check_copyright_config.yaml` (phase 4)
* `tools/idf_py_actions/{create_ext.py,hints.yml}` (phase 5)
* `.gitlab/ci/rules.yml`, `.gitlab/CODEOWNERS` (phase 7)
* `docs/en/api-guides/build-system.rst`, `docs/en/api-guides/tools/idf-tools.rst`,
  `README.md` (phase 8)

## 8. Alternatives considered

* **Full CMake language (`SILICA`)** with `CMakeDetermineSILICACompiler.cmake`,
  `CMakeSILICAInformation.cmake`, `CMakeTestSILICACompiler.cmake` and `project(... SILICA)`.
  Cleanest long-term shape, but it requires a per-file, argument-driven compiler and changes
  every `ENABLED_LANGUAGES` branch in cmakev2. Revisit when the compiler has a CLI.
* **`IDF_TOOLCHAIN=silica`.** Rejected: gcc/clang still assemble and link, and the Kconfig
  `IDF_TOOLCHAIN_*` symbols gate C-specific optimisation options.
* **Vendoring the compiler's `project_makefiles/`.** Rejected: they are macOS/clang specific
  and would couple this repo to the compiler tree. The Python wrapper reimplements the three
  ideas that matter (topological order, exit-75 loop, generated-file collection). See
  section 2.1 for the stage-by-stage comparison and the make-based variant.
