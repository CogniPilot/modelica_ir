# modelica_ir

A neutral intermediate representation (IR) for Modelica-like models, designed for exchanging flattened models between compilers and analysis tools.

## Overview

This repository defines a **JSON schema** for representing flattened Modelica models in a tool-agnostic format. It serves as an interchange format between:

- **Compilers** (e.g., [Rumoca](https://github.com/jgoppert/rumoca)) - Parse and flatten Modelica source code
- **Analysis Tools** (e.g., [Cyecca](https://github.com/jgoppert/cyecca)) - Perform symbolic analysis, simulation, reachability analysis

```
Modelica source → Rumoca compiler → modelica_ir JSON → Cyecca analysis → Results
```

## Features

- ✅ **Modelica 3.7 compliant** - Based on the official Modelica specification
- ✅ **Flattened representation** - Post-elaboration, ready for symbolic processing
- ✅ **Comprehensive** - Equations, algorithms, events, functions, connections
- ✅ **Extensible** - Metadata fields for domain-specific annotations (e.g., Lie groups)
- ✅ **Validated** - JSON Schema for automatic validation
- ✅ **Tool-agnostic** - Language-neutral JSON format

## Schema Version

**Current Version:** `0.2.0`

See [SCHEMA_CHANGELOG.md](SCHEMA_CHANGELOG.md) for version history and migration guides.

## Structure

### Top-Level Model

```json
{
  "ir_version": "0.2.0",
  "model_name": "MyModel",
  "variables": [...],
  "equations": [...],
  "initial_equations": [...],
  "algorithms": [...],
  "initial_algorithms": [...],
  "events": [...],
  "functions": [...],
  "metadata": {...}
}
```

### Variables

Variables are fully qualified with type, kind, attributes, and optional metadata:

```json
{
  "name": "vehicle.position.x",
  "vartype": "Real",
  "kind": "state",
  "shape": [],
  "unit": "m",
  "comment": "X position in inertial frame",
  "default": 0.0,
  "min": null,
  "max": null,
  "nominal": 1.0,
  "fixed": true
}
```

**Variable kinds:**
- `state` - Continuous state variable (has derivative)
- `algebraic` - Algebraic variable
- `input` - External input
- `output` - Output variable
- `parameter` - Fixed parameter
- `constant` - Compile-time constant
- `discrete` - Discrete state (changes only at events)
- `flow` - Flow variable (for connectors)
- `across` - Across variable (for connectors)
- `stream` - Stream variable (for transport)

### Equations

Equations are discriminated by `eq_type`:

#### Simple Equation
```json
{
  "eq_type": "simple",
  "lhs": {
    "op": "der",
    "args": [{"op": "component_ref", "parts": [{"name": "x"}]}]
  },
  "rhs": {
    "op": "component_ref",
    "parts": [{"name": "v"}]
  }
}
```

#### For-Equation
```json
{
  "eq_type": "for",
  "indices": [
    {
      "index": "i",
      "range": {"op": "range", "args": [...]}
    }
  ],
  "equations": [...]
}
```

#### If-Equation
```json
{
  "eq_type": "if",
  "branches": [
    {
      "condition": {...},
      "equations": [...]
    }
  ],
  "else_equations": [...]
}
```

#### When-Equation
```json
{
  "eq_type": "when",
  "branches": [
    {
      "condition": {"op": "edge", "args": [...]},
      "equations": [...]
    }
  ]
}
```

#### Connect-Equation
```json
{
  "eq_type": "connect",
  "lhs": [{"name": "resistor", "subscripts": []}, {"name": "p"}],
  "rhs": [{"name": "capacitor", "subscripts": []}, {"name": "n"}]
}
```

### Expressions

Expressions use a unified tree structure with an `op` field:

```json
{
  "op": "operator_name",
  "args": [...]
}
```

**Operators:**
- **Literals:** `"literal"` (with `value` field)
- **References:** `"component_ref"` (with `parts` array), `"var"` (deprecated)
- **Arithmetic:** `"+"`, `"-"`, `"*"`, `"/"`, `"^"`, `"neg"`
- **Element-wise:** `".+"`, `".-"`, `".*"`, `"./"`, `".^"`
- **Comparison:** `"=="`, `"!="`, `"<"`, `"<="`, `">"`, `">="`
- **Logical:** `"and"`, `"or"`, `"not"`
- **Conditional:** `"if"` (ternary)
- **Modelica operators:** `"der"`, `"pre"`, `"edge"`, `"change"`, `"initial"`, `"terminal"`
- **Math functions:** `"sin"`, `"cos"`, `"exp"`, `"log"`, `"sqrt"`, `"abs"`, etc.
- **Array:** `"array"`, `"range"`, `"min"`, `"max"`, `"sum"`, `"product"`
- **Function call:** `"call"` (with `func` and `args`)

**Example - Hierarchical Component Reference:**
```json
{
  "op": "component_ref",
  "parts": [
    {"name": "vehicle", "subscripts": []},
    {"name": "wheels", "subscripts": [{"op": "literal", "value": 1}]},
    {"name": "pressure", "subscripts": []}
  ]
}
```
Represents: `vehicle.wheels[1].pressure`

### Algorithms

Algorithm sections contain imperative statements:

```json
{
  "statements": [
    {
      "stmt": "assign",
      "target": [{"name": "sum"}],
      "expr": {"op": "literal", "value": 0}
    },
    {
      "stmt": "for",
      "indices": [{"index": "i", "range": {...}}],
      "body": [...]
    },
    {
      "stmt": "if",
      "branches": [
        {
          "condition": {...},
          "statements": [...]
        }
      ]
    }
  ]
}
```

**Statement types:**
- `assign` - Assignment: `target := expr`
- `if` - Conditional statement
- `for` - For loop
- `while` - While loop
- `when` - When statement
- `reinit` - Reinitialize state variable
- `break` - Break from loop
- `return` - Return from function
- `call` - Function call statement (assert, terminate, etc.)

### Events

Events represent discrete state changes triggered by conditions:

```json
{
  "condition": {
    "op": "<",
    "args": [
      {"op": "component_ref", "parts": [{"name": "h"}]},
      {"op": "literal", "value": 0}
    ]
  },
  "statements": [
    {
      "stmt": "reinit",
      "target": [{"name": "v"}],
      "expr": {...}
    }
  ],
  "is_initial": false
}
```

### Functions

Function definitions include inputs, outputs, and algorithm body:

```json
{
  "name": "myFunction",
  "inputs": [
    {"name": "x", "vartype": "Real", "kind": "input", "shape": []}
  ],
  "outputs": [
    {"name": "y", "vartype": "Real", "kind": "output", "shape": []}
  ],
  "protected_vars": [],
  "algorithm": {
    "statements": [...]
  },
  "is_pure": true
}
```

### Metadata

Model-level and variable-level metadata for annotations:

**Model metadata:**
```json
{
  "metadata": {
    "description": "Quadrotor model with SE(2,3) structure",
    "lie_structure": "SE23",
    "formulation": "mixed_invariant"
  }
}
```

**Variable metadata:**
```json
{
  "name": "q",
  "vartype": "Real",
  "kind": "state",
  "shape": [4],
  "metadata": {
    "lie_group": "SO3",
    "chart": "quaternion"
  }
}
```

## Examples

See [examples/](examples/) directory:

- [`bouncing_ball_v0.2.json`](examples/bouncing_ball_v0.2.json) - Classic bouncing ball with events
- [`bouncing_ball.json`](examples/bouncing_ball.json) - Legacy v0.1.0 format

## Validation

Validate JSON files against the schema using the provided Python tool:

```bash
cd modelica_ir
python tools/validate.py examples/bouncing_ball_v0.2.json
```

Or use any JSON Schema validator:

```python
import json
import jsonschema

with open('schemas/modelica_ir-0.2.0.schema.json') as f:
    schema = json.load(f)

with open('examples/bouncing_ball_v0.2.json') as f:
    model = json.load(f)

jsonschema.validate(model, schema)
print("✓ Valid!")
```

## Integration Guides

- [Cyecca Integration](docs/integration_guides/cyecca.md) - Importing modelica_ir into Cyecca
- [Rumoca Integration](docs/integration_guides/rumoca.md) - Exporting from Rumoca to modelica_ir

## Design Principles

1. **Flat and Simple** - Represents post-elaboration models, no class hierarchy
2. **Complete** - Captures all information needed for symbolic analysis and simulation
3. **Validated** - JSON Schema ensures conformance
4. **Extensible** - Metadata fields allow domain-specific extensions
5. **Tool-Neutral** - No assumptions about backend implementation

## Comparison to Other Formats

### vs. FMI (Functional Mock-up Interface)
- **FMI:** Binary C interface for co-simulation, XML model description
- **modelica_ir:** Symbolic representation for static analysis and transformation
- **Use together:** modelica_ir for analysis, FMI for runtime simulation

### vs. Modelica Source Code
- **Modelica:** High-level, hierarchical, object-oriented
- **modelica_ir:** Flat, explicit, ready for symbolic processing
- **Pipeline:** Modelica → (compiler) → modelica_ir

### vs. MathML / SymPy
- **MathML/SymPy:** General mathematical expressions
- **modelica_ir:** Domain-specific for DAE systems with discrete events
- **Richer:** Includes variables, equations, algorithms, events, functions

## Roadmap

- [ ] Add support for external objects
- [ ] Add support for synchronous features (clocked equations)
- [ ] Add support for partial derivatives annotations
- [ ] Consider adding simplified "canonical form" variant (ODE/DAE only)
- [ ] Add more comprehensive examples (electrical, mechanical, fluid systems)

## Contributing

Issues and pull requests welcome! Please ensure:

1. JSON examples validate against schema
2. Documentation is updated for schema changes
3. Version number is bumped for breaking changes
4. Changelog is updated

## License

See [LICENSE](LICENSE)

## References

- [Modelica Specification 3.7](https://specification.modelica.org/master/)
- [Rumoca Compiler](https://github.com/jgoppert/rumoca)
- [Cyecca Analysis Tool](https://github.com/jgoppert/cyecca)
- [JSON Schema Specification](https://json-schema.org/)
