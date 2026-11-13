# EnumCrafter

**Automatic C++ enum reflection through code generation.**

EnumCrafter scans C++ source files, extracts enum metadata, and generates a C++ header providing enum-to-string conversion, string-to-enum parsing, and enum iteration.

## Why EnumCrafter?

C++ does not provide built-in enum reflection. Converting enum values to strings, parsing strings back into enum values, or iterating over all enum members usually requires manually maintained code.

This becomes particularly inconvenient in projects with many enums:

- Every new enumerator requires updating conversion functions and lookup tables.
- Manually maintained mappings can become inconsistent with the actual enum declarations.
- Repeated switch statements and lookup tables introduce unnecessary boilerplate.
- Compiler-specific reflection techniques are not universally available.

Libraries such as `magic_enum` offer a convenient alternative, but compiler compatibility, implementation limitations, and toolchain restrictions can make them unsuitable for some projects, particularly embedded development environments.

EnumCrafter takes a different approach: it uses Python and Clang to inspect C++ source code **before compilation**, then generates ordinary C++ code containing explicit enum mappings.

The generated header does not depend on Python or Clang at runtime.

## Features

- Automatically discovers enum declarations in configured C++ source files.
- Generates enum-to-string conversion functions.
- Generates string-to-enum parsing functions.
- Generates iterable arrays containing all enumerators.
- Supports ONLY scoped enumerations (`enum class`).
- Handles enums declared globally, inside namespaces, and inside classes or structs.
- Extracts enum names, qualified names, source locations, and integer values.
- Exports extracted metadata as JSON.
- Automatically includes the source headers containing discovered enums.
- Supports configurable output directories and C++ namespaces.
- Processes source files in parallel.

## How it works

```text
       C++ source files
              |
              v
       Python + Clang
              |
              v
       Enum metadata
              |
              +------------------+
              |                  |
              v                  v
        JSON metadata       Code generator
                                 |
                                 v
                         EnumCrafter.hpp
                                 |
                                 v
                         Your C++ project
```

EnumCrafter performs the discovery and code generation during development or as part of your build workflow. Your application then uses the generated header like any other C++ header.

## Requirements

- Python 3.9 or newer.
- Clang / libclang and the Python bindings provided through the project dependencies.
- Dependencies listed in `requirements.txt`.
- A C++17-compatible compiler and standard library for the generated header.

**Why C++17?**

The current generated header uses:

- `std::string_view`
- `std::optional`
- Variable templates

These features require C++17.

Although EnumCrafter uses code generation to avoid relying on compiler reflection, the current generated output is not compatible with pre-C++17 compilers.

## Installation

Clone the repository and install its Python dependencies:

```bash
git clone https://github.com/mahyar1284/EnumCrafter.git
cd EnumCrafter

python -m pip install -r requirements.txt
```

## Configuration

Create a `config.json` file in your consuming project. You can use `config.example.json` as a starting point.

```json
{
    "sources": ["src/"],
    "include_dirs": [],
    "output": "generated",
    "namespace": "EnumCrafter"
}
```

### Configuration options

| Option | Description |
|---|---|
| `sources` | C++ files or directories to scan. Directories are scanned recursively. |
| `include_dirs` | Additional include directories passed to Clang. |
| `output` | Directory where generated files are written. |
| `namespace` | Namespace containing the generated utilities. Defaults to `EnumCrafter`. |

Paths are interpreted relative to the current working directory.

The scanner searches for `.cpp`, `.cc`, `.cxx`, `.hpp`, `.h`, and `.hxx` files.

## Running EnumCrafter

From the consuming project root, assuming EnumCrafter is located at `tools/EnumCrafter/` and the configuration file is `config.json`:

Generate metadata and the C++ header:

```bash
python tools/EnumCrafter/main.py
```

Use a different configuration file:

```bash
python tools/EnumCrafter/main.py path/to/config.json
```

Generate only the JSON metadata without generating the C++ header:

```bash
python tools/EnumCrafter/main.py --metadata-only
```

Or specify a configuration file:

```bash
python tools/EnumCrafter/main.py --metadata-only path/to/config.json
```

If running directly from the cloned EnumCrafter repository, use `python main.py` instead.

## Generated files

By default, EnumCrafter generates:

```text
generated/
    enum_metadata.json
    EnumCrafter.hpp
```

### enum_metadata.json

Contains the extracted metadata for discovered enums, including:

- Fully qualified enum name.
- Source file.
- Declaration line number.
- Whether the enum is scoped.
- Enumerator names and their integer values.

### EnumCrafter.hpp

Contains the generated C++ utilities.

The header automatically includes the source headers containing the discovered enums and provides the following interfaces:

| Utility | Purpose |
|---|---|
| `toString(EnumType)` | Converts an enum value into its enumerator name. |
| `parse<EnumType>(string)` | Parses a string into an enum value. |
| `makeIterable<EnumType>` | Provides a constexpr array containing all enumerators. |

**Do not edit `EnumCrafter.hpp` manually.** It is generated automatically and your changes will be overwritten the next time EnumCrafter runs.

## Usage

Assume the following enum declaration:

```cpp
namespace Application {

    enum class DeviceState {
        Idle,
        Running,
        Error,
        Uninitialized
    };

}
```

Include the generated header:

```cpp
#include "generated/EnumCrafter.hpp"
```

### 1. Enum to string

Use `toString()` to convert an enum value into its corresponding enumerator name:

```cpp
auto name = EnumCrafter::toString(
    Application::DeviceState::Running
);
```

Result:

```text
Running
```

The generated function uses a `switch` statement and returns `std::string_view`.

If the supplied enum value does not match any declared enumerator, it returns `"???"`.

### 2. String to enum

Use `parse<EnumType>()` to convert an enumerator name back into its enum value:

```cpp
auto state =
    EnumCrafter::parse<Application::DeviceState>("Error");

if (state.has_value()) {
    Application::DeviceState currentState = state.value();
}
```

The return type is `std::optional<EnumType>`.

If the string does not match any enumerator, the function returns `std::nullopt`.

String matching is case-sensitive and uses the exact enumerator names.

### 3. Iterate over enum values

Use `makeIterable<EnumType>` to access all declared enumerators:

```cpp
for (auto state :
     EnumCrafter::makeIterable<Application::DeviceState>) {

    // Process each enum value
}
```

The generated array is a `constexpr std::array` containing the enum values in declaration order.

## Supported enum declarations

EnumCrafter currently generates reflection utilities for **scoped enumerations (`enum class`)**.

For example:

```cpp
enum class DeviceState {
    Idle,
    Running,
    Error
};
```

It supports scoped enums declared:

```cpp
// Global scope
enum class StatusCode {
    SUCCESS,
    NOT_FOUND,
    ERROR
};

// Namespace scope
namespace Application {

    enum class DeviceType {
        Sensor,
        Actuator
    };

}

// Class or struct scope
class Device {

public:
    enum class State {
        Disconnected,
        Connected
    };

};
```

Traditional unscoped enumerations (`enum`) can be discovered in the metadata, but the current code generator does not generate reflection utilities for them.

## Limitations

- Only scoped enumerations (`enum class`) receive generated reflection utilities.
- Enum declarations must be discoverable by Clang.
- Required include directories must be supplied through `include_dirs`.
- The current generated header requires C++17.
- String-to-enum conversion is case-sensitive.
- Unknown enum values return `"???"` from `toString()`.
- Invalid strings return `std::nullopt` from `parse()`.

## Project structure

```text
EnumCrafter/
│
├── main.py
├── metadata_gen.py
├── code_gen.py
├── config.example.json
├── requirements.txt
└── README.md
```

## License

See the repository for license information.
