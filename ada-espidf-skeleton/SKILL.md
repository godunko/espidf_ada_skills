---
name: ada-espidf-skeleton
description: "Create starter skeletons for ESP-IDF firmware with Ada integration. Use when asked to scaffold, bootstrap, initialize, or generate templates for esp-idf, esp32 project with Ada integration."
argument-hint: "Target name, chip"
user-invocable: true
---

# ESP-IDF + Ada Skeleton Scaffolding

## What This Skill Does
Creates minimal, buildable starter structure for:
- ESP-IDF project skeletons (ESP32 family)
- Ada project skeletons (GNAT-style)

## When To Use
Use this skill when the user asks to:
- create a new ESP-IDF project skeleton with Ada integration
- scaffold an Ada starter project
- bootstrap firmware + Ada side-by-side templates
- generate boilerplate files for quick iteration

## Inputs To Collect
Collect these values before generating files:
- Root folder name
- ESP-IDF target chip (for example `esp32`, `esp32c3`, `esp32s3`)

If the user did not provide values, choose safe defaults:
- root: `espidf-ada-skeleton`
- target: `esp32s3`

## Project Layout (Flat Structure)
```
my_project/
├── CMakeLists.txt              # ESP-IDF root
├── crates/                     # Local Ada crates (a0b-tools, xtensa-dynconfig, espidf_gnat_runtime, bb-runtimes)
├── main/
│   └── CMakeLists.txt          # ESP-IDF component config
├── source/
│   └── main.adb                # Ada entry point (main application logic)
├── my_project.gpr              # Ada project file
├── alire.toml                  # Ada dependencies
├── sdkconfig.defaults          # ESP-IDF SDK defaults (chip-specific)
├── README.md                   # Build instructions
└── .gitignore                  # Build artifacts to ignore
```

All Ada crate repositories and tool builds are handled by the agent during project creation.

## Procedure
1. Create folder structure at the project root.
2. Clone required Ada crate repositories directly (agent handles all checkouts):
   - Clone `https://github.com/godunko/a0b-tools` into `crates/a0b-tools` (always required)
   - Clone `https://github.com/godunko/xtensa-dynconfig` into `crates/xtensa-dynconfig` (Xtensa MCUs only: esp32, esp32s2, esp32s3)
   - Clone `https://github.com/godunko/espidf_gnat_runtime` into `crates/espidf_gnat_runtime`
   - Clone `https://github.com/alire-project/bb-runtimes.git` into `crates/bb-runtimes` with branch `gnat-fsf-15`
3. Generate ESP-IDF skeleton files:
   - `CMakeLists.txt` at project root
   - `main/CMakeLists.txt` with ExternalProject_Add for Ada build
   - `sdkconfig.defaults` for ESP-IDF SDK defaults. It is important:
     - to set "CONFIG_IDF_TARGET" to the correct target to ensure the ESP-IDF build system selects the right configuration for the chip
     - to set "CONFIG_LIBC_NEWLIB" to "y" to ensure the ESP-IDF build system uses newlib, which is required for the Ada runtime
4. Generate Ada skeleton files at the same root:
   - `<name>.gpr` project file
   - `source/main.adb` hello-world style entrypoint
   - `alire.toml` with dependencies and `[[pins]]` pointing to local `crates/`
5. Generate optional configuration files:
   - `.gitignore` for build artifacts
6. Add a `README.md` describing the combined workspace and build commands.
7. Build required Ada tool crates directly (agent handles all builds):
   - `alr -C crates/a0b-tools build` (always)
   - `alr -C crates/xtensa-dynconfig build` (for Xtensa MCUs only)
8. Verify the generated project using the ESP-IDF build flow:
   - `idf.py set-target <chip>` (if target is not already configured)
   - `idf.py build`
   - Do not use `alr build` at project root to verify the scaffold.
9. Validate that all referenced files exist and contain minimal compilable content.

## References to Template Repositories
- **ESP32-C3 Integrated Ada+ESP-IDF**: https://github.com/godunko/esp32c3_template
- **ESP32-S3 Integrated Ada+ESP-IDF**: https://github.com/godunko/esp32s3_template

These templates show production patterns for merging Ada and C builds via Alire+CMake integration.

## ESP-IDF & Ada Integration Notes
- **Agent-driven setup**: The agent handles all repository checkouts and tool builds directly. No manual scripts required.
- In VS Code, prefer ESP-IDF extension commands for build/flash/monitor operations.
- If the tool is available, set target via ESP-IDF command workflow instead of shell scripts.
- Ada integration uses Alire (`alr`) for dependency and runtime management.
- Project build/verification must use ESP-IDF (`idf.py build`), since Ada build is integrated through ESP-IDF CMake.
- **Local crate pins**: Ada crates (a0b-tools, xtensa-dynconfig, espidf_gnat_runtime, bb-runtimes) are checked out into `crates/` and pinned via `[[pins]]` in alire.toml.
- Xtensa MCUs require `xtensa-dynconfig` crate for configuration settings.
- Ada code is compiled into a partially linked object file, which is then linked into the final ESP-IDF binary (not a standalone executable).

## Output Requirements
- Strictly follow content of provided templates, but adapt as needed for the specific target chip.
- List of maintainers in `alire.toml` should use both name and email.
- Do not overwrite existing files without user confirmation.
- Keep generated files ASCII-only unless user explicitly requests otherwise.
- Use concise comments only where structure is non-obvious.

## Agent Implementation Details
The agent must execute the following operations autonomously:

**Repository Cloning**:
- Use `git clone` to checkout each crate into `crates/` directory
- For `bb-runtimes`, use `git clone -b gnat-fsf-15` to checkout the correct branch
- Verify each checkout completes successfully before proceeding

**Tool Building**:
- Execute `alr -C crates/a0b-tools build` in a terminal to build a0b-tools
- For Xtensa targets, execute `alr -C crates/xtensa-dynconfig build` after a0b-tools completes
- Ensure all builds complete without errors before generating project files

**Project Generation**:
- Generate all skeleton files based on the target chip specifications
- Replace template placeholders (`<name>`, `<target-triple>`, `<chip>`) with actual values
- Use `run_in_terminal` to execute all git and `alr` commands asynchronously

**Verification**:
- After all files are created and tools are built, execute verification steps via terminal
- Run `idf.py build` to ensure the project is buildable

## Skeleton Templates (Based on Official Templates)

### ESP-IDF Root `CMakeLists.txt`
```cmake
cmake_minimum_required(VERSION 3.16)

include($ENV{IDF_PATH}/tools/cmake/project.cmake)

# Minimal build: include only main and dependencies
idf_build_set_property(MINIMAL_BUILD ON)

project(<name>)  # Replace <name> with the actual project name

# Clean Ada build artifacts on rebuild
set_property(TARGET "${PROJECT_NAME}.elf" APPEND PROPERTY
    ADDITIONAL_CLEAN_FILES "${CMAKE_CURRENT_SOURCE_DIR}/.objs"
                           "${CMAKE_CURRENT_SOURCE_DIR}/runtime")
```

### ESP-IDF `main/CMakeLists.txt`
```cmake
idf_component_register()

ExternalProject_Add(app_main_build
    PREFIX ${COMPONENT_DIR}
    SOURCE_DIR ${COMPONENT_DIR}
    BUILD_IN_SOURCE 1
    BUILD_ALWAYS 1
    CONFIGURE_COMMAND ""
    BUILD_COMMAND alr build -- -cargs:c -DESP_PLATFORM
        "-D$<JOIN:$<TARGET_PROPERTY:app_main,INTERFACE_COMPILE_DEFINITIONS>,$<SEMICOLON>-D>"
        "-I$<JOIN:$<TARGET_PROPERTY:app_main,INTERFACE_INCLUDE_DIRECTORIES>,$<SEMICOLON>-I>"
    COMMAND_EXPAND_LISTS
    INSTALL_COMMAND ""
    BUILD_BYPRODUCTS ${COMPONENT_DIR}/../.objs/app_main.o
)

add_prebuilt_library(app_main "${COMPONENT_LIB}" REQUIRES freertos)

target_link_libraries(
    ${COMPONENT_LIB} INTERFACE ${COMPONENT_DIR}/../.objs/app_main.o)
```

### Ada Project File `<name>.gpr`

The `Target` attribute is the only chip-specific value. Select it based on the MCU family:
- Xtensa (ESP32, ESP32-S2, ESP32-S3): `"xtensa-esp32-elf"`
- RISC-V (ESP32-C3, ESP32-C6, etc.): `"riscv64-elf"`

```gpr
with "runtime/build_libgnat.gpr";   --  Base GNAT runtime
with "runtime/build_libgnarl.gpr";  --  Tasking GNAT runtime
--  These runtime project files are generated by a0b-tools

with "config/<name>_config.gpr";    --  Generated by Alire; <name> must match alire.toml name field

project <Name> is

   for Target use "<target-triple>";  --  "xtensa-esp32-elf" or "riscv64-elf"
   for Runtime ("Ada") use "runtime";

   for Source_Dirs use ("source");
   for Object_Dir use ".objs";
   for Main use ("main.adb");

   package Builder is
      --  Compiles to object file, not executable (linked by CMake)
      for Executable ("main.adb") use "app_main.o";
   end Builder;

   package Binder is
      for Switches ("Ada") use
        ("-D4k",
         --  Default secondary stack size = nn [kilo|mega] bytes
         --
         --  Runtime uses "static" allocation strategy for secondary stacks
         --  thus size of the secondary stack should be specified.
         "-Q2",
         --  Generate nnn additional default-sized secondary stacks
         --
         --  Some tasks are created by ESP-IDF, for example, for USB stack,
         --  IP stack, event handling. If these tasks calls Ada subprograms
         --  which use secondary stack this number should be adjusted.
         "-Mapp_main");
         --  Set the binder generated main function's name to app_main as
         --  this is required by the SDK.
   end Binder;

   package Compiler is
      for Switches ("Ada") use ("-g", "-O2");
   end Compiler;

   package Linker is
      for Default_Switches ("Ada") use ("-Wl,-r", "-nostdlib");
   end Linker;

end <Name>;
```

### Ada `alire.toml`

This template is shared across MCU families. Only these dependency differences are required:
- RISC-V MCUs: use `gnat_riscv64_elf = "^15"`
- Xtensa MCUs: use `gnat_xtensa_esp32_elf = "^15"`, add `xtensa_dynconfig = "*"`, and pin `xtensa_dynconfig`

```toml
name = "<name>"
description = "ESP-IDF + Ada Integrated Skeleton"
version = "0.1.0-dev"

authors = ["Your Name"]
maintainers = ["Your Name <email@example.com>"]
licenses = "Apache-2.0 WITH LLVM-exception"
tags = ["embedded", "<chip-tag>"]

[configuration]
generate_ada = false
generate_c = false
generate_gpr = true

[[depends-on]]
a0b_base = "*"
a0b_tools = "*"
espidf_gnat_runtime = "*"
<gnat-toolchain-dependency>  # gnat_riscv64_elf = "^15" or gnat_xtensa_esp32_elf = "^15"
<xtensa-dynconfig-dependency>  # Xtensa only: xtensa_dynconfig = "*"

[[actions]]
type = "pre-build"
command = ["a0b-runtime",
          "--bb-runtimes=crates/bb-runtimes/",
          "--svd=crates/espidf_gnat_runtime/svd/<chip>.svd",
          "--runtime-description=crates/espidf_gnat_runtime/runtime-<chip>.json",
          "--no-startup"]

[[pins]]
a0b_tools = { path = "crates/a0b-tools" }
<xtensa-dynconfig-pin>  # Xtensa only: xtensa_dynconfig = { path = "crates/xtensa-dynconfig" }
espidf_gnat_runtime = { path = "crates/espidf_gnat_runtime" }
```

### Ada `source/main.adb`
```ada
with Ada.Text_IO; use Ada.Text_IO;

procedure Main is
begin
   Put_Line ("Hello, Ada world! Skeleton application.");
end Main;
```

