# EnumCrafter

EnumCrafter scans C++ sources, writes enum metadata as JSON, and generates a
C++ reflection header.

## Install

Use Python 3.9 or newer and install the dependency:

```bash
python -m pip install -r requirements.txt
```

## Configure

Copy `config.example.json` into the consuming project and update:

- `sources`: C++ files or directories to scan.
- `include_dirs`: additional include directories for libclang.
- `output`: directory for generated files. The tool writes `enum_metadata.json`
  and `EnumCrafter.hpp` there.
- `namespace`: C++ namespace for the generated functions (defaults to
  `EnumCrafter`).

Relative paths are interpreted from the current working directory, so run the
tool from the consuming project root or pass an appropriate config path.

## Run

From the consuming project root, with the config at `config.json`:

```bash
python tools/EnumCrafter/main.py
```

Pass another config path as a positional argument, or generate only the JSON:

```bash
python tools/EnumCrafter/main.py path/to/config.json
python tools/EnumCrafter/main.py --metadata-only path/to/config.json
```
