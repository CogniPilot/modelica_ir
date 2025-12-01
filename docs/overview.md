# Modelica IR Overview

This repository defines JSON schemas for representing Modelica models as intermediate representations (IR) suitable for simulation backends and code generators.

## Schema Hierarchy

```
Base Modelica IR (base_modelica_ir-0.1.0)
    │
    │  MCP-0031 compliant, simplified
    │
    ▼
DAE IR (dae_ir-0.1.0)  ⭐ Recommended
    │
    │  Extends Base Modelica with:
    │  - Explicit variable classification (states, algebraic, etc.)
    │  - Equation classification (continuous, event, initial)
    │  - State indices for solver mapping
    │  - Event indicators for hybrid systems
    │  - Structural metadata (n_states, is_ode, etc.)
    │
    ▼
Full Modelica IR (modelica_ir-0.2.0)
    │
    │  Adds connector-based features:
    │  - Connect equations
    │  - Unbalanced if-equations
    │
    ▼
Simulation Backend (Cyecca, etc.)
```

## DAE IR: The Recommended Schema

**DAE IR** (`dae_ir-0.1.0.schema.json`) is the recommended schema for simulation pipelines. It extends Base Modelica with explicit DAE structure following Modelica Specification Appendix B.

### Key Features

1. **Classified Variables** - Variables are pre-sorted into categories:
   - `states` - Continuous state variables (x)
   - `algebraic` - Algebraic variables (y)
   - `discrete_real` - Discrete real variables
   - `discrete_valued` - Integer/Boolean variables (m)
   - `parameters` - Parameters (p)
   - `constants` - Named constants (c)
   - `inputs` / `outputs` - Interface variables

2. **Classified Equations** - Equations are pre-sorted:
   - `continuous` - Continuous-time equations
   - `event` - Event-triggered equations
   - `discrete_real` / `discrete_valued` - Discrete update equations
   - `initial` - Initial condition equations

3. **State Indices** - Each state has a `state_index` for direct mapping to solver vectors

4. **Derivatives as `der(x)`** - Derivatives appear as function calls in equations, not separate variables

5. **Structural Metadata** - Pre-computed information:
   - `n_states`, `n_algebraic`, `n_equations`
   - `dae_index` - Differential index of the DAE
   - `is_ode` - True if system is pure ODE

### Example

```json
{
  "ir_version": "dae-0.1.0",
  "model_name": "Pendulum",

  "variables": {
    "states": [
      {"name": "theta", "state_index": 0, "start": 0.5},
      {"name": "omega", "state_index": 1, "start": 0.0}
    ],
    "algebraic": [],
    "parameters": [
      {"name": "L", "start": 1.0},
      {"name": "g", "start": 9.81}
    ]
  },

  "equations": {
    "continuous": [
      {
        "eq_type": "simple",
        "lhs": {"op": "der", "args": [{"op": "var", "name": "theta"}]},
        "rhs": {"op": "var", "name": "omega"}
      },
      {
        "eq_type": "simple",
        "lhs": {"op": "der", "args": [{"op": "var", "name": "omega"}]},
        "rhs": {
          "op": "*",
          "args": [
            {"op": "neg", "args": [{"op": "/", "args": [{"op": "var", "name": "g"}, {"op": "var", "name": "L"}]}]},
            {"op": "sin", "args": [{"op": "var", "name": "theta"}]}
          ]
        }
      }
    ],
    "initial": []
  },

  "structure": {
    "n_states": 2,
    "n_algebraic": 0,
    "is_ode": true
  }
}
```

## Workflow

```
┌─────────────────┐
│  Modelica (.mo) │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│     Rumoca      │  Rust Modelica compiler
│   (Compiler)    │  - Parses Modelica
└────────┬────────┘  - Classifies variables
         │           - Exports DAE IR JSON
         ▼
┌─────────────────┐
│   DAE IR JSON   │  Intermediate representation
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│     Cyecca      │  Python simulation backend
│   (Backend)     │  - BLT analysis
└────────┬────────┘  - Code generation
         │           - Simulation
         ▼
┌─────────────────┐
│  Simulation     │  JAX, NumPy, C, etc.
│    Output       │
└─────────────────┘
```

## When to Use Each Schema

| Schema | Use Case |
|--------|----------|
| **DAE IR** ⭐ | Simulation backends, code generation, BLT analysis |
| **Base Modelica IR** | eFMI compatibility, tool interoperability |
| **Full Modelica IR** | Connector-based models, full Modelica 3.7 features |

See [SCHEMA_SELECTION_GUIDE.md](../SCHEMA_SELECTION_GUIDE.md) for detailed guidance.

## Directory Structure

```
modelica_ir/
├── schemas/
│   ├── dae_ir-0.1.0.schema.json         ⭐ Recommended
│   ├── base_modelica_ir-0.1.0.schema.json
│   └── modelica_ir-0.2.0.schema.json
├── examples/
│   ├── bouncing_ball_dae.json           ⭐ DAE IR example
│   ├── bouncing_ball_v0.2.json
│   └── bouncing_ball_base.json
├── docs/
│   ├── overview.md                       (this file)
│   ├── BASE_MODELICA_ALIGNMENT.md
│   └── integration_guides/
│       ├── rumoca.md
│       └── cyecca.md
└── README.md
```

## Getting Started

1. **Generate DAE IR** from Modelica:
   ```bash
   rumoca model.mo --model=ModelName --json > model.json
   ```

2. **Load in Cyecca**:
   ```python
   from cyecca.io import load_dae_ir_json
   model = load_dae_ir_json(open('model.json').read())
   ```

3. **Validate** (optional):
   ```python
   from cyecca.io import validate_dae_ir_file
   errors = validate_dae_ir_file('model.json')
   ```

## References

- [Modelica Specification Appendix B](https://specification.modelica.org/) - DAE formalism
- [MCP-0031 Base Modelica](https://github.com/modelica/ModelicaSpecification/tree/MCP/0031) - Base language spec
- [Integration Guide: Rumoca](integration_guides/rumoca.md)
- [Integration Guide: Cyecca](integration_guides/cyecca.md)
