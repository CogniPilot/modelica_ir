# Modelica IR Schema Changelog

## Version 0.2.0 (2025-01-XX)

### Breaking Changes

#### 1. Equation Structure - Now Discriminated Union
**Before (v0.1.0):**
```json
{
  "lhs": {...},
  "rhs": {...}
}
```

**After (v0.2.0):**
```json
{
  "eq_type": "simple",
  "lhs": {...},
  "rhs": {...}
}
```

All equations now have an `eq_type` field to distinguish between:
- `"simple"` - Simple equality: `lhs = rhs`
- `"for"` - For-loop equation
- `"if"` - Conditional equation (structural if)
- `"when"` - When-clause equation
- `"connect"` - Connect equation

#### 2. Expression Component References - Now Hierarchical
**Before (v0.1.0):**
```json
{
  "op": "var",
  "args": [],
  "var": "x"
}
```

**After (v0.2.0):**
```json
{
  "op": "component_ref",
  "parts": [
    {
      "name": "x",
      "subscripts": []
    }
  ]
}
```

The new `component_ref` operator supports hierarchical references:
```json
{
  "op": "component_ref",
  "parts": [
    {"name": "vehicle", "subscripts": []},
    {"name": "position", "subscripts": []},
    {"name": "x", "subscripts": []}
  ]
}
```

And array indexing:
```json
{
  "op": "component_ref",
  "parts": [
    {
      "name": "positions",
      "subscripts": [
        {"op": "literal", "value": 1}
      ]
    }
  ]
}
```

**Migration:** The old `"var"` operator is still supported for backward compatibility but deprecated.

#### 3. Statement Target - Now Component Reference
**Before (v0.1.0):**
```json
{
  "stmt": "assign",
  "target": "x",
  "expr": {...}
}
```

**After (v0.2.0):**
```json
{
  "stmt": "assign",
  "target": [
    {
      "name": "x",
      "subscripts": []
    }
  ],
  "expr": {...}
}
```

This allows assignments to hierarchical components and array elements.

### New Features

#### 1. Initial Sections
Added separate fields for initial equations and algorithms:
```json
{
  "equations": [...],
  "initial_equations": [...],
  "algorithms": [...],
  "initial_algorithms": [...]
}
```

#### 2. If-Equations (Structural Conditionals)
New equation type for structural if-then-else in equations:
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

#### 3. Connect Equations
New equation type for physical connections:
```json
{
  "eq_type": "connect",
  "lhs": [{"name": "pin1"}],
  "rhs": [{"name": "pin2"}]
}
```

#### 4. When Equations
New equation type for when-clauses (declarative events):
```json
{
  "eq_type": "when",
  "branches": [
    {
      "condition": {...},
      "equations": [...]
    }
  ]
}
```

#### 5. Additional Statement Types
- `WhileStatement` - While loops
- `WhenStatement` - When statements in algorithms
- `BreakStatement` - Break from loops
- `ReturnStatement` - Return from functions
- `FunctionCallStatement` - Call statement (assert, terminate, etc.)

#### 6. Enhanced Variable Attributes
Added Modelica 3.7 variable attributes:
- `display_unit` - Display unit for visualization
- `min` / `max` - Value constraints
- `nominal` - Nominal value for scaling
- `fixed` - Whether initial value is fixed
- `variability` - Variability classification
- `causality` - Causality (FMI terminology)
- `metadata` - Variable-level metadata object

#### 7. Enhanced Expression Operators
Added comprehensive Modelica operators:
- **Comparison:** `"!="` (not equal)
- **Element-wise array:** `".+"`, `".-"`, `".*"`, `"./"`, `".^"`
- **Modelica operators:** `"change"`, `"initial"`, `"terminal"`
- **Transcendental:** `"asin"`, `"acos"`, `"atan"`, `"atan2"`, `"sinh"`, `"cosh"`, `"tanh"`
- **Numeric:** `"floor"`, `"ceil"`, `"mod"`, `"rem"`, `"div"`
- **Array:** `"min"`, `"max"`, `"sum"`, `"product"`

#### 8. Enhanced Function Definitions
Extended `FunctionDef` with:
- `protected_vars` - Protected local variables
- `is_pure` - Purity flag (default: true)
- `external` - External function specification (C/Fortran)

#### 9. Variable Type Extensions
Added `"String"` to primitive types and `"stream"` to variable kinds.

#### 10. Model Metadata
Added `metadata` object at model level for annotations, Lie-group information, etc.

### Improvements

1. **Better Documentation**: All schema fields now have descriptions
2. **Schema ID**: Added `$id` field for schema referencing
3. **Validation Rules**: Added conditional validation with `allOf`/`if`/`then`
4. **Default Values**: Specified defaults for optional fields
5. **Constraints**: Added enums for all categorical fields

### Deprecated

- `Expr.var` - Use `component_ref` with single part instead
- Flat string targets in statements - Use component reference arrays instead

### Migration Guide

#### For Cyecca (Consumer)
1. Update IR classes to support new equation types
2. Add `ComponentRefPart` class
3. Handle `eq_type` discriminator in equations
4. Add support for initial sections
5. Implement new statement types

#### For Rumoca (Producer)
1. Update JSON export to include `eq_type` in all equations
2. Convert `ComponentReference` to `parts` array format
3. Export initial equations/algorithms to separate fields
4. Add variable attributes (unit, min, max, etc.)
5. Include metadata fields

### Backward Compatibility

The schema maintains partial backward compatibility:
- Old `"var"` operator still validates (though deprecated)
- Required fields remain the same (ir_version, model_name, variables, equations)
- Optional fields have defaults

However, the discriminated union for equations is a **breaking change**. All equations must now include `eq_type`.

### Example Migration

**v0.1.0:**
```json
{
  "ir_version": "0.1.0",
  "equations": [
    {
      "lhs": {"op": "der", "args": [{"op": "var", "var": "x"}]},
      "rhs": {"op": "var", "var": "v"}
    }
  ]
}
```

**v0.2.0:**
```json
{
  "ir_version": "0.2.0",
  "equations": [
    {
      "eq_type": "simple",
      "lhs": {
        "op": "der",
        "args": [
          {
            "op": "component_ref",
            "parts": [{"name": "x", "subscripts": []}]
          }
        ]
      },
      "rhs": {
        "op": "component_ref",
        "parts": [{"name": "v", "subscripts": []}]
      }
    }
  ],
  "initial_equations": []
}
```

## Version 0.1.0 (Initial Release)

- Basic flattened model representation
- Variables with primitive types
- Simple equations (lhs = rhs)
- Expressions with op/args tree structure
- Algorithm sections with statements
- Events (when clauses)
- Functions (basic)
- Connections (string-based)
