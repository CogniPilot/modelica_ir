# Base Modelica IR Examples

This directory contains example models in Base Modelica JSON format, demonstrating various features and use cases.

## Basic Examples

### [bouncing_ball_base.json](bouncing_ball_base.json)
Classic bouncing ball with inelastic collision.

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

## Annotation Structure

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

## Using These Examples

### 1. Import into Cyecca

```python
from cyecca.io import import_base_modelica

# Import SO(3) example
model = import_base_modelica("so3_quaternion_base.json")

# Check Lie group annotations
q_var = model.get_variable("q")
print(f"Lie group: {q_var.lie_group_type}")
print(f"Chart: {q_var.manifold_chart}")

# Generate simulation code
from cyecca.codegen import generate_simulation_code
code = generate_simulation_code(model, manifold_constraints=True)
```

### 2. Validate Against Schema

```bash
python ../tools/validate.py so3_quaternion_base.json
```

### 3. Modify for Your Application

Each example includes:
- Comprehensive comments explaining equations
- Source location tracking for debugging
- Metadata with references and use cases

Copy and adapt these examples for your specific application domain.

## Comparison Table

| Model | Lie Group | Dimension | Embedding | Primary Use Case |
|-------|-----------|-----------|-----------|------------------|
| Bouncing Ball | None (R²) | 2 | 2 | Simple event systems |
| SO(3) Quaternion | SO(3) | 3 | 4 | Attitude dynamics |
| SE(3) Rigid Body | SE(3) | 6 | 7 | 6-DOF rigid bodies |
| SE_2(3) UAV | SE_2(3) | 9 | 15 | Underactuated vehicles |

## Further Reading

- [Lie Group Annotations Guide](../docs/LIE_GROUP_ANNOTATIONS.md) - Comprehensive documentation
- [Base Modelica Alignment](../../cyecca/docs/BASE_MODELICA_ALIGNMENT.md) - Cyecca integration details
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

- **Cyecca**: Python library for code generation from Base Modelica
- **Rumoca**: Modelica compiler targeting Base Modelica IR
- **Manopt**: Optimization on manifolds (MATLAB/Python)
- **Pinocchio**: Rigid body dynamics library using Lie groups

## Contributing

To add a new example:

1. Create JSON file following Base Modelica schema
2. Validate: `python ../tools/validate.py your_example.json`
3. Add Lie group metadata if using manifold representations
4. Update this README with description and use cases
5. Submit pull request

Ensure examples:
- Are minimal and focused on specific features
- Include comprehensive comments
- Provide source location tracking
- Follow naming conventions (snake_case for files, camelCase for model names)
