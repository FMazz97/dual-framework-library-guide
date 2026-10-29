# Appendix A — IDF Dependency Resolution

ESP-IDF dependency management is often presented as a choice between plain CMake and the Component Manager. In practice they are not alternatives: both run inside the same CMake configuration step and feed the same build system. What differs is **who tells the build system which components exist**, and how it learns what each one requires. Understanding this separation is essential to avoid build errors and ensure correct project configuration.

## Terminology

Everything in ESP-IDF's build revolves around components, so it is worth fixing the vocabulary first:

| Term | In ESP-IDF |
| :--- | :--- |
| **Component** | a self-contained piece of code that offers functionality and can be added to an application. Technically, any directory in `COMPONENT_DIRS` that contains a `CMakeLists.txt`, compiled into a static library and linked into the app. Libraries, drivers, and `main` are all components |
| **Project** | what is built into an application: a directory with the top-level CMake `project()`, which gathers the components it uses. A project is not a component itself |
| **Example** | a self-contained project that depends on a component in order to demonstrate it |
| **Package** | what the ESP Component Registry publishes: an archive of a component |

---

## Baseline ESP-IDF project structure

Regardless of how dependencies are resolved, every **ESP-IDF standalone project** needs a minimal set of **CMake** files:

- A root `CMakeLists.txt` defining the project name and importing ESP-IDF build logic:

  ```cmake
  cmake_minimum_required(VERSION 3.16)
  include($ENV{IDF_PATH}/tools/cmake/project.cmake)
  project(my_example)
  ```

- A `main/` directory, which is a component in its own right and a special one: it holds the project's entry point (e.g. `main.cpp`), is added to the build automatically, and automatically requires every other component. Like any component, it has its own `CMakeLists.txt`:

  ```cmake
  idf_component_register(
    SRCS "main.cpp"
    INCLUDE_DIRS "."
  )
  ```

The two workflows below only differ in **how the library component reaches the build system**.

---

## The big picture: three roles in one configuration step

| Role | Responsibility | How |
| :--- | :--- | :--- |
| **Discovery (CMake)** | decides which directories are components | scans `COMPONENT_DIRS`: `IDF_PATH/components`, the project's `components/` folder, and `EXTRA_COMPONENT_DIRS` |
| **Component Manager** | adds more components to that list | runs inside CMake's configuration step: reads manifests, solves dependencies, and downloads them into `managed_components`, which CMake then scans (local dependencies are used in place) |
| **Requirements (CMake)** | connects components to each other | `REQUIRES` and `PRIV_REQUIRES` define include paths and linking between components already known to the build; in a project they are typically declared by `main`, the component that uses the library |

> [!IMPORTANT]
> A component is named as its folder. This is the name the build system knows, in both workflows described below.

Whichever way a directory is found, it is presented to the build system together with its source. That source sets an **override order**, which can also be used on purpose: when two components share the same name, the one higher in this list replaces the others.

1. **Project components:** the `components/` folder inside the project (highest priority)
2. **`EXTRA_COMPONENT_DIRS`:** directories registered in the project's root `CMakeLists.txt`
3. **Managed components:** dependencies declared in `idf_component.yml` are resolved by the Component Manager and downloaded into `managed_components`, unless the manager is disabled
4. **ESP-IDF components:** the built-in ones under `IDF_PATH/components` (lowest priority)

For example, copying an ESP-IDF component into the project's `components/` folder and modifying it there overrides the original, which stays untouched.

---

## Pure CMake workflow: discovery through `EXTRA_COMPONENT_DIRS`

In a **pure CMake** workflow, component discovery is entirely filesystem-driven: no manifest is needed, because the Component Manager has nothing to resolve. A component placed in the project's `components/` folder is found automatically; any other location is registered through `EXTRA_COMPONENT_DIRS`.

- Register the required component's path in the project's root `CMakeLists.txt`:

  ```cmake
  cmake_minimum_required(VERSION 3.16)
  list(APPEND EXTRA_COMPONENT_DIRS "${CMAKE_CURRENT_LIST_DIR}/../path/to/<component>")
  include($ENV{IDF_PATH}/tools/cmake/project.cmake)
  project(my_example)
  ```

  Each registered directory can be a component itself (it contains a `CMakeLists.txt`) or a parent whose subdirectories are components.

> [!NOTE]
> `EXTRA_COMPONENT_DIRS` must be set after `cmake_minimum_required()` and **before** `include(.../project.cmake)`. Prefer `list(APPEND EXTRA_COMPONENT_DIRS ...)` with `${CMAKE_CURRENT_LIST_DIR}` over `set(...)`: it avoids discarding existing component paths and gives robust path resolution.

- Declare the dependency in the `CMakeLists.txt` of the component that uses it (here `main`) to make public headers and symbols accessible:

  ```cmake
  idf_component_register(
    SRCS "main.cpp"
    INCLUDE_DIRS "."
    REQUIRES <component>
  )
  ```

  The value passed to `REQUIRES` must match the exact **folder name** of the component. In `main` it is optional, since `main` automatically requires every component in the build, but keeping it explicit documents the dependency; in any other component it is needed.

The resolution chain for a pure CMake setup is:

```text
Project
│
├── CMakeLists.txt (list(APPEND EXTRA_COMPONENT_DIRS ...))
│         ↓
│   build system component paths
│   (ESP-IDF components + project + ../path/to/foo/)
│         ↓  then, still in the configuration step
│   ../path/to/foo/CMakeLists.txt → idf_component_register(...)
│
└── main/
    └── CMakeLists.txt (idf_component_register(... REQUIRES foo))
```

> [!TIP]
> To verify that a setup really is pure CMake, build with the `IDF_COMPONENT_MANAGER` environment variable set to `0`. In tests this made no difference for a component reached through `EXTRA_COMPONENT_DIRS`, with or without an `idf_component.yml`.

For more details, see [Build System](https://docs.espressif.com/projects/esp-idf/en/stable/esp32/api-guides/build-system.html) at [ESP-IDF Programming Guide](https://docs.espressif.com/projects/esp-idf/en/stable/esp32/index.html).

---

## Component Manager workflow: manifest-based resolution

The flow changes when a dependency is declared in `idf_component.yml` manifest:

```yaml
dependencies:
  foo:
    version: "*"
    override_path: "../path/to/foo"
```

Here the component is not registered by the root `CMakeLists.txt`, but resolved by the **Component Manager**. It is not a separate build step: it is enabled by default and runs automatically inside CMake's configuration step. When CMake configures the project, the Component Manager:

- processes the `idf_component.yml` manifests of the project's components (here `main/idf_component.yml`) and recursively solves their dependencies;
- downloads registry and Git dependencies into `managed_components` (as `<namespace>__<name>` folders);
- hands the resolved component folders over to the build system.

A local dependency like the one above is not downloaded: its folder path is passed directly to the build system, together with the base ESP-IDF components and the project's own components.

A Registry dependency enters the build under its folder name, `<namespace>__<name>`. In tests, a `REQUIRES "<name>"` in the `CMakeLists.txt` of the component that uses it is accepted, while `REQUIRES "<namespace>__<name>"` is not resolved and produces an error.

The Component Manager performs no name validation here: the component name still comes from the folder name, which for a local dependency must match the name declared in the manifest. Naming rules are only enforced at publication, by `compote component pack` and `compote component upload` (see [Chapter 1](../chapters/01-ecosystem-constraints.md#registry-naming-rules)).

```text
Project
│
├── CMakeLists.txt (standard / minimal)
│
└── main/
    └── idf_component.yml (override_path: "../path/to/foo")
              ↓  Component Manager (inside CMake's configuration step)
        build system component paths
        (ESP-IDF components + project + ../path/to/foo/)
              ↓  then, still in the configuration step
        ../path/to/foo/CMakeLists.txt → idf_component_register(...)
```

From the component paths step onwards, the component is processed exactly as in the pure CMake workflow.

### What the Component Manager adds to the build system

The Component Manager does not modify the project's CMake files. It only gives the build system what it needs to run CMake correctly:

- **Component paths:** the folders of the resolved dependencies, presented as *managed components* (the third level of the override order).
- **Remote dependencies:** only the Component Manager can resolve dependencies from the online ESP Component Registry (and from Git), downloading them into `managed_components`. A pure CMake setup can only use components that are already on the filesystem.
- **Requirements:** the dependencies declared in `idf_component.yml`, added to the component's requirements (as `MANAGED_REQUIRES` and `MANAGED_PRIV_REQUIRES`, merged into `REQUIRES` and `PRIV_REQUIRES`), so they don't need to be repeated in `CMakeLists.txt`.
- **Metadata:** the component version (`COMPONENT_VERSION`).

> [!NOTE]
> The Component Manager can be disabled by setting the `IDF_COMPONENT_MANAGER` environment variable to `0`. ESP-IDF then warns that an `idf_component.yml` was found but the manager is not enabled, and the dependency is simply never added: the failure surfaces later, at compile time (e.g. `fatal error: foo.h: No such file or directory`), not as a resolution error.

For more details, see [IDF Component Manager](https://docs.espressif.com/projects/esp-idf/en/stable/esp32/api-guides/tools/idf-component-manager.html), ["What is the ESP Component Registry?"](https://developer.espressif.com/blog/2024/10/what-is-the-esp-registry/) and [IDF Component Manager and ESP Component Registry Documentation](https://docs.espressif.com/projects/idf-component-manager/en/latest/).

---

## Pure CMake vs Component Manager

| | **Pure CMake** | **Component Manager** |
| :--- | :--- | :--- |
| **Declared in** | the root `CMakeLists.txt` (`EXTRA_COMPONENT_DIRS`), or by placing the component in `components/` | the `idf_component.yml` of the consuming component |
| **Component found by** | filesystem scan of the registered directories | dependency resolution (local path, registry, Git) |
| **Name** | the folder name | `<namespace>__<name>` for a Registry dependency; for a local dependency it must match the declared component name (rules enforced only at publication) |
| **Requirements** | explicit `REQUIRES` / `PRIV_REQUIRES` (optional in `main`) | added to the build system automatically from the manifest dependencies |
| **Override level** | project components, or `EXTRA_COMPONENT_DIRS` | managed components (lower than both) |
| **Side effects** | none | `managed_components` for downloaded dependencies |

---

## The common ground

Both workflows end up in the same place: the component's own `CMakeLists.txt`. There, `idf_component_register(...)` is how a component (seen as a library) exposes its source files, public include directories, and requirements to the build system:

```cmake
idf_component_register(...)
```

How the component got there, through `EXTRA_COMPONENT_DIRS`, the `components/` folder, or a manifest dependency, makes no difference to how it is built.

When PlatformIO is introduced, a third actor enters the picture: see [4.4 PlatformIO, Component Manager, and CMake are independent dependency-resolution systems](../chapters/04-examples.md#44-platformio-component-manager-and-cmake-are-independent-dependency-resolution-systems) for the combinations that work and the ones that don't.

---

<!-- markdownlint-disable MD033 -->
<div align="center">

[Introduction](../README.md) — [Appendix B - Yield & Watchdog ➡️](B-yield-watchdog.md)

</div>
<!-- markdownlint-enable MD033 -->
