# Schema Selection Guide

Which modelica_ir schema should you use? This guide helps you choose.

## Quick Decision Tree

```
Are you building a simulation backend or code generator?
│
├─ YES → Use dae_ir-0.1.0.schema.json ⭐ (explicit DAE structure)
│
└─ NO
   │
   Do you need connect equations preserved?
   │
   ├─ YES → Use modelica_ir-0.2.0.schema.json
   │
   └─ NO
      │
      Do you need strict MCP-0031 compliance?
      │
      ├─ YES → Use base_modelica_ir-0.1.0.schema.json
      │
      └─ NO → Use dae_ir-0.1.0.schema.json ⭐ (recommended default)
```

## Schema Comparison

### dae_ir-0.1.0.schema.json (DAE IR) ⭐ Recommended

**Use When:**
- ✅ Building simulation backends or code generators
- ✅ Want explicit state/algebraic classification without inference
- ✅ Need efficient BLT analysis and structural processing
- ✅ Want classified variables and equations in structured objects
- ✅ Targeting ODE/DAE solver integration

**Advantages:**
- **Explicit DAE structure** matching Modelica Spec Appendix B
- **Classified variables** - States, algebraic, discrete, parameters in separate arrays
- **Classified equations** - Continuous, event, discrete, initial in separate arrays
- **Event indicators** - Zero-crossing functions for hybrid systems
- **Structural metadata** - n_states, n_algebraic, n_equations, dae_index, is_ode
- **State indices** - Each state has `state_index` for direct mapping to solver vectors
- **Extends Base Modelica** - Same expression syntax, adds classification structure

**Disadvantages:**
- Requires compiler to perform variable classification
- Slightly larger file size due to additional metadata

**Example Use Cases:**
- Rumoca → Cyecca simulation pipeline
- JAX/NumPy/C code generation
- BLT sorting and causality analysis
- Index reduction algorithms
- FMU generation

---

### modelica_ir-0.2.0.schema.json (Full Modelica 3.7)

**Use When:**
- ✅ Working with connector-based models (electrical, hydraulic, mechanical)
- ✅ Need to preserve connect() semantics
- ✅ Want maximum Modelica 3.7 feature coverage
- ✅ Debugging and inspection (human-readable connect info)
- ✅ Rumoca exporting rich pre-flattening information

**Advantages:**
- Complete Modelica 3.7 representation
- Preserves high-level structure
- Rich metadata support
- Unbalanced if-equations allowed (more flexible)

**Disadvantages:**
- More complex (more features to handle)
- Not guaranteed Base Modelica compliant
- May require backend to expand connects

**Example Use Cases:**
- Electrical circuit analysis with resistors/capacitors/connectors
- Hydraulic systems with flow and pressure connectors
- Mechanical linkages with flanges
- Multi-physics systems

---

### base_modelica_ir-0.1.0.schema.json (MCP-0031 Base Modelica)

**Use When:**
- ✅ Targeting eFMI Equation Code
- ✅ Maximum tool interoperability required
- ✅ Simulation backend doesn't need connect semantics
- ✅ Guaranteed MCP-0031 compliance needed
- ✅ Simplified, normalized models preferred

**Advantages:**
- Guaranteed Base Modelica MCP-0031 compliance
- Simpler structure (fewer features to handle)
- Source location tracking built-in
- eFMI compatible
- Better for educational/inspection purposes

**Disadvantages:**
- Connects must be pre-expanded (loses high-level structure)
- Only balanced if-equations (more restrictive)
- Separated variable lists (constants/parameters/variables)

**Example Use Cases:**
- eFMI Equation Code generation
- Portable models for multiple tools
- Educational model introspection
- Third-party analysis tools
- Pure DAE systems without connectors

---

## Feature Matrix

| Feature | dae_ir ⭐ | modelica_ir v0.2.0 | base_modelica_ir |
|---------|-----------|--------------------|--------------------|
| **Explicit State Classification** | ✅ states/algebraic arrays | ❌ Must infer | ❌ Must infer |
| **State Indices** | ✅ state_index field | ❌ No | ❌ No |
| **Event Indicators** | ✅ Zero-crossings | ❌ No | ❌ No |
| **Structural Metadata** | ✅ n_states, n_algebraic, is_ode | ❌ No | ❌ No |
| **Equation Classification** | ✅ continuous/event/discrete/initial | ⚠️ Unified | ⚠️ Unified |
| **Derivatives as der(x)** | ✅ Standard syntax | ✅ Standard syntax | ✅ Standard syntax |
| **Connect Equations** | ❌ Must pre-expand | ✅ ConnectEquation | ❌ Must pre-expand |
| **If-Equations** | ⚠️ Balanced only | ✅ Balanced + Unbalanced | ⚠️ Balanced only |
| **When-Equations** | ✅ Full support | ✅ Full support | ✅ Full support |
| **For-Equations** | ✅ Full support | ✅ Full support | ✅ Full support |
| **Algorithms** | ✅ 7 statement types | ✅ 9 statement types | ✅ 6 statement types |
| **Source Tracking** | ✅ Optional | ⚠️ Optional | ✅ Required |
| **Extends Base Modelica** | ✅ Superset | ❌ No | ✅ Reference |
| **Modelica Spec App B** | ✅ Yes | ❌ No | ❌ No |
| **Simulation-Ready** | ✅ Direct solver mapping | ❌ Needs processing | ❌ Needs processing |

Legend: ✅ = Fully supported, ⚠️ = Partial/Different, ❌ = Not supported

---

## Common Scenarios

### Scenario 1: Electrical Circuit Simulation
**Model:** Resistor-Capacitor circuit with `connect()` statements

**Recommended:** `modelica_ir-0.2.0`

**Why:** Preserves connector semantics, making it easier to understand circuit topology.

**Alternative:** Use `base_modelica_ir-0.1.0` if you have a tool that expands connects first.

---

### Scenario 2: eFMI Equation Code Generation
**Model:** Any Modelica model

**Recommended:** `base_modelica_ir-0.1.0`

**Why:** Direct compatibility with eFMI Equation Code, which is based on Base Modelica.

**Alternative:** Convert from `v0.2.0` → `base-0.1.0` using converter tool (future).

---

### Scenario 3: Pure ODE/DAE System (No Connectors)
**Model:** Bouncing ball, mass-spring-damper, pendulum

**Recommended:** Either schema works, `modelica_ir-0.2.0` has more features

**Why:** No connectors means no advantage to Base Modelica's restrictions.

**Alternative:** Use `base_modelica_ir-0.1.0` for guaranteed interoperability.

---

### Scenario 4: Multi-Tool Workflow
**Model:** Need to exchange models between multiple tools

**Recommended:** `base_modelica_ir-0.1.0`

**Why:** MCP-0031 compliance guarantees all conforming tools can read it.

**Alternative:** Use `v0.2.0` if all tools support it (check compatibility).

---

### Scenario 5: Rumoca → Cyecca Simulation Pipeline ⭐
**Model:** Any Modelica model for simulation

**Recommended:** `dae_ir-0.1.0` (DAE IR)

**Why:**
- Explicit state/algebraic classification in separate arrays
- Derivatives written as `der(x)` in equations (standard Modelica syntax)
- Direct mapping to ODE/DAE solver APIs
- BLT analysis is more efficient with pre-classified variables
- Event indicators ready for hybrid simulation
- Structural metadata (n_states, n_algebraic, is_ode) for solver selection

---

## Migration Path

### From v0.1.0 → v0.2.0 (Full Modelica)

**Required Changes:**
1. Add `eq_type` to all equations
2. Convert flat variable references to `component_ref` with `parts`
3. Convert statement targets to component reference arrays
4. Add initial_equations/initial_algorithms fields

**Effort:** Medium (breaking changes but straightforward)

### From v0.1.0 → base-0.1.0 (Base Modelica)

**Required Changes:**
1. Add `eq_type` to all equations
2. Separate variables into constants/parameters/variables
3. Ensure all if-equations are balanced (add else branches)
4. Remove connect equations (expand to explicit coupling)
5. Add source tracking (source_ref + source_info)

**Effort:** High (more semantic changes required)

### From v0.2.0 → base-0.1.0 (Downgrade)

**Required Changes:**
1. Expand connect equations to explicit variable coupling
2. Balance all if-equations (add else branches)
3. Separate variables by variability
4. Remove event types (convert to when-equations)
5. Add source tracking
6. Simplify variability values

**Effort:** High (semantic transformations needed)

**Tool:** Converter utility (planned, not yet implemented)

---

## Cyecca Integration

### Recommended for Cyecca
**Schema:** `dae_ir-0.1.0` (DAE IR) ⭐

**Reason:**
- Explicit state/algebraic classification in separate arrays
- Derivatives written as `der(x)` in equations (standard Modelica syntax)
- Direct mapping to ODE/DAE solver APIs (scipy, JAX, etc.)
- BLT analysis is more efficient with pre-classified variables
- Event indicators ready for hybrid simulation
- Structural metadata enables automatic solver selection

### Legacy Support
**Schema:** `base_modelica_ir-0.1.0` (Base Modelica)

**Reason:**
- MCP-0031 compliance for tool interoperability
- eFMI integration
- Educational use cases

---

## Rumoca Integration

### Recommended Export Strategy

**Primary:** `dae_ir-0.1.0` (DAE IR) ⭐
- Rumoca performs variable classification during flattening
- Exports explicit state/algebraic arrays with state indices
- Derivatives written as `der(x)` in equations
- Generates event indicators from when-conditions
- Usage: `rumoca model.mo --model=ModelName --json`

**Note:** Rumoca currently only exports DAE IR format. Base Modelica export is planned for future releases.

---

## Summary

| Use Case | Schema | Rationale |
|----------|--------|-----------|
| **Simulation/Code Generation** ⭐ | dae-0.1.0 | Explicit DAE structure, no inference needed |
| **Rumoca → Cyecca Pipeline** ⭐ | dae-0.1.0 | Direct solver mapping, efficient BLT |
| **ODE/DAE Solvers** | dae-0.1.0 | State indices, event indicators |
| **Connector-based models** | v0.2.0 | Preserve connect semantics |
| **eFMI Equation Code** | base-0.1.0 | Direct compatibility |
| **Multi-tool exchange** | base-0.1.0 | MCP-0031 compliance |
| **Maximum compatibility** | base-0.1.0 | Guaranteed interoperability |
| **Maximum features** | v0.2.0 | Full Modelica 3.7 support |

---

## Need Help?

- **Schema Documentation:** See [README.md](README.md)
- **Version History:** See [SCHEMA_CHANGELOG.md](SCHEMA_CHANGELOG.md)
- **Base Modelica Details:** See [docs/BASE_MODELICA_ALIGNMENT.md](docs/BASE_MODELICA_ALIGNMENT.md)
- **Examples:** See [examples/](examples/) directory
- **Issues:** Report at [modelica_ir issues](https://github.com/cyecca/modelica_ir/issues)

---

**Last Updated:** 2025-11-30
**Schema Versions:** dae-0.1.0, v0.2.0, base-0.1.0
