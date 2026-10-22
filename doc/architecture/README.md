---
package: Minuit2
summary: Vendored standalone Minuit2 minimizer from ROOT 6.22/06, renamed to the FermiMinuit2 CMake targets; used only through optimizers::NewMinuit.
layer: foundation
depends_on: []
used_by: [optimizers]
last_reviewed_commit: c84a335
---

# Minuit2 Architecture

This package is a **vendored copy of upstream code**. Upstream ROOT
documentation is authoritative for Minuit2 itself (algorithms, classes,
API). This document covers only what is specific to the Fermitools: where
the copy came from, how it is built, what was changed, and how the rest of
the Fermitools uses it. The `README.md`, `DEVELOP.md` and `doc/` files at
the package root are upstream's and are not Fermi documentation.

## Purpose and scope

**Does:** provides the Minuit2 function minimizer (MIGRAD, SIMPLEX,
MINIMIZE, HESSE, MINOS, contours, scans) in the namespace `ROOT::Minuit2`,
without needing a ROOT installation.

**Does not:** contain any Fermi-specific code. There are no Fermi
algorithm changes, wrappers or tests in this repository. The Fermi adapter
lives in `optimizers` (`optimizers::NewMinuit`).

**Provenance.** The first commit (`bb94fa6`, June 2022) is
"Minuit2_standalone generated from ROOT 6.22.06". `version_number`
contains `6.22/06`, and `StandAlone.cmake` turns this into the CMake
project version `6.22.06`. The repository is `fermi-lat/Minuit2` with
default branch `main`.

## Position in the system

| Direction | Package | Why |
|-----------|---------|-----|
| Depends on | — | No Fermi packages. It needs only a C++ compiler and the standard library. OpenMP and MPI are **optional** and **off** in the Fermitools build (see *Build and test*). |
| Used by | `optimizers` | `NewMinuit` (`optimizers/NewMinuit.h`, `src/NewMinuit.cxx`) and `StMnMinos` (`src/StMnMinos.{h,cxx}`). These are the only `#include "Minuit2/..."` uses outside this package. `optimizers` links `FermiMinuit2::FermiMinuit2` **PUBLIC**, because `NewMinuit.h` exposes Minuit2 types. |
| Indirect | `pyLikelihood`, `Likelihood` (tests) | `pyLikelihood.i` `%include`s `optimizers/NewMinuit.h`, so it compiles against Minuit2 headers, and `NewMinuit` is reachable from Python. `Likelihood/src/test/test.cxx` instantiates `NewMinuit`. Neither includes a Minuit2 header directly. |

## Key abstractions

See the upstream Minuit2 documentation. The upstream types that Fermi code
touches are listed below. They are used only by `optimizers`.

| Upstream type | Fermi use |
|---------------|-----------|
| `FCNGradientBase` | Base of `optimizers::myFCN`, which adapts an `optimizers::Statistic` and supplies an analytic gradient (`CheckGradient()` returns `false`). |
| `MnUserParameters`, `MnUserParameterState` | Built from the free `optimizers::Parameter`s, including their bounds. |
| `MnMinimize` | The minimization actually used by `NewMinuit::find_min_only`. This is MIGRAD with a SIMPLEX fallback, not plain `MnMigrad`. |
| `MnHesse`, `MnMinos`, `MnContours`, `FunctionMinimum`, `MinimumError`, `MnUserCovariance` | HESSE, MINOS errors (through the subclass `optimizers::StMnMinos`), contours, and covariance and quality reporting. |
| `MnStrategy` | Set with `NewMinuit::setStrategy` (default 1). |

## Data flow and control flow

The Minuit2 internals are covered by the upstream docs. The Fermi call path
(`optimizers/src/NewMinuit.cxx`) is:

1. `OptimizerFactory::create("NewMinuit"|"NEWMINUIT", stat)` →
   `new NewMinuit(stat)`.
2. `find_min_only` builds `MnUserParameters`. Parameter names are mangled
   to `"<index>_<name>"`, the initial step is always `1.0`, and `(0,0)`
   bounds mean "unbounded". It then runs
   `MnMinimize(m_FCN, state, m_strategy)(m_maxEval, m_tolerance)`, stores a
   heap copy of the `FunctionMinimum`, and throws
   `optimizers::Exception` if `!IsValid()`.
3. `hesse`, `Minos`, `MnContour` and the covariance accessors reuse the
   stored `FunctionMinimum`.

## Public interfaces

- **C++ headers:** `inc/Minuit2/*.h` (namespace `ROOT::Minuit2`), plus the
  `ROOT::Math`/`ROOT::Fit` subset in `inc/Math/` and `inc/Fit/`. These
  are installed under `include/fermi/Minuit2/{Minuit2,Math,Fit}` and
  included as `"Minuit2/MnMigrad.h"`.
- **Libraries:** `FermiMinuit2` (the minimizer), `FermiMinuit2Math` (the
  standalone mathcore subset, built with `MATHCORE_STANDALONE` and
  `ROOT_Math_VecTypes`), and `FermiMinuit2Common` (an INTERFACE target
  that carries the OpenMP/MPI settings). They are static in the Fermitools
  build, because `add_library` is given no type and `BUILD_SHARED_LIBS` is
  not set.
- **CMake package:** installs `lib/cmake/FermiMinuit2/FermiMinuit2Config.cmake`
  and `FermiMinuit2Targets.cmake` (namespace `FermiMinuit2::`).
- **Applications, Python, file formats:** none.

## Build and test

- **Integration.** `Minuit2` is listed in `FERMI_COMPONENTS` in the top-level
  `CMakeLists.txt` and pulled in with `add_subdirectory`. The top-level
  project is `fermitools`, not `ROOT`, so `CMakeLists.txt` declares
  `project(FermiMinuit2)` and includes `StandAlone.cmake`. That file
  creates `FermiMinuit2Common`, then `src/` (`FermiMinuit2`) and
  `src/math/` (`FermiMinuit2Math`). Aliases: `FermiMinuit2::FermiMinuit2`,
  `FermiMinuit2::Math` and `FermiMinuit2::Common`. `optimizers` links
  `FermiMinuit2::FermiMinuit2`.
- **Options.** `minuit2_omp` (adds `OpenMP::OpenMP_CXX` to
  `FermiMinuit2Common`) and `minuit2_mpi` (adds `MPIPROC` and
  `MPI::MPI_CXX`) default to **OFF**. Nothing in ScienceTools
  (`CMakeLists.txt`, `cmake/`, `recipe/`) turns them on. Upstream comments
  say MPI uses the deprecated C++ bindings, and that both modes are for
  thread-safe FCNs only. Fermi FCNs (`myFCN` → `Statistic`) have not been
  checked for thread safety.
- **Copying from ROOT.** `minuit2_inroot` and `minuit2_standalone` (from
  `copy_standalone.cmake`) only apply when the directory sits inside a ROOT
  source tree. Here every file is already present, so nothing is copied.
- **Tests.** None run. `StandAlone.cmake` never adds `test/` or `examples/`.
  `test/CMakeLists.txt` uses ROOT macros (`ROOT_EXECUTABLE`,
  `ROOT_ADD_TEST`) and needs `find_package(ROOT)`. Minuit2 is exercised
  only indirectly, by `test_optimizers` (`optimizers/src/test/main.cxx`)
  and the Likelihood tests.

## Invariants and gotchas

**Local modifications vs upstream 6.22/06.** `git diff bb94fa6 HEAD`
touches only `CMakeLists.txt`, `StandAlone.cmake`, `src/CMakeLists.txt`,
`src/math/CMakeLists.txt` and `Minuit2Config.cmake.in`. **No C++ source
or header differs from upstream.** The changes are:
- Renamed the project and targets `Minuit2*` → `FermiMinuit2*` (commit
  `7c3a6b5`, "to avoid root collision") and the CMake namespace
  `Minuit2::` → `FermiMinuit2::`.
- Moved the install include directory to `include/fermi/Minuit2`.
- Removed the in-ROOT `ROOT_STANDARD_LIBRARY_PACKAGE(...)` block from
  `CMakeLists.txt`.
- Raised `cmake_minimum_required` from 3.1 to 3.5 to 3.10.

**The rename covers library and target names only, not symbols.** The C++
namespace is still `ROOT::Minuit2` and headers are still `"Minuit2/..."`.
If a build also links ROOT's own `libMinuit2` (possible only with
`FERMI_BUILD_ROOT=ON`, which affects `tip`/`celestialSources`), the
symbols would be duplicated. This is inferred from reading the code and
was not checked at runtime.

**Stale names in the installed CMake config.** `Minuit2Config.cmake.in`
defines `FermiMinuit2::Math` → `FermiMinuit2::Minuit2Math` and
`FermiMinuit2::Common` → `FermiMinuit2::Minuit2Common`. The exported
targets are actually `FermiMinuit2::FermiMinuit2Math` and
`FermiMinuit2::FermiMinuit2Common`, so an external
`find_package(FermiMinuit2)` consumer that uses `::Math` or `::Common`
would fail. This does not affect the in-tree build, which uses aliases
(from reading the code; not checked at runtime). `examples/simple` still
calls `find_package(Minuit2)` and links `Minuit2::Minuit2`.

**`export(PACKAGE FermiMinuit2)`** in `StandAlone.cmake`, together with
`cmake_policy(VERSION 3.12)`, may register the build tree in the user's
CMake package registry (`~/.cmake/packages`), because policy CMP0090 is
unset. This is inferred and was not checked.

**NewMinuit tolerance scaling.** `NewMinuit::setTolerance` passes
`1000 * tol / Up` to Minuit2. Upstream then targets
`EDM < 0.002 * toler * Up` (`ModularFunctionMinimizer.cxx` scales by `Up`,
and the 0.002 factor is in the builder). So a Fermi `tol` corresponds to
about `EDM < 2 * tol`. `myFCN::Up()` defaults to 0.5. `Minos` and
`MnContour` temporarily set it to `level/2` and reset it to 0.5
afterwards.

## Extension points

- **Upgrading Minuit2.** Regenerate a standalone tree from a newer ROOT
  (see upstream `DEVELOP.md`: `minuit2_standalone=ON` inside a ROOT
  checkout), then reapply the `FermiMinuit2` renames and the
  `include/fermi/Minuit2` install path. The diff of the build files listed
  above is the full set of changes to reapply. Then rebuild `optimizers`
  and run `test_optimizers` and the Likelihood tests.
- **Using Minuit2 features from Fermi code.** Add them to
  `optimizers::NewMinuit` rather than including Minuit2 directly from
  other packages.

## Further reading

- Upstream Minuit2 documentation (ROOT reference guide, "Minuit2");
  `README.md`, `DEVELOP.md` and `doc/Minuit2.md` in this package.
- `src/optimizers/doc/architecture/` for `NewMinuit` and the `Optimizer`
  abstraction.
- `docs/architecture/externals.md` in ScienceTools (integration of
  third-party code).
