# LightGBM — Claude Code Guide

LightGBM is a gradient boosting framework written in C++ with bindings for Python, R, and Java (SWIG). The repo has three main layers: a C++ core, a C API, and language-specific wrappers.

## Build

### C++ library and CLI

```sh
cmake -B build -S .
cmake --build build -j4
```

Optional cmake flags: `-DUSE_GPU=ON`, `-DUSE_CUDA=ON`, `-DUSE_MPI=ON`, `-DUSE_DEBUG=ON`, `-DUSE_OPENMP=OFF`.

### Python package (builds lib_lightgbm and installs)

```sh
sh ./build-python.sh install
```

Build only a wheel without installing:

```sh
sh ./build-python.sh bdist_wheel
```

Pass `--gpu`, `--cuda`, `--mpi`, `--no-isolation`, etc. as needed (see `build-python.sh` header for full list).

### C++ unit tests (Google Test)

```sh
cmake -B build -S . -DBUILD_CPP_TEST=ON
cmake --build build --target testlightgbm -j4
./testlightgbm
```

## Linting

```sh
pre-commit run --all-files
```

Hooks configured in `.pre-commit-config.yaml`:

| Hook | What it checks |
|---|---|
| cpplint | C++ style |
| cmakelint | CMake style (line length 120) |
| ruff-check / ruff-format | Python style/format (line length 120, numpy docstrings) |
| mypy | Python type checking (config in `python-package/pyproject.toml`) |
| biome-ci | JS/JSON (config in `biome.json`) |
| shellcheck | Shell scripts |
| yamllint | YAML files |
| typos | Spelling |
| rstcheck | reStructuredText docs |
| regenerate-parameters | Regenerates `src/io/config_auto.cpp` and `docs/Parameters.rst` from `include/LightGBM/config.h` |

## Testing

### Python tests

```sh
pytest tests/python_package_test/
```

Run a single file:

```sh
pytest tests/python_package_test/test_basic.py
```

Run a single test function:

```sh
pytest tests/python_package_test/test_basic.py::test_function_name
```

Test files: `test_basic.py`, `test_engine.py`, `test_sklearn.py`, `test_dask.py`, `test_callback.py`, `test_plotting.py`, `test_arrow.py`, `test_consistency.py`, `test_utilities.py`, `test_dual.py`.

### C++ tests

After building with `-DBUILD_CPP_TEST=ON` (see above):

```sh
./testlightgbm
```

Source files are in `tests/cpp_tests/`.

## Architecture

### Three-layer design

```
C++ core (src/, include/LightGBM/)
    ↓
C API  (include/LightGBM/c_api.h  ←→  src/c_api.cpp)
    ↓
Language bindings
  Python  (python-package/lightgbm/)
  R       (R-package/)
  Java    (swig/)
```

### Key C++ interfaces (`include/LightGBM/`)

| Header | Purpose |
|---|---|
| `config.h` | All training parameters (source of truth; auto-generates docs and C++ code) |
| `dataset.h` / `dataset_loader.h` | Dataset representation and loading |
| `boosting.h` | Boosting algorithm interface (GBDT, DART, GOSS, RF) |
| `objective_function.h` | Loss function interface |
| `metric.h` | Evaluation metric interface |
| `tree_learner.h` | Tree-building interface |
| `c_api.h` | Public C API used by all language bindings |
| `application.h` | Top-level entry point (CLI / library) |

Tree learner implementations live in `src/treelearner/`: serial, parallel (data-parallel and feature-parallel), GPU, and linear variants.

### Python package modules (`python-package/lightgbm/`)

| Module | Purpose |
|---|---|
| `basic.py` | `Dataset` and `Booster` classes; loads `lib_lightgbm` via ctypes |
| `engine.py` | `train()` and `cv()` functions |
| `sklearn.py` | scikit-learn compatible `LGBMClassifier` / `LGBMRegressor` / etc. |
| `dask.py` | Distributed training with Dask |
| `plotting.py` | `plot_importance()`, `plot_tree()`, etc. |
| `callback.py` | Built-in training callbacks |

## Code conventions

- **C++**: C++17 (`set(CMAKE_CXX_STANDARD 17)` in `CMakeLists.txt`).
- **Python**: ruff (line length 120, numpy docstring convention), mypy strict typing. Config in `python-package/pyproject.toml`.
- **JS/JSON**: biome, config in `biome.json`.
- **Parameters**: Never edit `src/io/config_auto.cpp` or `docs/Parameters.rst` by hand — they are auto-generated from `include/LightGBM/config.h` by `.ci/parameter-generator.py` (run automatically by the `regenerate-parameters` pre-commit hook).
