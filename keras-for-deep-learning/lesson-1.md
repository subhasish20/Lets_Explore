# Chapter 1 — Keras Repository Architecture and Entry Points

## A. Chapter Objectives

By the end of this chapter you will be able to:

1. Describe the top-level directory layout of the Keras repository and the *responsibility* of each top-level folder.
2. Explain the two-layer structure **`keras/src/*` (implementation)** vs **`keras/api/*` (public surface)**, and why Keras 3 introduced that split.
3. Trace the import chain from `import keras` down to the module that actually defines a symbol like `keras.layers.Dense`.
4. Know which subsystems are *backends* (`keras/src/backend/`) and how the rest of the tree depends on the backend via a single abstraction seam.
5. Navigate the repo with `grep`/`git grep` to find where a public symbol is defined versus merely re-exported.

You will not yet read the body of `Layer` or `Model` — that starts in Chapters 4 and 7. This chapter is about the *map*, not the *territory*.

## B. Where This Fits in Keras

Every later chapter is "open file X, read class Y." If you don't know that public `keras.layers.Dense` and implementation `keras/src/layers/core/dense.py` are two different objects connected by a re-export, you will constantly look in the wrong file. Chapter 1 builds that mental model once and it pays off for the next 29 chapters.

The two concepts from this chapter that recur everywhere:

- **The API/implementation seam.** Public names live in one place; real code lives in another. Every chapter must distinguish them.
- **The backend seam.** All tensor math flows through `keras.src.backend`. There is no direct `import tensorflow as tf` in layer code. This is the single most important architectural decision in Keras 3.

## C. Actual Source Files

**Expected layout — verify each path with `ls` or `git ls-files` before trusting it.**

Top level of the repository:

| Path | Responsibility |
|---|---|
| `keras/` | The importable Python package. |
| `keras/api/` | Hand-maintained public API re-export tree (Keras 3). Every `from keras.src... import X` that makes `keras.X` work is here. |
| `keras/src/` | All implementation. Nothing here is "public" in the strict sense — it's all reachable but not the documented surface. |
| `keras/_tf_keras/` | The `tf.keras` compatibility shim (`tensorflow.keras` maps here when TF uses Keras 3). |
| `guides/` | Long-form user documentation (Markdown). Not executed by tests. |
| `examples/` | Runnable end-to-end examples. |
| `benchmarks/` | Performance microbenchmarks. |
| `shell/` | Utility scripts (formatting, API generation, Docker). |
| `pyproject.toml` / `setup.py` | Packaging, dependencies, entry points. |
| `requirements*.txt` | Pinned dependency sets (common, TF, JAX, torch, etc.). |
| `CONTRIBUTING.md` | The rules a PR must follow — required reading for Chapter 28–30. |
| `README.md` | High-level orientation and installation matrix. |

Inside `keras/src/` (the implementation tree):

| Path | Responsibility |
|---|---|
| `keras/src/backend/` | Backend abstraction. Contains `common/`, `tensorflow/`, `jax/`, `torch/`, `numpy/`, and the switcher `__init__.py`. |
| `keras/src/layers/` | All layer classes: `core/`, `convolutional/`, `normalization/`, `attention/`, `rnn/`, `preprocessing/`, etc. |
| `keras/src/models/` | `Model`, `Sequential`, and the Functional model. |
| `keras/src/ops/` | Stateless tensor operations (`numpy`, `nn`, `math`, `image`, `linalg`, `core`). Thin wrappers over `backend`. |
| `keras/src/trainers/` | `fit`/`predict`/`evaluate` loops and the `Trainer` mixin. |
| `keras/src/optimizers/` | Optimizer implementations plus `base_optimizer.py`. |
| `keras/src/losses/` | Loss functions and `Loss` base class. |
| `keras/src/metrics/` | Metric classes and `Metric` base class. |
| `keras/src/callbacks/` | Callback base + built-ins (early stopping, checkpointing, TensorBoard, …). |
| `keras/src/saving/` | `.keras` (zip) format, `saving_api`, serialization, deserialization. |
| `keras/src/initializers/`, `regularizers/`, `constraints/` | Layer building blocks. |
| `keras/src/random/` | `SeedGenerator` and RNG utilities. |
| `keras/src/dtype_policies/` | `DTypePolicy`, mixed precision policy objects. |
| `keras/src/distribution/` | Data-parallel / model-parallel distribution APIs. |
| `keras/src/export/` | SavedModel / TF Lite / ONNX export. |
| `keras/src/utils/` | `naming.py`, `tracking.py`, `python_utils.py`, `file_utils.py`, and dozens more. |
| `keras/src/legacy/` | Backwards-compat shims. |
| `keras/src/applications/`, `datasets/` | Pretrained models and small datasets. |

**How to verify the tree from your clone:**

```bash
cd ~/projects/keras
# Top-level listing
ls

# The implementation tree, one level deep
ls keras/src

# The public API tree (Keras 3)
ls keras/api 2>/dev/null || echo "no keras/api — likely Keras 2.x layout"

# Count modules per subsystem to get a feel for size
for d in keras/src/*/; do printf '%6d  %s\n' "$(git ls-files "$d" | wc -l)" "$d"; done | sort -rn
```

The last command is genuinely informative — the number of files per subsystem tells you where the "weight" of Keras lives. In a typical Keras 3 revision, `layers/`, `backend/`, and `ops/` are the three heaviest subtrees.

## D. Read the Code in the Correct Order

Read in this order, in your editor and at the shell. Do not skip step 2.

**1. The package entry point.**

```bash
less keras/__init__.py
```

**Expected — verify:** In Keras 3 this file is short and its only job is to pull the public surface into the `keras` namespace. You'll likely see something like:

```python
from keras import api as _api
from keras.api import *  # noqa
```

or a set of `from keras.api... import ...` lines. The exact text depends on the revision. The important observation: **`keras/__init__.py` does not define layers, models, or ops** — it imports them from `keras/api/`.

**2. The public API tree.**

```bash
less keras/api/__init__.py
find keras/api -maxdepth 2 -type f -name '*.py' | sort | head -40
```

`keras/api/` is generated and hand-maintained. Each subfolder (`keras/api/layers/`, `keras/api/models/`, …) mirrors the *public* namespaces you use at the REPL. Inside each, the file typically contains one line per symbol:

```python
from keras.src.layers.core.dense import Dense
```

That single line is the seam. `keras.layers.Dense` (public) → `keras.src.layers.core.dense.Dense` (implementation). **This is the fact Chapter 1 exists to teach.**

**3. Trace one concrete symbol end-to-end.**

```bash
# Where is Dense re-exported?
git grep -n "class Dense" -- keras/src

# Where is it bound to the public name?
git grep -n "import Dense" -- keras/api

# Confirm the runtime identity (needs an importable keras)
python -c "import keras, keras.src.layers.core.dense as d; print(keras.layers.Dense is d.Dense)"
```

The last line should print `True`. If it prints `False`, the public symbol and the class you found are not the same object — usually a legacy shim or a version skew. That's worth investigating on the spot.

**4. The backend seam.**

```bash
less keras/src/backend/__init__.py
ls keras/src/backend
```

**Expected — verify:** You will see per-backend directories and a top-level switcher. In Keras 3, `keras/src/backend/__init__.py` reads an environment variable (typically `KERAS_BACKEND`) and imports symbols from the chosen backend package into the local namespace. Everything downstream then does `from keras.src import backend` and calls `backend.numpy.matmul(...)`, never `tf.matmul`.

**5. The ops layer (thin on top of backend).**

```bash
less keras/src/ops/__init__.py
head -60 keras/src/ops/numpy.py
```

Ops are what layers actually call. `keras.ops.matmul` dispatches to `backend.numpy.matmul`, which dispatches to the active backend's implementation. Chapter 15–18 will make this concrete; for now, just notice that ops are the *public, backend-agnostic* math surface.

**6. The trainers seam.**

```bash
ls keras/src/trainers
head -40 keras/src/trainers/trainer.py 2>/dev/null || head -40 keras/src/trainers/trainer.py
```

**Expected — verify:** `Trainer` is the mixin that provides `fit`, `evaluate`, and `predict` to `Model`. We will read it in Chapter 11. For now, note only that even the training loop calls into `backend`, never into a specific framework directly.

## E. Explain the Implementation

**Concept 1 — the API/implementation split.**

Why two trees? Because Keras 3 supports multiple backends *and* a stable public API. If `Layer`, `Dense`, and `Sequential` lived directly in `keras/`, then every public symbol would also be an implementation symbol, and renaming an internal module would break users. By putting the public surface in `keras/api/` and the code in `keras/src/`, Keras gets:

- A single file per public namespace where you can see the *entire* public surface at a glance (`keras/api/layers/__init__.py` reads like a table of contents).
- Freedom to refactor `keras/src/` without a deprecation cycle, as long as the re-export keeps the same name and object.
- A machine-checkable invariant: any public symbol that has no line in `keras/api/` is, by construction, not part of the supported API.

The trade-off: reading the code requires two hops to see a public symbol's body. That's a real cost — you'll feel it in every chapter — but it's the price of a stable public API over a moving implementation.

**Concept 2 — the backend seam.**

The rule that makes Keras 3 work as a multi-backend library is: **no file outside `keras/src/backend/` imports a specific framework.** Layers import `keras.src.ops` (backend-agnostic). Ops import `keras.src.backend` (switching module). Only `keras/src/backend/tensorflow/`, `keras/src/backend/jax/`, `keras/src/backend/torch/`, and `keras/src/backend/numpy/` are allowed to `import tensorflow`, `import jax`, etc.

You can verify this invariant yourself with a single command — it's one of the most useful one-liners in the whole codebase:

```bash
cd ~/projects/keras
git grep -nE '^\s*(import|from)\s+(tensorflow|torch|jax)\b' -- keras/src ':!keras/src/backend'
```

**Expected — verify:** This should return **zero or a handful of hits**, all of which are special cases (e.g., `keras/src/export/`, `keras/src/utils/`, or optional-dependency guards inside `try/except`). If you see a hit in `keras/src/layers/`, that's a bug or a version skew worth reporting.

*Python mechanics worth naming here.*

- **Re-export is just `import`.** `from keras.src.layers.core.dense import Dense` binds the *same class object* under a new name in the `keras.api` module. There's no wrapper, no subclass, no proxy. `keras.layers.Dense is keras.src.layers.core.dense.Dense` returns `True`. If you ever see `False`, someone has introduced an indirection (e.g., a deprecation shim that wraps the class).
- **`keras/__init__.py` is a namespace assembler.** It does not define logic; it imports from `keras.api` so that `import keras` yields the full surface. This is the standard Python idiom for "split a large package into an internal tree and a public tree."
- **The backend switcher uses import-time dispatch.** Choosing the backend is not done with `if` statements scattered through the code; it's done once when `keras.src.backend` is first imported. After that, `backend.numpy.matmul` is resolved to the chosen backend's function. This is why changing `KERAS_BACKEND` mid-process is not supported — the decision is frozen at import.

## F. Execution Flow

The chain from `import keras` to a usable `keras.layers.Dense` looks like this:

```
import keras
  └─ executes keras/__init__.py
       └─ imports keras/api/__init__.py
            └─ imports keras/api/layers/__init__.py
                 └─ from keras.src.layers.core.dense import Dense
                      └─ executes keras/src/layers/core/dense.py
                           └─ imports keras.src.layers.layer (base)
                           └─ imports keras.src.ops (backend-agnostic math)
                                └─ keras/src/ops/__init__.py
                                     └─ imports keras.src.backend (chooses backend)
                                          └─ reads KERAS_BACKEND, loads that backend pkg
```


Two points to internalize:

1. **The backend is chosen at import time of `keras.src.backend`.** Not per call. Not per layer. Once.
2. **The public/implementation seam appears exactly once per public symbol.** One `from ... import ...` line in `keras/api/`. Locate that line and you've located the implementation.

## G. Practical Exploration

Run these in order. Each is a small, safe inspection — nothing writes to your tree.

```bash
cd ~/projects/keras

# 1. Confirm the two-tree structure
ls
ls keras/src | head -40
ls keras/api 2>/dev/null | head -40

# 2. Where is Dense defined, and where is it re-exported?
git grep -n "class Dense" -- keras/src
git grep -n "Dense" -- keras/api/layers

# 3. Runtime identity check (if keras is importable)
python - <<'PY'
import keras
import keras.src.layers.core.dense as d
print("same object:", keras.layers.Dense is d.Dense)
print("module of public name:", keras.layers.Dense.__module__)
print("module of impl name:  ", d.Dense.__module__)
PY

# 4. The backend-seam invariant
git grep -nE '^\s*(import|from)\s+(tensorflow|torch|jax)\b' -- keras/src ':!keras/src/backend'

# 5. Size of each subsystem (tells you where to spend your time)
for d in keras/src/*/; do printf '%6d  %s\n' "$(git ls-files "$d" | wc -l)" "$d"; done | sort -rn
```

**Interpretation guide:**

- Command 3 printing `same object: True` confirms the re-export is a pure name binding. Printing `False` means an indirection was introduced — chase it before continuing.
- Command 4 returning only a few lines (ideally all inside `try/except ImportError` guards or under `keras/src/export/`) confirms the backend seam is intact.
- Command 5 gives you the shape of the repo: if `layers/` dominates, that's expected.

If any command fails because the path doesn't exist, that itself is data — tell me which one and I'll reconcile it with your actual layout.

## H. Hands-on Exercise

Pick one public symbol that you care about, and locate both its public re-export and its implementation. Recommended choice: `keras.ops.matmul`. It's a small, self-contained example of the whole chain.

Task:

```bash
cd ~/projects/keras

# Step 1: public re-export location
git grep -n "matmul" -- keras/api

# Step 2: implementation location
git grep -n "def matmul" -- keras/src

# Step 3: backend dispatch (repeat per backend)
git grep -n "def matmul" -- keras/src/backend
```

Write down three paths and one sentence each:

1. The `keras/api/...` file and line where `matmul` becomes public.
2. The `keras/src/ops/...` file and line where the backend-agnostic `matmul` is defined.
3. One backend's `matmul` implementation (e.g., the TensorFlow one).

Then answer: **when you call `keras.ops.matmul(a, b)` at the REPL, how many function frames deep is the actual framework call?** Count them by hand from the source you just located.

Undo: this exercise doesn't modify anything. There is nothing to revert.

## I. Common Misunderstandings

- **"`keras.layers.Dense` is a wrapper around some internal class."** No. It *is* `keras.src.layers.core.dense.Dense`. The re-export binds the same object.
- **"`keras/src/` is private and I shouldn't read it."** You should absolutely read it. It's the implementation. It's simply not the *stable public API* — you can read it, learn from it, and even import from it, but doing so risks breaking on a version bump.
- **"`KERAS_BACKEND` can be changed at runtime."** It's read once, at import of `keras.src.backend`. Changing the environment variable after that does nothing. To switch backends you must start a new process.
- **"All frameworks are imported when I `import keras`."** No — only the chosen backend is. This is why Keras can be installed with only one framework present.
- **"The layout in the GitHub `master` branch is what's in my clone."** Not necessarily. This is why we started with `git rev-parse --short HEAD`. If your revision and GitHub `master` disagree on a path, your clone wins for the rest of the course.
- **"`keras/api/` is autogenerated so I can ignore it."** You cannot — it's the map from public names to implementation. It's the single most useful folder for *finding* things.

## J. Chapter Summary

You now have the map:

- The repo has a **public tree** (`keras/api/`) and an **implementation tree** (`keras/src/`), connected by one-line re-exports.
- The implementation tree has a **backend seam** at `keras/src/backend/`, chosen once at import, never crossed outside that folder.
- `keras/src/ops/` is the backend-agnostic math surface used by layers and trainers.
- `keras/__init__.py` assembles the public surface; it defines no logic.
- The three heaviest subsystems (by file count) are typically `layers/`, `backend/`, and `ops/` — that's where the codebase's weight sits.

Everything in Chapters 2–30 is a walk down one branch of this map.

## K. Knowledge Check

Answer any subset; I'll grade before moving on if you like, or you can take **Next Chapter** and answer later.

1. *(Recall)* Name the two top-level trees that together define Keras's public surface, and state the role of each in one sentence.

2. *(Code reading)* Given the output of `git grep -n "class Dense" -- keras/src` and `git grep -n "Dense" -- keras/api/layers`, how would you determine — without running Python — whether the public symbol and the implementation class are the same object?

3. *(Reasoning)* Why does Keras choose the backend at *import time* rather than per-call? What does that choice make possible, and what does it make impossible?

4. *(Design)* What problem does the API/implementation split solve, and what is the cost it imposes on someone reading the source?

5. *(Navigation)* Write the `git grep` command you would use to find *every* place in `keras/src/` that imports TensorFlow directly outside the backend package, and explain what an unexpected hit would mean.
