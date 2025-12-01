# Base Modelica IR Alignment (MCP-0031)

## Overview

This document describes the alignment between our `modelica_ir` schemas and the [MCP-0031 Base Modelica specification](https://github.com/modelica/ModelicaSpecification/tree/MCP/0031/RationaleMCP/0031).

## What is Base Modelica?

Base Modelica is an **intermediate language specification** proposed in MCP-0031 that:

- Sits between high-level Modelica and simulation backends
- Separates frontend language concerns from backend simulation semantics
- Removes complex high-level constructs while preserving semantic equivalence
- Enables portable models and tool interoperability
- Serves as the basis for eFMI Equation Code

> "Base Modelica is a language to describe hybrid (continuous and discrete) systems with emphasis on defining the dynamic behavior."
> — MCP-0031 Rationale

## Schema Hierarchy

```
Base Modelica IR (base_modelica_ir-0.1.0)
    │
    │  MCP-0031 compliant
    │
    ▼
DAE IR (dae_ir-0.1.0)  ⭐ Recommended for simulation
    │
    │  Extends Base Modelica with:
    │  - Explicit variable classification
    │  - Equation classification
    │  - State indices
    │  - Event indicators
    │
    ▼
Full Modelica IR (modelica_ir-0.2.0)
    │
    │  Adds connector-based features
    │
    ▼
Simulation Backend
```

## Schema Variants

We provide **three schema variants**:

### 1. dae_ir-0.1.0.schema.json (DAE IR) ⭐ Recommended
**Purpose:** Explicit DAE structure for simulation backends

**Extends Base Modelica with:**
- **Classified variables** - `states`, `algebraic`, `discrete_real`, `discrete_valued`, `parameters`, `constants`, `inputs`, `outputs`
- **Classified equations** - `continuous`, `event`, `discrete_real`, `discrete_valued`, `initial`
- **State indices** - Each state has `state_index` for direct solver mapping
- **Event indicators** - Zero-crossing functions for hybrid systems
- **Structural metadata** - `n_states`, `n_algebraic`, `dae_index`, `is_ode`
- **Derivatives as `der(x)`** - Standard Modelica syntax in equations

**Use When:**
- Building simulation backends or code generators
- Want explicit classification without inference
- Targeting ODE/DAE solvers (scipy, JAX, etc.)
- Working with Rumoca → Cyecca pipeline

### 2. base_modelica_ir-0.1.0.schema.json (MCP-0031 Aligned)
**Purpose:** Simplified intermediate representation aligned with Base Modelica specification

**Excludes (per MCP-0031):**
- ❌ Connect equations (replaced with explicit variable coupling)
- ❌ Conditional components (structure fixed at translation time)
- ❌ Protected visibility in classes (all members public)
- ❌ Each keyword (array iteration simplified)
- ❌ Unbalanced if-equations (all branches must be balanced)
- ❌ Synchronous features (clocks removed)

**Includes:**
- ✅ Simple equations (lhs = rhs)
- ✅ Balanced if-equations (all branches define same variables)
- ✅ When-equations (with restrictions)
- ✅ For-equations (under review in Base Modelica 0.1)
- ✅ Arrays and subscripting
- ✅ Functions (user-defined + built-in subset)
- ✅ Source location tracking

**Use When:**
- Targeting eFMI Equation Code
- Need maximum tool interoperability
- Want guaranteed Base Modelica compliance

### 3. modelica_ir-0.2.0.schema.json (Full Modelica 3.7)
**Purpose:** Comprehensive representation of flattened Modelica 3.7 models

**Includes:**
- All equation types (simple, for, if, when, **connect**)
- Full statement types (including when-statements)
- Events with imperative statements
- Connect equations (expanded from connectors)
- Rich metadata and annotations
- Full Modelica 3.7 operator set

**Use When:**
- You need to preserve connect semantics
- Working with connector-based models (electrical, hydraulic, mechanical)
- Require full Modelica feature set

## Key Differences

| Feature | DAE IR ⭐ | Base Modelica IR | Full Modelica IR |
|---------|-----------|------------------|------------------|
| **Variable Classification** | ✅ Explicit (states, algebraic, etc.) | ❌ Must infer | ❌ Must infer |
| **Equation Classification** | ✅ Explicit (continuous, event, etc.) | ❌ Unified list | ❌ Unified list |
| **State Indices** | ✅ `state_index` field | ❌ No | ❌ No |
| **Event Indicators** | ✅ Zero-crossings | ❌ No | ❌ No |
| **Structural Metadata** | ✅ n_states, is_ode, etc. | ❌ No | ❌ No |
| **Derivatives** | ✅ `der(x)` in equations | ✅ `der(x)` in equations | ✅ `der(x)` in equations |
| **Connect Equations** | ❌ Must pre-expand | ❌ Must pre-expand | ✅ ConnectEquation |
| **If-Equations** | ✅ Balanced only | ✅ Balanced only | ✅ Balanced + unbalanced |
| **Source Tracing** | ✅ Optional | ✅ Required | ⚠️ Optional |

## Conversion: Full → Base Modelica

The transformation from full Modelica IR to Base Modelica IR involves:

### 1. Expand Connect Equations
```json
// Full Modelica IR
{
  "eq_type": "connect",
  "lhs": [{"name": "resistor"}, {"name": "p"}],
  "rhs": [{"name": "capacitor"}, {"name": "n"}]
}

// Base Modelica IR (expanded)
[
  {
    "eq_type": "simple",
    "lhs": {"op": "component_ref", "parts": [{"name": "resistor.p.v"}]},
    "rhs": {"op": "component_ref", "parts": [{"name": "capacitor.n.v"}]}
  },
  {
    "eq_type": "simple",
    "lhs": {
      "op": "+",
      "args": [
        {"op": "component_ref", "parts": [{"name": "resistor.p.i"}]},
        {"op": "component_ref", "parts": [{"name": "capacitor.n.i"}]}
      ]
    },
    "rhs": {"op": "literal", "value": 0}
  }
]
```

### 2. Balance If-Equations
```json
// Full Modelica IR (unbalanced - allowed)
{
  "eq_type": "if",
  "branches": [{"condition": {...}, "equations": [{"eq_type": "simple", "lhs": "y", "rhs": "x"}]}],
  "else_equations": []  // No else - unbalanced!
}

// Base Modelica IR (balanced - required)
{
  "eq_type": "if",
  "branches": [{"condition": {...}, "equations": [{"eq_type": "simple", "lhs": "y", "rhs": "x"}]}],
  "else_equations": [{"eq_type": "simple", "lhs": "y", "rhs": {"op": "literal", "value": 0}}]  // Explicit else
}
```

### 3. Separate Variables by Variability
```json
// Full Modelica IR
{
  "variables": [
    {"name": "g", "variability": "constant", "value": 9.81},
    {"name": "m", "variability": "parameter", "value": 1.0},
    {"name": "x", "variability": "continuous"}
  ]
}

// Base Modelica IR
{
  "constants": [{"name": "g", "variability": "constant", "start": 9.81}],
  "parameters": [{"name": "m", "variability": "parameter", "start": 1.0}],
  "variables": [{"name": "x", "variability": "continuous"}]
}
```

### 4. Add Source Tracing
```json
// Base Modelica IR requires source location tracking
{
  "equations": [
    {
      "eq_type": "simple",
      "lhs": {...},
      "rhs": {...},
      "source_ref": "eq_001"
    }
  ],
  "source_info": {
    "eq_001": {
      "file": "BouncingBall.mo",
      "line": 15,
      "column": 3,
      "original_code": "der(h) = v;"
    }
  }
}
```

## Design Rationale

### Why Two Schemas?

1. **Full Modelica IR (v0.2.0):**
   - Rumoca can export rich information before final flattening
   - Preserves connect semantics for tools that need them
   - Supports full Modelica 3.7 analysis
   - Better for human inspection and debugging

2. **Base Modelica IR (base-0.1.0):**
   - Guarantees interoperability with Base Modelica tools
   - Simpler for simulation backends
   - eFMI Equation Code compatibility
   - Ensures MCP-0031 compliance

### Recommended Workflow

```
Modelica Source
      ↓
   Rumoca Compiler
      ↓
Full Modelica IR (v0.2.0)   ← Rich, preserves connects
      ↓
[Optional Converter Tool]
      ↓
Base Modelica IR (base-0.1.0)   ← Simplified, guaranteed compliance
      ↓
Cyecca / eFMI / Backend Tools
```

## Validation

### Full Modelica IR
```python
import json
import jsonschema

with open('schemas/modelica_ir-0.2.0.schema.json') as f:
    schema = json.load(f)

with open('examples/bouncing_ball_v0.2.json') as f:
    model = json.load(f)

jsonschema.validate(model, schema)  # Validates against full Modelica 3.7
```

### Base Modelica IR
```python
import json
import jsonschema

with open('schemas/base_modelica_ir-0.1.0.schema.json') as f:
    schema = json.load(f)

with open('examples/bouncing_ball_base.json') as f:
    model = json.load(f)

jsonschema.validate(model, schema)  # Validates against Base Modelica MCP-0031
```

## MCP-0031 Compliance Checklist

Our Base Modelica IR schema implements the following MCP-0031 requirements:

- [x] **No connect equations** - Removed from equation types
- [x] **No unbalanced if-equations** - Else branch required
- [x] **Simplified array dimensions** - Single syntax (shape array)
- [x] **No protected visibility** - All variables public at class level
- [x] **Separated constants/parameters** - Top-level separation
- [x] **Source location tracking** - source_ref and source_info fields
- [x] **Variability simplification** - Only constant/parameter/discrete/continuous
- [x] **Balanced structures** - All equations define same variables in branches
- [ ] **For-equation decision** - Kept (pending MCP-0031 final decision)
- [ ] **Synchronous features** - Not yet implemented (low priority)

## References

- [MCP-0031 Rationale](https://github.com/modelica/ModelicaSpecification/blob/MCP/0031/RationaleMCP/0031/ReadMe.md)
- [Modelica Specification 3.7-dev](https://specification.modelica.org/master/)
- [Base Modelica Directory](https://github.com/modelica/ModelicaSpecification/tree/MCP/0031/RationaleMCP/0031)

## Status

- **Full Modelica IR (v0.2.0):** ✅ Complete
- **Base Modelica IR (base-0.1.0):** ✅ Complete (aligned with MCP-0031 draft 0.1)
- **Converter Tool:** ⚠️ Not yet implemented (future work)

## Contributing

When updating schemas:
1. Ensure Base Modelica IR remains MCP-0031 compliant
2. Full Modelica IR can add features as Modelica 3.7 evolves
3. Document differences in this file
4. Update examples for both schemas
5. Add migration guides for breaking changes
