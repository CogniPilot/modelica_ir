# Rumoca Integration Guide

How Rumoca exports the DAE IR format.

## Overview

Rumoca is a Rust-based Modelica compiler that parses Modelica (.mo) files and exports them to DAE IR JSON format. The DAE IR format extends Base Modelica with explicit variable and equation classification following Modelica Specification Appendix B.

## Basic Usage

```bash
# Export a model to DAE IR JSON
rumoca model.mo --model=ModelName --json

# Output goes to stdout - redirect as needed
rumoca model.mo --model=ModelName --json > model.json
```

## What Rumoca Does

During export, Rumoca performs:

1. **Variable Classification** - Categorizes variables into:
   - `states` - Continuous state variables (x in Appendix B)
   - `algebraic` - Algebraic variables (y in Appendix B)
   - `discrete_real` - Discrete real variables
   - `discrete_valued` - Integer/Boolean/Enumeration variables (m in Appendix B)
   - `parameters` - Parameters (p in Appendix B)
   - `constants` - Named constants (c in Appendix B)
   - `inputs` - Input variables (u in Appendix B)
   - `outputs` - Output variables

2. **Equation Classification** - Categorizes equations into:
   - `continuous` - Continuous-time equations
   - `event` - Event-triggered equations
   - `discrete_real` - Discrete real update equations
   - `discrete_valued` - Discrete integer/boolean equations
   - `initial` - Initial condition equations

3. **State Index Assignment** - Each state variable receives a `state_index` for direct mapping to solver vectors

4. **Event Indicator Extraction** - Zero-crossing functions from when-conditions

## Output Format

Rumoca outputs DAE IR JSON with derivatives represented as `der(x)` function calls in equations:

```json
{
  "ir_version": "dae-0.1.0",
  "model_name": "BouncingBall",

  "variables": {
    "states": [
      {"name": "h", "vartype": "Real", "state_index": 0, "start": 1.0},
      {"name": "v", "vartype": "Real", "state_index": 1, "start": 0.0}
    ],
    "algebraic": [],
    "parameters": [
      {"name": "g", "vartype": "Real", "start": 9.81}
    ]
  },

  "equations": {
    "continuous": [
      {
        "eq_type": "simple",
        "lhs": {"op": "der", "args": [{"op": "var", "name": "h"}]},
        "rhs": {"op": "var", "name": "v"}
      },
      {
        "eq_type": "simple",
        "lhs": {"op": "der", "args": [{"op": "var", "name": "v"}]},
        "rhs": {"op": "neg", "args": [{"op": "var", "name": "g"}]}
      }
    ],
    "initial": [
      {"eq_type": "simple", "lhs": {"op": "var", "name": "h"}, "rhs": {"op": "literal", "value": 1.0}},
      {"eq_type": "simple", "lhs": {"op": "var", "name": "v"}, "rhs": {"op": "literal", "value": 0.0}}
    ]
  },

  "structure": {
    "n_states": 2,
    "n_algebraic": 0,
    "n_equations": 2,
    "is_ode": true
  }
}
```

## Key Design Decisions

### Derivatives as `der(x)` in Equations

Derivatives are NOT separate variables. They appear as `der(x)` function calls in equations:

```modelica
// Modelica source
der(h) = v;
der(v) = -g;
```

```json
// DAE IR output
{
  "equations": {
    "continuous": [
      {"lhs": {"op": "der", "args": [{"op": "var", "name": "h"}]}, "rhs": {"op": "var", "name": "v"}},
      {"lhs": {"op": "der", "args": [{"op": "var", "name": "v"}]}, "rhs": {"op": "neg", "args": [{"op": "var", "name": "g"}]}}
    ]
  }
}
```

This follows standard Modelica syntax and avoids redundant derivative variable declarations.

### State Indices

Each state variable has a `state_index` field for direct mapping to solver state vectors:

```json
"states": [
  {"name": "h", "state_index": 0, ...},
  {"name": "v", "state_index": 1, ...}
]
```

This enables efficient code generation without name lookups.

### Structural Metadata

The `structure` object provides pre-computed structural information:

```json
"structure": {
  "n_states": 2,
  "n_algebraic": 0,
  "n_equations": 2,
  "dae_index": 0,
  "is_ode": true
}
```

## Supported Modelica Features

Rumoca currently supports:

- Continuous equations with derivatives
- Algebraic equations
- Initial equations
- When-equations (converted to event indicators)
- For-equations (loop expansion)
- If-equations (balanced)
- Parameters and constants
- Real, Integer, Boolean types
- Arrays and subscripting
- Built-in functions (sin, cos, sqrt, etc.)

## Example: Bouncing Ball

Input (`bouncing_ball.mo`):
```modelica
model BouncingBall
  parameter Real e = 0.8 "Coefficient of restitution";
  parameter Real g = 9.81 "Gravity";
  Real h(start = 1) "Height";
  Real v(start = 0) "Velocity";
equation
  der(h) = v;
  der(v) = -g;
  when h < 0 then
    reinit(v, -e * pre(v));
  end when;
end BouncingBall;
```

Command:
```bash
rumoca bouncing_ball.mo --model=BouncingBall --json
```

## Schema Validation

Validate output against the DAE IR schema:

```bash
# Using Python jsonschema
python -c "
import json
import jsonschema

with open('schemas/dae_ir-0.1.0.schema.json') as f:
    schema = json.load(f)
with open('output.json') as f:
    model = json.load(f)
jsonschema.validate(model, schema)
print('Valid!')
"
```

## Troubleshooting

### Common Issues

1. **Model not found** - Ensure `--model=Name` matches the model name in the file
2. **Parse errors** - Check Modelica syntax; rumoca reports line/column
3. **Unsupported features** - Some Modelica 3.7 features are not yet implemented

### Getting Help

- Check rumoca tests in `rumoca/tests/` for working examples
- Report issues at the project repository
