# Nodes and Namespaces (Transition Architecture)

## Purpose
Document the planned node protocol and namespace system that will replace the current flat-key
routing path. These modules exist in the repo but are not yet wired into the production pipeline.
Understanding them is essential before extending the engine.

## The problem they solve
The current production path (`engine/routing.py` + `engine/motion.py`) emits taps as short
flat keys like `fov.g`, `fov.h`, `fov_geom_norm`. As the system grows — more parameters, more
debug signals, streaming, per-stage introspection — flat keys collide and become impossible to
route reliably. The node/namespace layer is the answer.

## Two modules, one design

### `engine/namespace.py`
Defines stable key construction and the canonical domain vocabulary.

```python
class Domain(str, Enum):
    FEATURE  = "feature"
    GEOMETRY = "geometry"
    MEDIUM   = "medium"
    TEMPORAL = "temporal"
    MODIFIER = "modifier"
    OUTPUT   = "output"
```

`Domain` maps exactly to the namespace scheme in `namespace_map.md`. It is the enum that makes
the proposed `feature.spectral.centroid`, `geometry.fov.out`, `output.az` keys machine-checkable
rather than just documentation convention.

`Namespace` is a frozen dataclass that builds tap keys:
```python
ns = Namespace("geometry.fov")
ns.tap("out")   # → "geometry.fov.out"
ns.tap("debug.curve256")  # → "geometry.fov.debug.curve256"
```

Helper `ns(domain, *parts)` builds one-shot keys without creating a `Namespace` object:
```python
ns(Domain.OUTPUT, "az")  # → "output.az"
```

### `engine/nodes/base.py`
Defines the `Node` protocol: every processing stage (Geometry, Medium, Hysteresis, Brownian)
should eventually implement this interface.

```python
class Node(Protocol):
    ns: Namespace

    def apply(
        self,
        x: Signal,          # 1D time series (np.ndarray)
        *,
        ctx: NodeContext,   # fps, namespace string
        taps: Taps,         # mutable dict for debug signals
    ) -> Signal:
        ...
```

`NodeBase` provides the boilerplate — namespace storage, `tap_key()`, `tap_set()` — so
concrete nodes only need to override `apply()`.

`tap()` helper writes a signal into the taps dict safely (converts to float64, optional copy):
```python
tap(taps, ns.tap("out"), result)
```

`NodeContext` carries per-run shared state:
```python
@dataclass
class NodeContext:
    fps: float
    namespace: str = ""
```

## Relationship to the current codebase

```mermaid
graph LR
  subgraph Now
    R[routing.py\nflat keys] --> M[motion.py\nmotion dict]
  end
  subgraph Planned
    GN[GeometryNode\n.apply()] --> Namespace[namespace.py\nDomain + Namespace]
    MN[MediumNode\n.apply()] --> Namespace
    HN[HysteresisNode\n.apply()] --> Namespace
  end
  Now -.migration.-> Planned
```

The current `geometry.py`, `medium.py`, `hysteresis.py` do all heavy computation but emit
output through direct dict assignments in `routing.py`. The planned transition wraps each stage
in a node class, giving it a namespace, and has `routing.py` call `.apply()` instead of
inline functions.

## Migration path (from namespace_map.md)
1. **Stabilize contracts** — lock the canonical keys per stage using `Domain` and `Namespace`.
2. **Wrap as nodes** — implement `GeometryNode`, `MediumNode`, `HysteresisNode` that call
   existing stage functions but emit namespaced taps.
3. **Dual outputs** (one release) — emit both legacy keys (`fov_geom_norm`, `fov.g`) and
   namespaced keys to avoid breaking UI/export consumers.
4. **Routing swap** — in `routing.py`, prefer the node `.apply()` path; keep legacy fallback.
5. **UI + export switch** — update readers to prefer namespaced keys with legacy fallback.
6. **Remove legacy keys** — only after training + UI regression is clean.

## Current status
- `engine/namespace.py` and `engine/nodes/base.py` are implemented and importable.
- `engine/nodes/normalize.py` exists as a working example node.
- No production code currently imports from `engine/nodes/` or uses `Domain`/`Namespace` at runtime.
- The transition is pre-requisite for streaming (which needs per-frame stateful `.apply()` calls)
  and for the namespace-keyed OSC output (`net.*` keys in the transport schema).

## Key files
- `engine/namespace.py`
- `engine/nodes/base.py`
- `engine/nodes/normalize.py`
- `engine/routing.py` (current production path)
- `docs/obsidian/01_architecture/namespace_map.md` (full migration plan)
