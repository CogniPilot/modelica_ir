# Cyecca Integration Guide

How Cyecca imports and processes DAE IR JSON.

## Overview

Cyecca is a Python-based simulation backend that loads DAE IR JSON, performs BLT (Block Lower Triangular) analysis, and generates simulation code. It provides tools for structural analysis, causality assignment, and code generation.

## Basic Usage

```python
from cyecca.io import load_dae_ir_json

# Load from JSON string
json_str = open('model.json').read()
model = load_dae_ir_json(json_str)

# Or load from file path
from cyecca.io import import_dae_ir
model = import_dae_ir('model.json')

print(model)
# Model: BouncingBall
#   States: 2
#   Algebraic Vars: 0
#   Parameters: 2
#   Equations: 2
```

## The Model Object

After loading, you have a `Model` object with classified variables and equations:

```python
# Access classified variables
for state in model.states:
    print(f"State: {state.name}, index: {state.state_index}")

for param in model.parameters:
    print(f"Parameter: {param.name}, value: {param.start}")

for alg in model.algebraic_vars:
    print(f"Algebraic: {alg.name}")

# Access equations
for eq in model.equations:
    print(f"Equation: {eq.lhs} = {eq.rhs}")

for eq in model.initial_equations:
    print(f"Initial: {eq.lhs} = {eq.rhs}")
```

## BLT Analysis

Cyecca performs Block Lower Triangular analysis to determine equation evaluation order:

```python
from cyecca.analysis import blt_sort, BlockType

result = blt_sort(model)

print(f"Number of blocks: {result.n_blocks}")
print(f"States: {result.state_variables}")
print(f"Algebraic: {result.algebraic_variables}")
print(f"Has algebraic loops: {result.has_algebraic_loops}")
print(f"Is well-posed: {result.is_well_posed}")

# Iterate through blocks in evaluation order
for block in result.blocks:
    if block.block_type == BlockType.SCALAR:
        print(f"Scalar block: {block.variables}")
    else:
        print(f"Algebraic loop: {block.variables}")
```

## Working with Expressions

DAE IR expressions are imported as Python objects:

```python
from cyecca.ir.expr import VarRef, FunctionCall, BinaryOp, Literal

# Derivatives appear as FunctionCall with func="der"
for eq in model.equations:
    if isinstance(eq.lhs, FunctionCall) and eq.lhs.func == "der":
        # This is a derivative equation: der(x) = ...
        state_ref = eq.lhs.args[0]  # VarRef to the state
        print(f"Derivative of {state_ref.name}")
```

## Pipeline: Rumoca to Cyecca

The typical workflow:

```bash
# 1. Generate DAE IR JSON from Modelica
rumoca model.mo --model=ModelName --json > model.json
```

```python
# 2. Load into Cyecca
from cyecca.io import load_dae_ir_json

with open('model.json') as f:
    json_str = f.read()
model = load_dae_ir_json(json_str)

# 3. Perform BLT analysis
from cyecca.analysis import blt_sort
result = blt_sort(model)

# 4. Check model validity
if result.is_well_posed:
    print("Model is well-posed")
else:
    for warning in result.warnings:
        print(f"Warning: {warning}")
```

## Handling Derivatives

In DAE IR, derivatives are NOT separate variables - they appear as `der(x)` function calls in equations. Cyecca handles this automatically:

```python
# The model has states (x), not derivative variables (der_x)
for state in model.states:
    print(f"State: {state.name}")
    # Derivatives are in equations as der(state.name)

# Find derivative equations
from cyecca.ir.expr import FunctionCall, VarRef

for eq in model.equations:
    if isinstance(eq.lhs, FunctionCall) and eq.lhs.func == "der":
        state_name = eq.lhs.args[0].name
        print(f"der({state_name}) = {eq.rhs}")
```

## Validation

Validate JSON against the DAE IR schema before loading:

```python
from cyecca.io import validate_dae_ir_file

errors = validate_dae_ir_file('model.json')
if errors:
    for error in errors:
        print(f"Validation error: {error}")
else:
    print("Valid DAE IR JSON")
```

## Example: Complete Workflow

```python
import subprocess
from cyecca.io import load_dae_ir_json
from cyecca.analysis import blt_sort

# Run rumoca to generate JSON
result = subprocess.run(
    ['rumoca', 'model.mo', '--model', 'MyModel', '--json'],
    capture_output=True,
    text=True
)

if result.returncode != 0:
    raise RuntimeError(f"rumoca failed: {result.stderr}")

# Load the model
model = load_dae_ir_json(result.stdout)

# Analyze
blt = blt_sort(model)

print(f"Model: {model.name}")
print(f"  States: {[s.name for s in model.states]}")
print(f"  Algebraic: {[a.name for a in model.algebraic_vars]}")
print(f"  Parameters: {[p.name for p in model.parameters]}")
print(f"  Well-posed: {blt.is_well_posed}")
print(f"  Algebraic loops: {blt.has_algebraic_loops}")
```

## API Reference

### Loading Functions

- `import_dae_ir(path: str) -> Model` - Load from file path
- `load_dae_ir_json(json_str: str) -> Model` - Load from JSON string
- `export_dae_ir(model: Model) -> dict` - Export model to dict

### Validation Functions

- `validate_dae_ir(data: dict) -> list[str]` - Validate dict against schema
- `validate_dae_ir_file(path: str) -> list[str]` - Validate JSON file

### Model Properties

- `model.states` - List of state variables
- `model.algebraic_vars` - List of algebraic variables
- `model.parameters` - List of parameters
- `model.equations` - List of equations
- `model.initial_equations` - List of initial equations
- `model.events` - List of events
- `model.n_states` - Number of states
- `model.n_algebraic` - Number of algebraic variables

### BLT Analysis

- `blt_sort(model) -> BLTResult` - Perform BLT analysis
- `result.blocks` - Ordered list of equation blocks
- `result.state_variables` - Set of state variable names
- `result.algebraic_variables` - Set of algebraic variable names
- `result.is_well_posed` - True if system is well-posed
- `result.has_algebraic_loops` - True if algebraic loops exist
