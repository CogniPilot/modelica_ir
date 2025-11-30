# Schema Selection Guide

Which modelica_ir schema should you use? This guide helps you choose.

## Quick Decision Tree

```
Do you need connect equations preserved?
│
├─ YES → Use modelica_ir-0.2.0.schema.json
│
└─ NO
   │
   Do you need MCP-0031 Base Modelica compliance?
   │
   ├─ YES → Use base_modelica_ir-0.1.0.schema.json
   │
   └─ NO → Use modelica_ir-0.2.0.schema.json (more features)
```

## Schema Comparison

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

| Feature | modelica_ir v0.2.0 | base_modelica_ir base-0.1.0 |
|---------|--------------------|-----------------------------|
| **Connect Equations** | ✅ ConnectEquation | ❌ Must pre-expand |
| **If-Equations** | ✅ Balanced + Unbalanced | ⚠️ Balanced only |
| **When-Equations** | ✅ Full support | ✅ Full support |
| **For-Equations** | ✅ Full support | ✅ Full support |
| **Events (separate)** | ✅ Event type | ❌ Use when-equations |
| **Algorithms** | ✅ 9 statement types | ✅ 6 statement types |
| **Variables** | ✅ Unified list | ⚠️ Separated lists |
| **Variability** | ✅ Full (5 types) | ⚠️ Simplified (4 types) |
| **Source Tracking** | ⚠️ Optional | ✅ Required |
| **Operators** | ✅ 70+ operators | ⚠️ ~35 operators |
| **MCP-0031 Compliant** | ❌ No | ✅ Yes |
| **Modelica 3.7 Complete** | ✅ Yes | ❌ Subset |
| **eFMI Compatible** | ⚠️ Partial | ✅ Yes |

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

### Scenario 5: Rumoca → Cyecca Analysis
**Model:** Any Modelica model for reachability analysis

**Recommended:** `modelica_ir-0.2.0` initially, consider `base-0.1.0` for production

**Why:**
- v0.2.0 for development (more debugging info, preserves structure)
- base-0.1.0 for production (simpler, faster, guaranteed compatibility)

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

### Recommended for Cyecca v1.0
**Schema:** `modelica_ir-0.2.0` (Full Modelica)

**Reason:**
- Cyecca already has ComponentRef, FunctionCall(der/pre/edge)
- Supports for-equations and when-equations
- Adding if-equations and algorithms is straightforward
- More features = better analysis capabilities

### Future: Cyecca v2.0
**Additional Support:** `base_modelica_ir-0.1.0` (Base Modelica)

**Reason:**
- eFMI integration
- Multi-tool interoperability
- Educational use cases

---

## Rumoca Integration

### Recommended Export Strategy

**Option 1: Export Both**
- Rumoca exports both v0.2.0 and base-0.1.0
- User chooses at export time: `rumoca export --format=full|base`

**Option 2: Export Full, Convert**
- Rumoca exports v0.2.0 only
- Separate converter tool: `modelica_ir_convert --input=full --output=base`

**Option 3: Detect and Export**
- Rumoca detects if model uses connectors
- Automatically chooses v0.2.0 (has connects) or base-0.1.0 (no connects)

**Recommendation:** Start with **Option 1**, add **Option 2** later.

---

## Summary

| Use Case | Schema | Rationale |
|----------|--------|-----------|
| **Connector-based models** | v0.2.0 | Preserve connect semantics |
| **eFMI Equation Code** | base-0.1.0 | Direct compatibility |
| **Multi-tool exchange** | base-0.1.0 | MCP-0031 compliance |
| **Cyecca analysis** | v0.2.0 | More features, richer info |
| **Pure ODE/DAE** | Either | Choose based on other factors |
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

**Last Updated:** 2025-01-29
**Schema Versions:** v0.2.0, base-0.1.0
