# Modelica IR Examples

This directory contains example models in various JSON formats, demonstrating the different schema options.

## DAE IR Examples ⭐ Recommended

DAE IR extends Base Modelica with explicit variable and equation classification.

### [bouncing_ball_dae.json](bouncing_ball_dae.json)
Classic bouncing ball in DAE IR format with explicit DAE structure.

**Features demonstrated:**
- Explicit variable classification (states, algebraic, parameters)
- State indices for direct solver mapping
- Derivatives as `der(x)` in equations (standard Modelica syntax)
- Classified equations (continuous, event, initial)
- Event indicators for hybrid simulation
- Structural metadata (n_states, n_algebraic, is_ode)

**Physics:** Simple gravity + collision model with coefficient of restitution.

## Base Modelica IR Examples

Base Modelica IR follows MCP-0031 with flat variable arrays.

### [bouncing_ball_base.json](bouncing_ball_base.json)
Classic bouncing ball in Base Modelica format (MCP-0031 compliant).

**Features demonstrated:**
- Constants, parameters, and variables separation
- When-equations for discrete events
- Source tracking (line/column information)
- Reinit statements for state variable discontinuities

**Physics:** Simple gravity + collision model with coefficient of restitution.

## Lie Group Examples

These examples demonstrate manifold-aware modeling using Lie group annotations, which enable geometric integration and avoid coordinate singularities.

### [so3_quaternion_base.json](so3_quaternion_base.json)
Rotation dynamics on SO(3) using unit quaternions.

**Features demonstrated:**
- SO(3) Lie group annotations
- Quaternion kinematics (dq/dt = 0.5 * q ⊗ [0; ω])
- Tangent space (so(3)) angular velocity
- Manifold constraint (||q|| = 1)

**Use cases:** Attitude control, gyroscope modeling, spacecraft orientation

**Key insight:** Quaternions avoid gimbal lock inherent in Euler angles while maintaining computational efficiency.

### [se3_rigid_body_base.json](se3_rigid_body_base.json)
6-DOF rigid body dynamics on SE(3).

**Features demonstrated:**
- SE(3) Lie group (position + orientation)
- Semidirect product structure: SE(3) = R³ ⋊ SO(3)
- Mixed-frame velocities (world-frame linear, body-frame angular)
- Newton-Euler equations
- External forces and torques

**Use cases:** Robot kinematics, multi-body dynamics, spacecraft simulation

**Key insight:** SE(3) representation naturally decouples translation and rotation while preserving geometric structure.

### [se23_uav_base.json](se23_uav_base.json)
UAV dynamics on SE_2(3) extended pose manifold.

**Features demonstrated:**
- SE_2(3) Lie group (position + orientation + velocity)
- Extended pose matrix representation [p; R; v]
- Underactuated dynamics (thrust aligned with body z-axis)
- Right-invariant kinematics
- Body-frame velocities and accelerations
- Geometric control formulation

**Use cases:** Quadrotor control, fixed-wing UAV, thrust-vectored vehicles

**Key insight:** SE_2(3) captures the coupling between thrust direction and attitude, essential for underactuated aerial vehicles. This formulation enables geometric control design with global stability guarantees.

## Full Modelica IR Examples

Full Modelica IR (v0.2.0) provides comprehensive Modelica 3.7 support.

### [bouncing_ball_v0.2.json](bouncing_ball_v0.2.json)
Bouncing ball in full Modelica IR format.

### [bouncing_ball.json](bouncing_ball.json)
Legacy v0.1.0 format (deprecated).

## Schema Comparison

| File | Schema | Version | Use Case |
|------|--------|---------|----------|
| bouncing_ball_dae.json | DAE IR ⭐ | dae-0.1.0 | Simulation, code generation |
| bouncing_ball_base.json | Base Modelica | base-0.1.0 | MCP-0031 compliance |
| bouncing_ball_v0.2.json | Full Modelica | 0.2.0 | Rich analysis tools |
| *_base.json (Lie groups) | Base Modelica | base-0.1.0 | Manifold-aware dynamics |

## Using These Examples

### 1. Import into Cyecca

```python
from cyecca.io import import_dae_ir, load_dae_ir_json

# Import from file
model = import_dae_ir("bouncing_ball_dae.json")

# Or load from string
with open("bouncing_ball_dae.json") as f:
    model = load_dae_ir_json(f.read())

# Access classified variables directly
print(f"States: {[v.name for v in model.states]}")
print(f"Algebraic: {[v.name for v in model.algebraic_vars]}")
print(f"Parameters: {[v.name for v in model.parameters]}")

# Check Lie group annotations (if present)
q_var = model.get_variable("q")
if q_var and hasattr(q_var, 'lie_group_type'):
    print(f"Lie group: {q_var.lie_group_type}")
    print(f"Chart: {q_var.manifold_chart}")
```

### 2. Validate Against Schema

```bash
python ../tools/validate.py bouncing_ball_dae.json
```

### 3. Generate with Rumoca

```bash
# Compile Modelica to DAE IR JSON
rumoca bouncing_ball.mo --model=BouncingBall --json > bouncing_ball_dae.json
```

## DAE IR vs Base Modelica

| Aspect | DAE IR ⭐ | Base Modelica |
|--------|----------|---------------|
| **Variables** | Classified object (states, algebraic, etc.) | Flat arrays |
| **Equations** | Classified object (continuous, event, etc.) | Flat array |
| **Derivatives** | `der(x)` in equations | `der(x)` in equations |
| **State indices** | Explicit `state_index` field | Not available |
| **Structural metadata** | n_states, n_algebraic, is_ode | Not available |
| **Extends** | Base Modelica | MCP-0031 |

## Annotation Structure (Lie Groups)

All Lie group examples follow this pattern:

### 1. Variable Annotations
```json
{
  "name": "q",
  "vartype": "Real",
  "shape": [4],
  "annotations": {
    "lie_group": "SO3",
    "manifold_chart": "quaternion",
    "convention": "scalar_first",
    "description": "Attitude quaternion"
  }
}
```

### 2. Model Metadata
```json
{
  "metadata": {
    "lie_groups": {
      "SO3": {
        "description": "Special Orthogonal Group in 3D",
        "dimension": 3,
        "embedding_dimension": 4,
        "charts": {
          "quaternion": {
            "coordinates": ["q"],
            "constraint": "||q|| = 1"
          }
        },
        "tangent_space": {
          "name": "so3",
          "coordinates": ["omega"]
        }
      }
    }
  }
}
```

## Comparison Table

| Model | Lie Group | Dimension | Embedding | Primary Use Case |
|-------|-----------|-----------|-----------|------------------|
| Bouncing Ball | None (R²) | 2 | 2 | Simple event systems |
| SO(3) Quaternion | SO(3) | 3 | 4 | Attitude dynamics |
| SE(3) Rigid Body | SE(3) | 6 | 7 | 6-DOF rigid bodies |
| SE_2(3) UAV | SE_2(3) | 9 | 15 | Underactuated vehicles |

## Further Reading

- [Lie Group Annotations Guide](../docs/LIE_GROUP_ANNOTATIONS.md) - Comprehensive documentation
- [Schema Selection Guide](../SCHEMA_SELECTION_GUIDE.md) - Choosing the right schema
- [MCP-0031 Specification](https://github.com/modelica/ModelicaSpecification/blob/MCP/0031/RationaleMCP/0031/) - Base Modelica standard

## References

### Academic Papers

1. **Bullo & Murray (1999)** - "Proportional Derivative (PD) Control on the Euclidean Group"
   - Foundation of geometric control on Lie groups

2. **Lee, Leok, McClamroch (2010)** - "Geometric tracking control of a quadrotor UAV on SE(3)"
   - SE(3) formulation for UAV control

3. **Sola et al. (2018)** - "A micro Lie theory for state estimation in robotics"
   - Practical guide to Lie groups in robotics (arXiv:1812.01537)

### Software

- **Cyecca**: Python library for code generation from DAE IR
- **Rumoca**: Modelica compiler targeting DAE IR
- **Manopt**: Optimization on manifolds (MATLAB/Python)
- **Pinocchio**: Rigid body dynamics library using Lie groups

## Contributing

To add a new example:

1. Create JSON file following DAE IR schema (recommended) or Base Modelica schema
2. Validate: `python ../tools/validate.py your_example.json`
3. Add Lie group metadata if using manifold representations
4. Update this README with description and use cases
5. Submit pull request

Ensure examples:
- Are minimal and focused on specific features
- Include comprehensive comments
- Follow naming conventions (snake_case for files, camelCase for model names)
