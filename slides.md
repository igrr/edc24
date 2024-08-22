---
theme: seriph
highlighter: shiki
lineNumbers: false
transition: fade
title: Espressif DevCon24 — Mastering ESP-IDF Build System
layout: intro
addons:
  - "@katzumi/slidev-addon-qrcode"
  - slidev-addon-asciinema

---

# Mastering ESP-IDF Build System

## Ivan Grokhotkov<br>VP of Software Platforms @ Espressif

### 2024/09/04

---
layout: default
---

# Outline

<div class="grid grid-cols-2 gap-2">
<div>
<v-clicks>

- Part 1: Introduction
- Part 2: Creating an ESP-IDF application
- Part 3: Build system cookbook

</v-clicks>
</div>

<div class="container mx-auto">
<img src="https://api.qrserver.com/v1/create-qr-code/?size=180x180&data=https://igrr.github.io/edc24" class="object-center mx-auto py-10" width=180 height=180/>
<div class="text-center">

Slides: [https://igrr.github.io/edc24 <mdi-launch />](https://igrr.github.io/edc24)

Note: The talk is based on ESP-IDF v5.3. The code snippets might need to be adjusted for other ESP-IDF versions.

</div>
</div>
</div>

<style>
  .slidev-vclick-hidden {
  opacity: 0.3 !important;
  pointer-events: none;
};
</style>

---
layout: intro
---

# Part 1: Introduction


---

# Why this talk?

<v-clicks>

- I have a pile of code in my 'main' directory, how to structure it?
- Why do I need to specify paths like this `../../../esp-idf/components/spi_flash/include` (you don't!)
- My app grew to support multiple products, how to structure the code?

</v-clicks>


---

# What does ESP-IDF build system do?

<div class="grid grid-cols-2 gap-2">

<div>
<img src="/project.png" class="h-100"/>
</div>

<v-clicks>

1. Builds ESP-IDF _applications_ consisting of _components_.
2. Handles compile-time _configuration_ based on Kconfig.
3. Generates partition table, linker scripts, bootloader, filesystem images.
4. Provides integration with tools: configuration, flashing, serial monitor, debugging, emulation.
5. Integrates with [ESP Component Registry <mdi-launch />](https://components.espressif.com/).

</v-clicks>

</div>

---

# idf.py, CMake, Ninja?

<div class="grid grid-cols-2 gap-2">

<div>
<img src="/build-system-parts.png" />
</div>

<div>
<v-clicks>

1. ESP-IDF build system is implemented in CMake.
2. `idf.py` CLI frontend or IDE extension runs CMake.
3. CMake executes CMakeLists.txt files in application and ESP-IDF and generates a build script.
4. The build script is executed by Ninja.
5. Various build steps run the cross-compiler toolchain and IDF-specific tools where necessary.

</v-clicks>

<v-click>

<hr>

In this talk, we will focus on using this system from application perspective.
</v-click>

</div>
</div>


---

# Why not just "modern CMake"?

<v-clicks>

- Until IDF v4.x, a GNU Make -based build system was used:

  <div class="grid grid-cols-2 gap-2">
  <div>

  ```make
  # project Makefile:
  PROJECT_NAME := app
  include $(IDF_PATH)/make/project.mk
  ```
  </div>
  <div>

  ```make
  # main/component.mk
  COMPONENT_ADD_INCLUDEDIRS := .
  COMPONENT_SRCDIRS := src port
  ```

  </div>
  </div>

- CMake-based build system introduced in IDF v4 tried to keep usage as similar as possible.
- IDF build system does use "modern CMake" concepts internally. However,
  1. The approach to adding (registering) libraries differs.
  2. A plain CMake library can not be used as an IDF component without wrapping it.

- See some of the Github issues for discussions of incompatibilities between IDF build system and plain CMake: [#9929](https://github.com/espressif/esp-idf/issues/9929), [#10579](https://github.com/espressif/esp-idf/issues/10579).
- Future IDF versions will move closer to plain CMake.

</v-clicks>

---
layout: intro
---

# Part 2: Let's build an app

---

# Creating a new project

Let's use `idf.py create-project` to create an app.


<div class="grid grid-cols-2 gap-2">

<div>

```
$ idf.py create-project app
```

</div>

<div v-click>

```
📂 app/
├── 🔧 CMakeLists.txt
└── 📂 main/
    ├── 📜 app.c
    └── 🔧 CMakeLists.txt
```
</div>

<div v-click>

###### Project CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.16)

include($ENV{IDF_PATH}/tools/cmake/project.cmake)
project(app)
```
</div>

<div v-click>

###### `main` component CMakeLists.txt

```cmake
idf_component_register(SRCS "app.c"
                       INCLUDE_DIRS ".")
```
</div>

<div v-click>

###### app.c
```c
#include <stdio.h>

void app_main(void){}
```
</div>

</div>


---

# Let's build it

<RenderWhen context="main">
  <Asciinema src="casts/app_build.cast" :playerProps="{speed: 2, rows: 12}"/>
</RenderWhen>

---

# A look at the result

<div class="grid grid-cols-2 gap-2">

<div>

```
📂 app/
├── 📁 build/
├── 📂 main/
│   ├── 📜 app.c
│   └── 🔧 CMakeLists.txt
├── 🔧 CMakeLists.txt
└── sdkconfig
```

</div>

<div></div>

<div v-click>

###### Build directory

```
📂 build/
├── 📂 config/                 # Generated configuration
│   └── 📜 sdkconfig.h
├── 📂 bootloader/             # 2nd stage bootloader
│   └── bootloader.bin
├── 📂 partition_table/        # Partition table
│   └── partition-table.bin
├── app.bin                    # Application binary
├── app.elf                    # ... ELF file
├── app.map                    # ... Linker map file
├── compile_commands.json      # Compilation database
├── project_description.json   # Project metadata
├── flasher_args.json          # esptool.py arguments
└── flash_args                 # esptool.py arguments
```

</div>
<div v-click>

###### `sdkconfig` file

- Stores project configuration
- Can be modified manually or using GUI tools
- Can be tracked in Git or added to .gitignore

</div>
</div>

---

# Configuration system

<v-switch>
<template #0>
  First-time config generation:
  <img src="/kconfig1.png" class="w-190"/>
</template>
<template #1>
  Editing sdkconfig:
  <img src="/kconfig2.png" class="w-190"/>
</template>
<template #2>
  Re-generation:
  <img src="/kconfig3.png" class="w-190"/>
</template>
<template #3>
  Extract new defaults:
  <img src="/kconfig4.png" class="w-190"/>
</template>
<template #4>

- Each component defines its configuration in a `Kconfig` file
- `sdkconfig` is created based on defaults in `sdkconfig.defaults` and `Kconfig` files
- `sdkconfig.h` and other files are generated based on `sdkconfig`
- `sdkconfig` or `sdkconfig.defaults`?
  - Keep `sdkconfig` in Git if you want to ensure config stability
  - Recommended: keep `sdkconfig.defaults` in Git
    - To allow new default values from components
    - To document which options are different from defaults

</template>
</v-switch>


---

# Component CMakeLists file, idf_component_register

<div class="grid grid-rows-2 gap-2">
<div>
```cmake
idf_component_register(
                       SRCS src1.c src2.c

                       SRC_DIRS dir1 dir2
                       EXCLUDE_SRCS src1 src2

                       INCLUDE_DIRS dir1 dir2
                       PRIV_INCLUDE_DIRS dir1 dir2

                       REQUIRES component1 component2
                       PRIV_REQUIRES component1 component2

                       LDFRAGMENTS ldfragment1 ldfragment2
                       EMBED_FILES file1 file2
                       EMBED_TXTFILES file1 file2 
                       WHOLE_ARCHIVE)
```
</div>

<div>

- Define source files (`SRCS`) or directories (`SRC_DIRS` + `EXCLUDE_SRCS`)
- Define include directories (`INCLUDE_DIRS`, `PRIV_INCLUDE_DIRS`)
- Define dependencies (`REQUIRES`, `PRIV_REQUIRES`)

</div>
</div>

---

# Adding source files and headers

<div class="grid grid-cols-2 gap-2">
<div>

```{3-9}
📂 app/
├── 📁 build/
├── 📂 main/
│   ├── 📂 include/
│   │   └── 📜 app_common.h
│   ├── 📂 src/
│   │   ├── 📜 main.c
│   │   └── 📜 app_common.c
│   └── 🔧 CMakeLists.txt
├── 🔧 CMakeLists.txt
└── sdkconfig
```

###### `main` component CMakeLists.txt

```cmake
idf_component_register(SRCS "src/main.c"
                            "src/app_common.c"
                       PRIV_INCLUDE_DIRS "include")
```

</div>
<div>

###### Tips

- Use `SRCS` rather than `SRC_DIRS` if possible
- Don't use paths outside of a component (`../components/other/include`)
- Use double quotes if the file or directory name contains spaces

</div>
</div>

---

# Adding dependencies on external components

<div class="grid grid-cols-2 gap-2">
<div>

```{3-4,11,15}
📂 app/
├── 📁 build/
├── 📂 managed_components/
│   └── 📁 espressif__led_strip/
├── 📂 main/
│   ├── 📂 include/
│   │   └── 📜 app_common.h
│   ├── 📂 src/
│   │   ├── 📜 main.c
│   │   └── 📜 app_common.c
│   ├── 🧩 idf_component.yml
│   └── 🔧 CMakeLists.txt
├── 🔧 CMakeLists.txt
├── sdkconfig
└── dependencies.lock
```

###### `idf_component.yml`

```yaml
dependencies:
  espressif/led_strip: "^2.5.4"
```

</div>
<div>

1. `idf_component.yml` file defines external dependencies of the component
2. External components are stored in `managed_components/` directory
3. Add `managed_components` to `.gitignore` and don't modify it
4. `dependencies.lock` stores actual versions of dependencies, keep it for reproducible builds

<hr>

- [Example  <mdi-launch />](https://github.com/espressif/esp-idf/tree/master/examples/build_system/cmake/component_manager)
- [idf_component.yml docs  <mdi-launch />](https://docs.espressif.com/projects/idf-component-manager/en/latest/reference/manifest_file.html)

</div>
</div>

---

# Adding Kconfig options

<div class="grid grid-cols-2 gap-2">
<div>

```{10}
📂 app/
└── 📂 main/
    ├── 📂 include/
    │   └── 📜 app_common.h
    ├── 📂 src/
    │   ├── 📜 main.c
    │   └── 📜 app_common.c
    ├── 🧩 idf_component.yml
    ├── 🔧 CMakeLists.txt
    └── Kconfig
```

</div>
<div>

```
menu "App main"

    config APP_DIAGNOSTICS
        bool "Enable diagnostic reports"
        default n
        help
            This option enables sending diagnostic reports
            to the server

    config APP_DIAGNOSTICS_URL
        string "Diagnostic reports endpoint"
        depends on APP_DIAGNOSTICS
        default "https://api.mycompany.com/v1/device_diag"

endmenu
```

</div>
</div>

- Add options to `Kconfig` or `Kconfig.projbuild`
- [Kconfig syntax reference <mdi-launch />](https://www.kernel.org/doc/html/next/kbuild/kconfig-language.html#kconfig-syntax)
- Prefer Kconfig for things that really need to be known at compile time
- Prefer NVS or another storage method for factory settings

---

# As the app grows...


<div class="grid grid-cols-2 gap-2">
<div>

```
📂 app/
└── 📂 main/
    ├── 📂 include/
    ├── 📂 src/
    │   ├── 📜 main.c
    │   ├── 📜 app_common.c
    │   ├── 📜 flash.c
    │   ├── 📜 sdcard.c
    │   ├── 📜 console.c
    │   ├── 📜 defaults.c
    │   ├── 📜 board.c
    │   ├── 📜 wifi_connect.c
    │   ├── 📜 backend.c
    │   └── 📜 settings.c
    └── 🔧 CMakeLists.txt
```

</div>
<div>

```
📂 app/
├── 📂 components/
│   ├── 📂 app_storage/
│   │   ├── 📂 include/
│   │   │   └── 📜 app_storage.h
│   │   ├── 📜 flash.c
│   │   └── 📜 sdcard.c
│   └── 📂 app_settings/
│       ├── 📂 include/
│       │   └── 📜 app_settings.h
│       ├── 📜 settings.c
│       └── 📜 defaults.c
└── 📂 main/
```

</div>
</div>

Features may be split into their own components:

- To simplify maintenance
- To reuse them between products
- To force clear interfaces between parts of the code


---

# Creating new components

<div class="grid grid-cols-2 gap-2">
<div>


```{4-12,14}
📂 app/
├── 📁 build/
├── 📁 managed_components/
├── 📂 components/
│   └── 📂 app_common/
│       ├── 📂 include/
│       │   └── 📜 app_common.h
│       ├── 📜 app_common.c
│       └── 🔧 CMakeLists.txt
├── 📂 main/
│   ├── 📂 src/
│   │   └── 📜 main.c
│   ├── 🧩 idf_component.yml
│   └── 🔧 CMakeLists.txt
├── 🔧 CMakeLists.txt
├── sdkconfig
└── dependencies.lock
```

</div>

<div>

- `components` directory contains components local to the project
- The structure is the same as for the `main` component

</div>

</div>

---

# New component

<div class="grid grid-cols-2 gap-2">
<div>

###### `CMakeLists.txt`

```cmake
idf_component_register(SRCS "app_common.c"
                       INCLUDE_DIRS "include")
```

###### `app_common.h`

```c
#pragma once

#include "esp_err.h"

#ifdef __cplusplus
extern "C" {
#endif

esp_err_t app_common_init_storage(void);

#ifdef __cplusplus
}
#endif
```

</div>
<div>

###### `app_common.c`

```c
#include "esp_check.h"
#include "nvs_flash.h"
#include "app_common.h"

static const char* TAG = "app_common";

esp_err_t app_common_init_storage(void)
{
    ESP_RETURN_ON_ERROR(nvs_flash_init(),
      TAG, "Failed to initialize NVS");

    return ESP_OK;
}
```


</div>
</div>


---

# Let's try to build it

<div v-click>

```
.../app/components/app_common/app_common.c:2:10: fatal error: nvs_flash.h: No such file or directory
    2 | #include "nvs_flash.h"
      |          ^~~~~~~~~~~~~
compilation terminated.
```
</div>

<div v-click>

idf.py hint:

```
Compilation failed because app_common.c (in "app_common" component) includes nvs_flash.h,
provided by nvs_flash component(s).

However, nvs_flash component(s) is not in the requirements list of "app_common".

To fix this, add nvs_flash to PRIV_REQUIRES list of idf_component_register call
in app/components/app_common/CMakeLists.txt.
```
</div>


<div v-click>

To use functions defined in another component, it must be added to `PRIV_REQUIRES`:

###### `CMakeLists.txt`

```cmake
idf_component_register(SRCS "app_common.c"
                       INCLUDE_DIRS "include"
                       PRIV_REQUIRES "nvs_flash")
```
</div>



---

# Requirements

<v-clicks>

- `PRIV_REQUIRES` specifies the list of components used by current component's sources.
  - Automatically propagates public include directories, compile options, compile definitions, link options of dependents to the current component.
  - Equivalent to `target_link_libraries(... PRIVATE ...)`
- `REQUIRES` specifies the list of components used by current component's header files.
  - In addition to doing what `PRIV_REQUIRES` does, also propagates include directories, compile options, compile definitions, link options from dependents to other components which depend on it.

</v-clicks>

<div v-click>
<hr>

| `idf_component_register` keyword | CMake keyword | Dependency's properties propagated to.. |
|----------------------------------|-------------|------------------------------------------|
| `PRIV_REQUIRES`                  | `PRIVATE`   | only component sources                   |
| `REQUIRES`                       | `PUBLIC`    | both to component sources and dependents |

</div>


---

# Example of REQUIRES and PRIV_REQUIRES

<img src="/component-deps.png" >

<v-clicks>

1. app_common.c uses nvs_flash.h => app_common CMakeLists needs `PRIV_REQUIRES nvs_flash`
2. main.c uses app_common.h => main CMakeLists needs `PRIV_REQUIRES app_common`
3. app_common.h uses esp_err.h => app_common CMakeLists needs `REQUIRES esp_common`

</v-clicks>

---

# Special behavior of `main` component

<div>

In `main` component only, not specifying any dependencies (`REQUIRES` or `PRIV_REQUIRES`) automatically makes `main` dependent on **all components in the build**.

</div>

_Note: This behavior will most likely get deprecated in the future._

[Docs: Main component requirements <mdi-launch />](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-guides/build-system.html#main-component-requirements)

---

# Common requirements

<div>

For convenience and backward compatibility, the following components are automatically added as dependencies to every other component:

- cxx
- newlib 
- freertos 
- esp_hw_support
- heap
- log
- soc
- hal
- esp_rom
- esp_common
- esp_system

</div>

_Note: this list will most likely shrink in the future._

[Docs: Common component requirements <mdi-launch />](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-guides/build-system.html#common-component-requirements)

---

# Limitations of requirements

#### 1. Requirements cannot depend on configuration.

```cmake
set(reqs ...)
if(CONFIG_APP_ENABLE_PSRAM)
  list(APPEND reqs esp_psram)
endif()
# Doesn't work:
idf_component_register(... PRIV_REQUIRES ${reqs})
```

<div v-click>

```cmake
set(reqs ...)
idf_component_register(... PRIV_REQUIRES ${reqs})

# Can add a dependency late (if the component is included in the build)
if(CONFIG_APP_ENABLE_PSRAM)
  idf_component_optional_requires(PRIVATE esp_psram)
endif()
```
</div>


<div v-click>

#### 2. Component CMakeLists file is evaluated twice

... once to find the requirements, second time to actually create a library.

Don't call CMake commands which don't work in [script mode <mdi-launch />](https://cmake.org/cmake/help/latest/manual/cmake-commands.7.html#scripting-commands) before calling `idf_component_register`.

</div>

---

# Compiling only what is needed

- In project CMakeLists.txt file, `COMPONENTS` variable can be set to the list of components to be built.
- The build system will build these components and their dependencies.
- If `COMPONENTS` is not set, all components are built.

[Docs <mdi-launch />](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-guides/build-system.html#including-components-in-the-build)

<hr>

<div class="grid grid-cols-2 gap-2" v-click>
<div>

###### Project CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.16)

include($ENV{IDF_PATH}/tools/cmake/project.cmake)
set(COMPONENTS main)
project(app)
```

</div><div>

###### main/CMakeLists.txt

```cmake
idf_component_register(SRCS "src/main.c"
                       PRIV_REQUIRES app_common)

```

</div>
</div>

<div v-click>

<hr>

Note: to enable certain features, don't forget to add respective components into the build:

- `esp_psram` for PSRAM support
- `esp_gdbstub` for GDB Stub
- `espcoredump` for Core Dump

</div>



---
layout: intro
---

# Part 3: Cookbook



---

# Conditionally adding files

###### Conditionally adding files

Prepare the list of sources:

```cmake
set(srcs "common.c")

if(CONFIG_ENABLE_FEATURE_X)
  list(APPEND srcs "feature_x.c")
endif()

idf_component_register(SRCS ${srcs})
```

Or add sources after `idf_component_register`:

```cmake
idf_component_register(SRCS common.c)

if(CONFIG_ENABLE_FEATURE_X)
  target_sources(${COMPONENT_LIB} PRIVATE "feature_x.c")
endif()
```


---

# Adding compiler flags

### For the whole component:

```cmake
target_compile_options(${COMPONENT_LIB} PRIVATE -O3)
```

[Reference: target_* commands <mdi-launch />](https://cmake.org/cmake/help/latest/manual/cmake-commands.7.html#project-commands)

### For a single source file:

```cmake
set_source_files_properties(some_file.c
    PROPERTIES COMPILE_OPTIONS
    -Wno-format)
```

[Reference: Properties on source files <mdi-launch />](https://cmake.org/cmake/help/latest/manual/cmake-properties.7.html#id6)

### For the whole project:

```cmake
idf_build_set_property(COMPILE_OPTIONS "-fsanitize=undefined" APPEND)
```

[Reference: IDF build properties <mdi-launch />](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-guides/build-system.html#cmake-build-properties)

---

# Setting various options


| **Purpose** | **`target_*` function** | **CMake property** | **IDF build property** |
| ------- | ------------------- | -------------- | ------------------ |
| Compiler flags | `target_compile_options` | `COMPILE_OPTIONS` | `COMPILE_OPTIONS` |
| Compiler flags for C | `target_compile_options` | `C_COMPILE_OPTIONS` | `C_COMPILE_OPTIONS` |
| Compiler flags for C++ | `target_compile_options` | `CXX_COMPILE_OPTIONS` | `CXX_COMPILE_OPTIONS` |
| Preprocessor macros | `target_compile_definitions` | `COMPILE_DEFINITIONS` | `COMPILE_DEFINITIONS` | 
| Linker options | `target_link_options` | `LINK_OPTIONS` | `LINK_OPTIONS` |


---

# Wrapping external libraries

<v-click>

### CMake libraries

Libraries which support being included via `add_subdirectory` can typically be wrapped easily:

[Example <mdi-launch />:](https://github.com/espressif/idf-extra-components/blob/master/fmt/CMakeLists.txt)
```cmake
idf_component_register( )

set(FMT_INSTALL OFF)
add_subdirectory(fmt)

target_link_libraries(${COMPONENT_LIB} INTERFACE fmt::fmt)
```

</v-click>
<v-click>

### Other cases

- Use `ExternalProject_Add` to build the library.
- Call `add_prebuilt_library` to create a pre-compiled library target.
- Call `target_link_libraries` to link the pre-compiled library to the wrapper component.
- Examples: [tinyxml2 <mdi-launch />](https://github.com/espressif/esp-idf/blob/master/examples/build_system/cmake/import_lib/components/tinyxml2/CMakeLists.txt), [thorvg <mdi-launch />](https://github.com/espressif/idf-extra-components/blob/master/thorvg/CMakeLists.txt), [eigen <mdi-launch />](https://github.com/espressif/idf-extra-components/blob/master/eigen/CMakeLists.txt)

</v-click>

---

# Embedding data and creating filesystems

<v-clicks>

- Embed text or binary data:
  - `idf_component_register(... EMBED_TXTFILES cert.pem EMBED_FILES logo.png)`
  - [Docs <mdi-launch />](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-guides/build-system.html#embedding-binary-data)
- Create FAT filesystem and flash it together with the app:
  - `fatfs_create_spiflash_image(partition_name data_dir FLASH_IN_PROJECT)`
  - [Example <mdi-launch />](https://github.com/espressif/esp-idf/tree/master/examples/storage/fatfsgen), [Docs <mdi-launch />](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-reference/storage/fatfs.html#build-system-integration-with-fatfs-partition-generator)
- Create LittleFS filesystem and flash it together with the app:
  - `littlefs_create_partition_image(partition_name data_dir FLASH_IN_PROJECT)`
  - [Example <mdi-launch />](https://github.com/espressif/esp-idf/tree/master/examples/storage/littlefs)
- Generate NVS partition and flash it together with the app:
  - `nvs_create_partition_image(nvs nvs_data.csv FLASH_IN_PROJECT)`
  - [Example <mdi-launch />](https://github.com/espressif/esp-idf/tree/master/examples/storage/nvsgen)

</v-clicks>

---

# Doing work before or after the build

<div class="grid grid-cols-2 gap-2">
<div v-click>

### Generating intermediate files

```cmake
# Define how to produce ${custom_binary} file:
add_custom_command(OUTPUT ${custom_binary}
    COMMAND ${script_name} ${args}
    DEPENDS ${source_file}
    )

# Make component build dependent on that target
add_custom_target(custom_binary DEPENDS ${custom_binary})
add_dependencies(${COMPONENT_LIB} custom_binary)

# Use the generated file.
# For example, include the binary into the app:
target_add_binary_data(
  ${COMPONENT_LIB} ${custom_binary} BINARY)
```

- Use `add_custom_command` to define how to build a file, then use it in later build steps.
- Useful for code generation, converting data between formats, etc.

</div>

<div>

<div v-click>

### Performing post-build steps

```cmake
add_custom_command(TARGET app POST_BUILD
  COMMAND ${custom_script_to_run}
  VERBATIM)
```

- `POST_BUILD` custom command runs automatically after the target is built.

</div>

<div v-click>
<hr>

### Custom commands to run manually

```cmake
add_custom_target(custom_step
    COMMAND ...
    DEPENDS ...
    )
```

- Custom target can be invoked manually (`idf.py custom_step`)
- Can specify dependencies to make sure other targets are built (e.g. `app`)

</div>

</div>

</div>

---

# Customizing

<v-clicks>

- Change `main` component name using `EXTRA_COMPONENT_DIRS` ([docs <mdi-launch />](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-guides/build-system.html#renaming-main-component))
- Change sdkconfig file name or path. For example: `set(SDKCONFIG ${CMAKE_BINARY_DIR}/sdkconfig)`
- Change the build directory. For example: `idf.py -B build.esp32 build flash monitor`

</v-clicks>

---

# Multiple build configurations

<div>Often a single app has to support multiple hardware or product configurations. In IDF, this can be achieved by creating multiple sdkconfig.default files:</div>

<div class="grid grid-cols-2 gap-2">
<div v-click>

```
📂 app/
📂 app/
├── 📁 components/
├── 📁 main/
├── 📂 build.prod1/
│   └── sdkconfig
├── 📂 build.prod2/
│   └── sdkconfig
├── 📂 prod/
│   ├── 1                      # Build args, product 1
│   └── 2                      # ...                 2
├── 🔧 CMakeLists.txt
├── sdkconfig.defaults         # Common settings
├── sdkconfig.defaults.prod1   # Settings for product 1
└── sdkconfig.defaults.prod2   # ...                  2
```

- [Example <mdi-launch />](https://github.com/espressif/esp-idf/tree/master/examples/build_system/cmake/multi_config)

</div>

<div>

<div v-click>

###### idf.py verbose usage

```shell
idf.py \
  -B build.prod1 \
  -D SDKCONFIG_DEFAULTS="sdkconfig.defaults; \
                         sdkconfig.defaults.prod1" \
  -D SDKCONFIG=build.prod1/sdkconfig \
  build flash monitor
```

</div>

<div v-click>

###### prod/1 Arguments file

```
-B build.prod1 \
  -D SDKCONFIG_DEFAULTS="sdkconfig.defaults; \
                         sdkconfig.defaults.prod1" \
  -D SDKCONFIG=build.prod1/sdkconfig
```

###### idf.py usage

```shell
idf.py @prod/1 build flash monitor
```

</div>
</div>
</div>

---

# Sharing components between projects


<div class="grid grid-cols-2 gap-2">
<div>

When a single app can no longer support multiple products, but part of codebase is the same, the following structure can be used:


```
📂 project/
├── 📂 product1/
│   ├── 📁 main/
│   ├── 🔧 CMakeLists.txt
│   └── sdkconfig.defaults
├── 📂 product2/
│   ├── 📁 main/
│   ├── 🔧 CMakeLists.txt
│   └── sdkconfig.defaults
└── 📂 common_components/
    ├── 📁 app_storage/
    ├── 📁 app_cloud/
    └── 📁 app_ota/
```

</div>
<div>

Common components can be added:

<v-clicks>

1. Using `EXTRA_COMPONENT_DIRS` in project CMakeLists.txt:
   ```cmake
   list(APPEND EXTRA_COMPONENT_DIRS
      "${CMAKE_CURRENT_LIST_DIR}/../common_components")
   ```
2. Using `idf_component.yml` (recommended):
   ```cmake
   dependencies:
     app_storage:
       path: ../../common_components/app_storage
   ```

</v-clicks>

<div v-click>

Later, common components can be moved to other Git repositories, and included

- using `idf_component.yml` (change `path` to `git`)
- using submodules

</div>

</div>
</div>


---
layout: intro
---

# Thank you for your attention!
