# modelica_ir

JSON schemas for representing Modelica models as intermediate representations (IR), designed for tool interoperability and code generation.

## Overview

This repository defines **two complementary JSON schemas** for representing Modelica models:

### 1. Base Modelica IR (`base_modelica_ir-0.1.0`)
**Simplified, flattened format based on [MCP-0031](https://github.com/modelica/ModelicaSpecification/blob/MCP/0031/RationaleMCP/0031/)**

- ✅ **Compilation target** - Designed for code generation and simulation backends
- ✅ **Minimal complexity** - Flattened, no connect equations, balanced if-equations only
- ✅ **DAE-ready** - Clear separation of constants, parameters, and variables
- ✅ **Extensible** - Supports tool-specific annotations (e.g., Lie groups for manifold-aware solvers)
- ✅ **Standard-aligned** - Based on emerging Base Modelica standard (MCP-0031)

**Primary use case:** Exchange format between compilers (Rumoca) and backends (Cyecca)

```
Modelica source → Rumoca compiler → Base Modelica JSON → Cyecca backend → Generated code
```

### 2. Full Modelica IR (`modelica_ir-0.2.0`)
**Comprehensive format following Modelica 3.7 specification**

- ✅ **Feature-complete** - Supports connect equations, unbalanced if-equations, events
- ✅ **Source-level representation** - Preserves high-level structure
- ✅ **Analysis-friendly** - Useful for IDE tools, refactoring, documentation generation

**Primary use case:** Internal compiler representation, analysis tools, IDE plugins

## Which Schema Should I Use?

**Use Base Modelica IR (`base_modelica_ir-0.1.0`) if:**
- You're building a simulation backend or code generator
- You need maximum tool interoperability
- You want a simplified, standardized format

**Use Full Modelica IR (`modelica_ir-0.2.0`) if:**
- You're building analysis tools that need rich structural information
- You need to represent models before flattening (e.g., compiler intermediate stages)
- You want comprehensive Modelica 3.7 feature support

**For the Rumoca → Cyecca pipeline:** Use **Base Modelica IR** with Lie group annotations.

See [SCHEMA_SELECTION_GUIDE.md](SCHEMA_SELECTION_GUIDE.md) for detailed comparison.

## Features

### Base Modelica IR
- ✅ **MCP-0031 compliant** - Aligned with emerging standard
- ✅ **Flattened representation** - Post-elaboration, ready for code generation
- ✅ **Extensible annotations** - Tool-specific metadata support
- ✅ **Source tracking** - Line/column information for error reporting
- ✅ **Validated** - JSON Schema for automatic validation

### Full Modelica IR
- ✅ **Modelica 3.7 compliant** - Based on official specification
- ✅ **Comprehensive** - Equations, algorithms, events, functions, connections
- ✅ **Hierarchical references** - Component references like `vehicle.wheels[1].pressure`
- ✅ **70+ operators** - Full expression language support
- ✅ **Validated** - JSON Schema for automatic validation

## Current Versions

- **Base Modelica IR:** `base-0.1.0` (aligned with MCP-0031)
- **Full Modelica IR:** `0.2.0` (aligned with Modelica 3.7)

See [SCHEMA_CHANGELOG.md](SCHEMA_CHANGELOG.md) for version history and migration guides.

## Structure

### Base Modelica IR Structure

```json
{
  "ir_version": "base-0.1.0",
  "base_modelica_version": "0.1",
  "model_name": "BouncingBall",

  "constants": [
    {"name": "g", "type": "Real", "value": 9.81, "unit": "m/s^2"}
  ],

  "parameters": [
    {"name": "e", "type": "Real", "value": 0.7, "unit": "1"}
  ],

  "variables": [
    {"name": "h", "type": "Real", "variability": "continuous"},
    {"name": "v", "type": "Real", "variability": "continuous"}
  ],

  "equations": [
    {"eq_type": "simple", "lhs": {...}, "rhs": {...}},
    {"eq_type": "when", "condition": {...}, "statements": [...]}
  ],

  "source_info": {
    "modelica_version": "3.7",
    "generated_by": "Rumoca 0.1.0"
  },

  "metadata": {
    "lie_groups": {
      "orientation": {"type": "SO3", "variables": ["q[1]", "q[2]", "q[3]", "q[4]"]}
    }
  }
}
```

**Key features:**
- Separate lists for constants, parameters, and variables (DAE-ready)
- Simpler equation types (no connect, balanced if-equations only)
- Source tracking for error reporting
- Extensible metadata for tool-specific annotations

### Full Modelica IR Structure

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

**Key features:**
- Unified variable list
- Rich equation types (connect, unbalanced if, events)
- Algorithm sections with imperative statements
- Function definitions

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

### Base Modelica IR Examples
- [`bouncing_ball_base.json`](examples/bouncing_ball_base.json) - Bouncing ball in Base Modelica format (MCP-0031)
- Demonstrates: constants/parameters/variables separation, when-equations, source tracking

### Full Modelica IR Examples
- [`bouncing_ball_v0.2.json`](examples/bouncing_ball_v0.2.json) - Bouncing ball in full Modelica IR format
- [`bouncing_ball.json`](examples/bouncing_ball.json) - Legacy v0.1.0 format

**Recommended:** Use Base Modelica examples as templates for new models.

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
