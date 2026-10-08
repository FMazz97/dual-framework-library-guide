
# ESP-IDF dependency resolution: CMake and Component Manager are different layers

## Baseline ESP-IDF Project Structure

Regardless of how dependencies are resolved, every **ESP-IDF standalone project** needs a minimal set of **CMake** files:

- A root `CMakeLists.txt` defining the project name and importing ESP-IDF build logic:

  ```cmake
  cmake_minimum_required(VERSION 3.16)
  include($ENV{IDF_PATH}/tools/cmake/project.cmake)
  project(my_example)
  ```

- A `main/` directory containing the entry point (e.g., `main.cpp`) and its own `main/CMakeLists.txt`:

  ```cmake
  idf_component_register(
    SRCS "main.cpp"
    INCLUDE_DIRS "."
  )
  ```

The choice between pure *CMake* and the *Component Manager* simply dictates **how** the project links to your library component.

---

## Component resolution through `EXTRA_COMPONENT_DIRS`

In a pure **CMake** workflow, component discovery is entirely filesystem-driven.

- Register the requred component's path in the project's root `CMakeLists.txt`:

  ```cmake
  list(APPEND EXTRA_COMPONENT_DIRS "${CMAKE_CURRENT_LIST_DIR}/../path/to/<component>")
  # set(EXTRA_COMPONENT_DIRS "../path/to/<component>")
  ```

  ESP-IDF scans the specified directories, finds each component's `CMakeLists.txt`, and registers the component under the exact **name** of the folder that contains it.

  > Prefer `list(APPEND EXTRA_COMPONENT_DIRS)` with `${CMAKE_CURRENT_LIST_DIR}` over `set(EXTRA_COMPONENT_DIRS)` to avoid overwriting existing paths and to ensure robust, absolute path resolution.

- Declare the dependency in `main/CMakeLists.txt` to make public headers and symbols accessible:

  ```cmake
  idf_component_register(
    SRCS "main.cpp"
    INCLUDE_DIRS "."
    REQUIRES <component>
  )
  ```

  Notice that the value passed to `REQUIRES` must match the exact **folder name** of the component registered in the previous step.

The complete resolution chain for a pure **CMake** setup is therefore structured as follows:

```text
Project
│
├── CMakeLists.txt (list(APPEND EXTRA_COMPONENT_DIRS ...))
│         ↓
│   components/foo/
│         ↓
│   CMakeLists.txt → idf_component_register(...)
│
└── main/
    └── CMakeLists.txt (idf_component_register(... REQUIRES foo))
```

In this workflow, component's `idf_component.yml` is completely ignored. The component builds successfully even if no manifest exists, because **discovery and naming are handled entirely by CMake based on the filesystem structure and explicit `REQUIRES` statements.**

---

## Component resolution through `idf_component.yml`

The flow changes when the project uses the **Component Manager**:

```yaml
dependencies:
  foo:
    version: "*"
    override_path: "../components/foo"
```

Here, the first actor is no longer CMake, but the **Espressif Component Manager**, while the `CMakeLists.txt` files remain minimal and unchanged.

The Component Manager performs the following sequential steps:

- reads the project's `idf_component.yml`;
- locates the local dependency through `override_path`;
- loads the component's own `idf_component.yml`;
- validates the component's `name:` field (enforcing strict lowercase validation rules);
- injects validated component into the **CMake** build system environment.

Only after this validation sequence is successful does CMake take over.

```text
Project
│
├── CMakeLists.txt (standard / minimal)
│
└── main/
    │
    └── idf_component.yml (override_path: "../path/to/foo")
              ↓
        components/foo/
              ↓
        idf_component.yml (manifest name validation)
              ↓
        CMakeLists.txt → idf_component_register(...)
```

This explains why naming rules become strict when using the **Component Manager**. Unlike the pure `EXTRA_COMPONENT_DIRS` workflow, the component must first pass manifest identity validation before CMake ever processes its directory.

---

## The common ground

Both workflows ultimately converge at the same endpoint in the component `CMakeLists.txt`:

```cmake
idf_component_register(...)
```

This macro remains the **sole mechanism that tells ESP-IDF which source files, include directories, and compile-time requirements belong to the component.** The structural difference lies entirely in how the component is discovered, named, and validated before it reaches the compiler.
