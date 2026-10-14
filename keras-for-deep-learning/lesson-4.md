# Chapter 4 — The Base Layer Class

**Source-access reminder.** Still no live access to your clone or GitHub. Every file-level claim is marked *expected — verify* with a command that confirms it in your tree. The `Layer` class in particular has changed shape across Keras 3 minor revisions — the `__call__` path, the `add_weight` signature, and the build-config methods are the parts most likely to differ. Confirm each path with `less` before trusting my description.

## A. Chapter Objectives

By the end of this chapter you will be able to:

1. Name the two base classes `Layer` inherits from, and state what each contributes.
2. Explain the **build/call lifecycle**: `__init__` → `build` → `__call__` → `call`, and exactly when each runs.
3. Read `Layer.__call__` and describe the five things it does around the user's `call()`.
4. Explain `add_weight` as the *only* supported way to introduce a variable into a layer, and why direct assignment is discouraged.
5. Describe `compute_output_spec` vs `compute_output_shape` and when each is invoked.
6. Explain how serialization works: `get_config` / `from_config` vs `get_build_config` / `build_from_config`.
7. Understand `__setattr__` on `Layer` and the "not JSON-serializable" boundary between attributes and tracked state.
8. Locate the tests that exercise `Layer` and know which test files matter when you change it.

You will not yet read every line of `layer.py` — it is long. This chapter gives you the *shape* of the class and the methods you'll come back to in Chapters 5–7.

## B. Where This Fits in Keras

`Layer` is the center of gravity of the entire library. Chapter 3 introduced the primitives (`Tracker`, `global_state`, `get_uid`, `python_utils`). This chapter is where they all appear together, orchestrated by one class. Every concrete layer — `Dense`, `Conv2D`, `MultiHeadAttention`, the normalization layers, the RNN cells — inherits from `Layer` and changes only `__init__`, `build`, and `call`. `Model` (Chapter 7) inherits from `Layer` too, adding `fit`/`predict`/`evaluate` on top.

If you understand `Layer`, you understand 80% of Keras by volume. If you don't, every concrete layer chapter reads as a wall of methods with no shared structure.

## C. Actual Source Files

**Expected layout — verify each path with `ls`.**

| Path | Responsibility |
|---|---|
| `keras/src/layers/layer.py` | The `Layer` class. Long (600–1000+ lines depending on revision). |
| `keras/src/ops/operation.py` | The `Operation` base class — the piece `Layer` composes for symbolic/graph dispatch. |
| `keras/src/backend/common/variables.py` | `KerasVariable` — what `add_weight` returns. Chapter 5. |
| `keras/src/backend/common/keras_tensor.py` | `KerasTensor` — what layers accept in symbolic mode. Chapter 9. |
| `keras/src/utils/tracking.py` | `Tracker` — base for ownership. Chapter 3. |
| `keras/src/utils/python_utils.py` | Argument helpers used by `Layer.__init__`. Chapter 3. |
| `keras/src/dtype_policies/dtype_policy.py` | `DTypePolicy` — resolved from the `dtype=` argument. |
| `keras/src/layers/core/dense.py` | The smallest, most-read concrete layer. A natural test case. |
| `keras/src/layers/layer_test.py` (or `keras/src/layers/layer_test.py`) | The unit tests for `Layer`. **Read these before changing anything.** |

**Verify:**

```bash
cd ~/projects/keras
ls keras/src/layers/layer.py
ls keras/src/ops/operation.py
ls keras/src/layers/*_test.py 2>/dev/null | head
git ls-files keras/src | grep -i "layer_test\|test_layer" | head
wc -l keras/src/layers/layer.py keras/src/ops/operation.py
```

If `layer_test.py` isn't where you expect it, `git ls-files | grep test` will find the test files. Keras keeps tests alongside implementations in Keras 3 (`*_test.py`), unlike Keras 2 which used a separate `tests/` tree.

## D. Read the Code in the Correct Order

**1. Start with a concrete layer, not the base.**

Read `Dense` first — it's short, and it makes the base class concrete.

```bash
less keras/src/layers/core/dense.py
```

**Expected — verify:** A file around 100–200 lines. It defines:

- `class Dense(Layer)` decorated with `@keras_export("keras.layers.Dense")`.
- `__init__(self, units, activation=None, use_bias=True, kernel_initializer=..., bias_initializer=..., kernel_regularizer=None, bias_regularizer=None, activity_regularizer=None, kernel_constraint=None, bias_constraint=None, **kwargs)`.
- `build(self, input_shape)` — computes fan-in, creates `self.kernel` and `self.bias` via `self.add_weight(...)`.
- `call(self, inputs)` — `x = ops.matmul(inputs, self.kernel)`, add bias, apply activation.
- `compute_output_shape(self, input_shape)` — returns the shape with last dim = `units`.
- `get_config(self)` — returns the config dict for serialization.

Every one of those methods is either an override of a `Layer` hook or a plain method. Nothing else. Once you've read `Dense`, you've read every Keras layer in skeleton form. Everything else is a variation.

**2. Now the base class, top-down.**

```bash
less keras/src/layers/layer.py
```

Read in this order, in the file:

1. The module docstring and imports — which utilities it uses (you should recognize most of them from Chapter 3).
2. `class Layer(BackendLayer, Operation, ...)` or `class Layer(Operation, ...)` — check what it inherits from in *your* revision.
3. `__init__`.
4. `__call__` (this is where the magic is).
5. `build` (the default no-op).
6. `call` (the default raises `NotImplementedError`).
7. `add_weight`.
8. `compute_output_spec` and `compute_output_shape`.
9. `get_config` / `from_config` / `get_build_config` / `build_from_config`.
10. `__setattr__`, `__getattr__` if present.
11. `__repr__` and `__str__`.

Do not read linearly past `__call__` the first time. `__call__` is the orchestration method; everything after it is called *from* it or *by* it.

**3. The composition base.**

```bash
less keras/src/ops/operation.py
```

**Expected — verify:** `Operation` defines a smaller `__call__` that handles the symbolic/eager dispatch and delegates the actual "compute" to a `compute_output_spec` + `compute_output` pair, or to `call` if it's a `Layer`. **The relationship between `Operation.__call__` and `Layer.__call__` is the part of the codebase most likely to differ between revisions.** Read carefully and note the exact call chain in your tree.

**4. The tests.**

```bash
find keras/src -name "*layer*test*" -o -name "*test*layer*" | head
less keras/src/layers/layer_test.py 2>/dev/null || less $(find keras/src -name "*layer*test*" | head -1)
```

Read three or four tests end to end. They show you the *contract* of the class — what the maintainers actually test for, which is what they consider the supported behavior. Skip tests that require TensorFlow if you don't have it, but read the names.

## E. Explain the Implementation

**Concept 1 — the two (or three) base classes.**

In a typical Keras 3 revision, `Layer` looks roughly like:

```python
class Layer(BackendLayer, Operation, KerasSaveable):
    ...
```

or, in a simpler revision, `class Layer(Operation, ...)`. The exact inheritance list matters. Verify with:

```bash
grep -n "^class Layer" keras/src/layers/layer.py
```

What each contributes (in the common case):

- **`BackendLayer`** (sometimes `keras.src.backend.Layer`): backend-specific scaffolding — dtype handling, autocast policy, how to enter the backend's functional call context. Since Keras 3's backends differ in how they implement forward passes, this hook lets the base `Layer` be backend-agnostic.
- **`Operation`**: the symbolic-dispatch mechanism. It defines the small `__call__` that inspects arguments, and if any is a `KerasTensor`, routes to a symbolic code path instead of executing eagerly. `Layer` extends this to add build/name-scope/activity-regularizer logic.
- **`KerasSaveable`** (or similar): the marker interface that says "this object participates in `keras.saving`". It carries the `__reduce__` / config methods.

The design intent: `Layer` should be about *user-facing layer semantics* (build/call, weights, names, config), while backend and dispatch concerns live in the mixins. If you find yourself wanting to add a method to `Layer`, ask first whether it belongs in `Operation` or the backend mixin.

**Concept 2 — the build/call lifecycle.**

The supported contract for a concrete layer:

1. `__init__` runs when the user constructs the layer. It sets hyperparameters (`self.units`, `self.use_bias`, …) but does **not** create weights — because at construction time it doesn't know the input shape.
2. First call with real or symbolic input: `Layer.__call__` detects that the layer is not yet built, computes the input shape/spec, and calls `self.build(input_shape)`.
3. `build(input_shape)` creates the variables via `add_weight`. It must be idempotent-safe in the sense that once built, `self.built = True` prevents it from being called again on the same shape.
4. `call(inputs)` runs the actual forward computation. It assumes `build` has already run.
5. Subsequent calls skip `build`.

This is why `Dense` works without the user knowing `input_shape` in advance, and why changing the input shape after the layer is built raises an error rather than re-building. It's a deliberate one-build-per-layer contract, and it's the single most important thing to internalize about Keras layers.

**Concept 3 — `Layer.__call__` is the orchestrator.**

Reading the method carefully, `Layer.__call__` typically does the following (names vary — verify in your tree):

1. **Argument preprocessing.** Injects `training` if not passed (some layers accept `training` and need it during `call`).
2. **Symbolic dispatch.** If any input is a `KerasTensor`, route to `Operation`'s symbolic path (which uses `compute_output_spec` and builds a graph node). Chapter 9.
3. **Build check.** Call `self._maybe_build(inputs)` — if not built, build.
4. **Name scope.** Wrap the call in `with backend.name_scope(self.name):` (or similar). This is where `global_state` from Chapter 3 comes in.
5. **Dtype policy.** Set the autocast context for the duration of the call (mixed precision — Chapter 23).
6. **Call the user's `call`.** `outputs = self.call(inputs, **kwargs)`.
7. **Post-processing.** Apply activity regularizer, update any metrics the layer owns, restore dtype context, exit name scope.

The exact ordering and the exact method names differ between revisions. What doesn't differ is the *shape*: pre → build → scope → call → post.

**Concept 4 — `add_weight` is the only supported variable path.**

`Layer.add_weight(...)` does three things:

1. Resolves the initializer (a string, a callable, or an `Initializer` instance) into an `Initializer`.
2. Constructs a `KerasVariable` (Chapter 5) with the resolved dtype, shape, and trainable flag.
3. Registers the variable with the layer's `Tracker` (Chapter 3), so it appears in `layer.weights`, `layer.trainable_weights`, `layer.variables`, and gets saved.

The signature is something like:

```python
def add_weight(
    self,
    shape=None,
    initializer=None,
    dtype=None,
    trainable=True,
    autocast=True,
    regularizer=None,
    constraint=None,
    aggregation="mean",
    name=None,
):
```

**Verify this signature in your revision** — it changed between Keras 2 and Keras 3 (in particular, `name` is a keyword in Keras 3, and there's no positional `name`), and may change again. `git grep -n "def add_weight" keras/src/layers/layer.py`.

The important consequence: **direct assignment `self.w = some_tensor` does not register a variable.** It creates a plain attribute. It will not be trainable, not be saved, not appear in `.weights`. This is not a bug — it's the boundary between hyperparameters (tracked as attributes) and state (tracked as variables). The `Tracker` mechanism makes the distinction visible.

**Concept 5 — `compute_output_spec` vs `compute_output_shape`.**

Two methods with overlapping names:

- **`compute_output_spec(self, *args, **kwargs)`** — returns a `KerasTensor` (or a `TensorSpec`) describing the output *including dtype and structure*. This is the modern, preferred method. Used by the symbolic path.
- **`compute_output_shape(self, input_shape)`** — returns a `TensorShape`. Older, still supported for compatibility and for layers whose output shape doesn't depend on dtype.

A well-behaved Keras 3 layer implements `compute_output_spec`. It may also implement `compute_output_shape` as a convenience. If it implements neither, `Layer` falls back to **running the layer symbolically on a fake input** — which works but is slower and can fail for layers with data-dependent output shapes.

**Concept 6 — serialization has two halves.**

`get_config` / `from_config` handle **construction arguments**: the values you passed to `__init__`. A layer that overrides `__init__` with new arguments must override `get_config` to include them; otherwise `from_config` will fail on reload.

`get_build_config` / `build_from_config` handle **post-build state**: shapes, and anything that was determined at first call. This is a Keras 3 addition. Before Keras 3, saving a model that wasn't yet built didn't round-trip cleanly; now it can.

Verify:

```bash
git grep -n "def get_build_config\|def build_from_config" keras/src/layers/
git grep -n "def get_build_config\|def build_from_config" keras/src/layers/core/dense.py
```

**Concept 7 — `__setattr__` and the not-JSON-serializable boundary.**

Some layers override `__setattr__` to intercept certain assignments (e.g., to auto-track a sublayer assigned after construction). This is one of the subtler parts of `Layer` — reading it well requires understanding Python's attribute-resolution order.

If your revision's `Layer` overrides `__setattr__`, the pattern is something like:

```python
def __setattr__(self, name, value):
    # if value is trackable, register it with the tracker
    # then call the base __setattr__
    ...
```

The takeaway: not every attribute on a layer is a plain Python attribute. Some are mediated. Chapter 5 will unpack this when we look at how variables and sublayers are tracked.

*Python mechanics worth naming here.*

- **Method resolution order (MRO).** `Layer`'s MRO matters because `__call__` may be defined in more than one base. When `Dense(...)(x)` runs, Python uses `type(Dense).__mro__`. Print it: `python -c "from keras.layers import Dense; print([c.__name__ for c in Dense.__mro__])"`. The order you see tells you which `__call__` wins.
- **`super()` in mixed hierarchies.** `Layer.__init__` typically calls `super().__init__(**kwargs)`, which routes to `Operation.__init__` and then to the next class in the MRO. Reading `Layer.__init__` in isolation is misleading — you must know what the *next* class does.
- **`functools.wraps` and decorators.** Some lifecycle hooks are wrapped with decorators. Look for `@functools.wraps` or `@wraps` around methods in `Layer`.
- **Sentinel values.** `Layer.__init__` uses the `python_utils` conventions from Chapter 3 (`is_default`, `default`) to distinguish "user passed None" from "user passed nothing" for arguments like `dtype` and `trainable`.

## F. Execution Flow

The canonical path for a first call to a Dense layer:

```
y = Dense(64, activation="relu")(x)
```

executes as:

```
Dense(64, activation="relu")
  └─ Dense.__init__(units=64, activation="relu", **kwargs)
        └─ Layer.__init__(name=..., dtype=..., trainable=..., ...)
              └─ Operation.__init__()
              └─ Tracker.__init__()
              └─ self._name_scope = name_scope(...)  # global_state
              └─ self._dtype_policy = DTypePolicy(dtype)
              └─ self._tracker.lock()
        └─ self.units = 64
        └─ self.activation = activations.get("relu")

(x)  ── Python invokes type(Dense).__call__(layer, x)
  └─ Layer.__call__(x)
        └─ if not self.built:
              shape = compute_input_spec(x).shape
              self.build(shape)
                  └─ Dense.build(shape)
                        └─ self.kernel = self.add_weight(shape=(..., 64), ...)
                              └─ KerasVariable created  (Chapter 5)
                              └─ self._tracker.track(kernel)
                        └─ self.bias = self.add_weight(...)  if use_bias
        └─ with backend.name_scope(self.name):
              with self._dtype_policy.scope():   # mixed precision, Chapter 23
                  outputs = self.call(x)
                      └─ Dense.call(x)
                            └─ ops.matmul(x, self.kernel)
                            └─ ops.add(..., self.bias) if use_bias
                            └─ self.activation(outputs) if activation
        └─ apply activity_regularizer if set
        └─ return outputs
```


1. **`__init__` never creates weights.** Not `Dense`, not any other layer. Variables appear only in `build`.
2. **`Layer.__call__` is not `call`.** Users override `call`; `__call__` is the orchestrator. Confusing these two is the most common source of "why doesn't my custom layer work" bugs.
3. **`build` runs at most once per layer instance in a given graph.** After that, changing input shape is an error, not a rebuild.

## G. Practical Exploration

Read-only except where noted.

```bash
cd ~/projects/keras

# 1. The class header and its bases
grep -n "^class Layer" keras/src/layers/layer.py
python -c "from keras.layers import Dense; print([c.__name__ for c in Dense.__mro__])"

# 2. Signatures of the key methods
python - <<'PY'
import inspect
from keras.src.layers.layer import Layer
for name in ("__init__", "__call__", "build", "call", "add_weight",
             "compute_output_spec", "compute_output_shape",
             "get_config", "from_config",
             "get_build_config", "build_from_config"):
    fn = getattr(Layer, name, None)
    if fn is None:
        print(f"{name:24s} MISSING in this revision")
    else:
        try:
            print(f"{name:24s} {inspect.signature(fn)}")
        except (TypeError, ValueError):
            print(f"{name:24s} <no signature>")
PY

# 3. The smallest concrete layer, for reference
less keras/src/layers/core/dense.py

# 4. Layer tests — read a handful of test names
ls keras/src/layers/*_test.py 2>/dev/null | head
grep -n "    def test_" $(git ls-files keras/src/layers/layer_test.py) 2>/dev/null | head -20

# 5. Track where Layer.__call__ ends up — a live trace of a single forward pass
python - <<'PY'
import keras
l = keras.layers.Dense(4)
import numpy as np
y = l(np.ones((2, 3), dtype="float32"))
print("built:", l.built)
print("weights:", [w.name for w in l.weights])
print("output shape:", y.shape)
PY
```

**Interpretation guide:**

- Command 2: the exact signatures matter. If `add_weight` in your tree has a different first argument than `shape`, note it — the Keras 2 → 3 transition changed this and some third-party layers still use the old form.
- Command 4: if `layer_test.py` doesn't exist at that path, `git ls-files | grep layer_test` finds it. Keras keeps tests adjacent to implementations in `src/`.
- Command 5: `l.built` should be `False` immediately after construction and `True` after the first call. `l.weights` should be non-empty only after the call. This is the build/call contract in two lines.

## H. Hands-on Exercise

**Task: build the smallest custom layer that exercises every hook.**

Create `~/tmp/my_layer.py` (outside the repo — do not modify Keras):

```python
import numpy as np
import keras
from keras import ops

class Scale(keras.layers.Layer):
    """y = scale * x, with scale as a learned scalar weight."""

    def __init__(self, init_value=1.0, **kwargs):
        super().__init__(**kwargs)
        self.init_value = float(init_value)

    def build(self, input_shape):
        self.scale = self.add_weight(
            shape=(),
            initializer=keras.initializers.Constant(self.init_value),
            trainable=True,
            name="scale",
        )
        super().build(input_shape)   # note the call — see question 3

    def call(self, inputs):
        return self.scale * inputs

    def compute_output_spec(self, inputs):
        return keras.KerasTensor(inputs.shape, dtype=inputs.dtype)

    def get_config(self):
        cfg = super().get_config()
        cfg.update({"init_value": self.init_value})
        return cfg

# smoke test
l = Scale(init_value=2.5, name="my_scale")
x = np.ones((2, 3), dtype="float32")
print("before call: built =", l.built, "weights =", l.weights)
y = l(x)
print("after call:  built =", l.built, "weights =", [w.name for w in l.weights])
print("y =", np.asarray(y))

# round-trip
clone = Scale.from_config(l.get_config())
print("reconstructed name:", clone.name, "init_value:", clone.init_value)
```

Run it:

```bash
python ~/tmp/my_layer.py
```

Answer these:

1. **Why is `l.weights` empty before the call but non-empty after?** Point to the exact line in `Layer.__call__` (in your clone) that triggers `build`.
2. **What happens if you delete the `super().build(input_shape)` call inside `Scale.build`?** Try it, then read `Layer.build` in the source. Is `super().build(...)` doing anything important, or is it decorative?
3. **What happens if you remove `compute_output_spec`?** The layer still works in eager mode. Does it work inside a `keras.Sequential`? Try it. What error message do you get, and which file raised it?
4. **Remove `get_config`.** Does `Scale.from_config(l.get_config())` still work? What does the config look like without it?
5. **What is `self.scale` typed as?** `print(type(l.scale))`. Is it a NumPy array, a `KerasVariable`, or something else? Where does the multiplication `self.scale * inputs` actually dispatch?

**Undo:** `rm ~/tmp/my_layer.py`. Nothing in the repo was touched.

## I. Common Misunderstandings

- **"`build` is called automatically by `__init__`."** No. It's called on first `__call__`. That's why you can construct a `Dense(64)` without knowing the input size.
- **"`__call__` and `call` are the same thing."** No. `Layer.__call__` is the orchestrator you should never override in a custom layer. `call` is the hook you override. Overriding `__call__` breaks name scoping, dtype policy, symbolic dispatch, and weight tracking in ways that are painful to debug.
- **"I can just do `self.w = np.zeros(...)` and it'll be a variable."** No. It's a plain attribute. `self.add_weight(...)` is the only supported path. Directly assigning a `KerasVariable` (which you'd rarely need to do) is allowed in some revisions but not recommended.
- **"Calling `add_weight` twice with the same `name` raises."** It may or may not, depending on revision. It's not a uniqueness guarantee like `unique_object_name` on layers — the variable name is a hint for debugging, not an identity.
- **"`compute_output_shape` is required."** It isn't. In Keras 3, `compute_output_spec` is the preferred method. If neither is present, the layer runs symbolically on a fake input — slower, and fails for layers with data-dependent shapes.
- **"`get_config` should include the weights."** Never. Weights are saved via `get_build_config` + the weights file in the `.keras` archive. `get_config` is construction arguments only.
- **"`super().__init__(**kwargs)` is optional."** It isn't. Skipping it means `name`, `dtype`, `trainable`, and the tracker aren't set up. Errors from this omission look mysterious because the failure happens later.
- **"Layer instances can be re-used with a different input shape."** Not after they're built. If you need that, either pass the shape at construction time (via an `Input` layer or by building explicitly), or don't build until you know.
- **"`Layer` is stable across Keras 2 and Keras 3."** It isn't. `add_weight`'s signature, `__call__`'s symbolic branch, and the build-config methods are all Keras 3 changes. Third-party code written for Keras 2 often breaks here.

## J. Chapter Summary

- `Layer` composes (typically) `Operation` + a backend mixin + a serialization interface. Each contributes one concern; the MRO determines which `__call__` runs.
- The **build/call lifecycle** is `__init__` → first `__call__` → `build` → `call`. This one-build-per-layer contract is the central invariant.
- `Layer.__call__` orchestrates: argument preprocessing, symbolic dispatch, build-if-needed, name scope, dtype policy, user `call`, and post-processing. You never override it in a layer.
- `add_weight` is the only path to a trainable variable; it constructs a `KerasVariable` and registers it with `Tracker`.
- Serialization is two halves: `get_config`/`from_config` (construction args) and `get_build_config`/`build_from_config` (build-time state).
- Every concrete layer — `Dense`, `Conv2D`, `MultiHeadAttention` — overrides only `__init__`, `build`, `call`, and (optionally) the serialization methods.

You now have the vocabulary to read any layer in `keras/src/layers/`. Chapters 5 and 6 turn that vocabulary into fluency.

## K. Knowledge Check

Answer any subset; I'll grade before moving on if you like, or you can take **Next Chapter** and answer later.

1. *(Recall)* Name the three phases of the layer lifecycle in order, and state which one creates variables.

2. *(Code reading)* Given the `Scale` layer from the exercise, trace what happens the first time it is called inside a `keras.Sequential`. Which file calls `Scale.build`? Which line of `Layer.__call__`?

3. *(Reasoning)* Why is `build` deferred to the first call rather than run in `__init__`? Name the specific design pressure that produces this choice, and one cost it imposes on users.

4. *(Design)* `Layer.__call__` enters a `name_scope`, applies a dtype policy, and then calls `call`. Why does the dtype policy go *inside* the name scope rather than the other way around? (Hint: think about what each scope affects.)

5. *(Navigation)* Write the `git grep` command to find every class in `keras/src/` that overrides `build` but not `compute_output_spec`. What does the result tell you about which layers rely on the "run-on-fake-input" fallback?
