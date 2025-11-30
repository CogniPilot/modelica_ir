# Lie Group Annotations in Base Modelica

This document describes best practices for annotating Lie group structures in Base Modelica JSON files. These annotations enable manifold-aware simulation of systems with configuration spaces that are not simple Euclidean spaces, such as rigid body dynamics, attitude control, and geometric mechanics.

## Table of Contents

1. [Overview](#overview)
2. [Supported Lie Groups](#supported-lie-groups)
3. [Annotation Structure](#annotation-structure)
4. [Variable Annotations](#variable-annotations)
5. [Model Metadata](#model-metadata)
6. [Examples](#examples)
7. [Solver Considerations](#solver-considerations)
8. [Best Practices](#best-practices)

## Overview

Lie groups are smooth manifolds with a group structure, commonly used to represent:
- **Rotations**: SO(2), SO(3) - Special Orthogonal Groups
- **Rigid body poses**: SE(2), SE(3) - Special Euclidean Groups
- **Extended poses**: SE_2(3) - Extended Special Euclidean Group (includes velocity)

### Why Use Lie Group Annotations?

1. **Avoid singularities**: Euler angles have gimbal lock; quaternions/rotation matrices don't
2. **Geometric integration**: Preserve group structure during numerical integration
3. **Physically consistent**: Constraints (e.g., unit quaternion norm) maintained automatically
4. **Efficient computation**: Native matrix/quaternion operations are optimized
5. **Control design**: Geometric controllers exploit manifold structure

## Supported Lie Groups

### SO(3) - Special Orthogonal Group in 3D

**Description**: 3D rotations (orientation/attitude)

**Dimension**: 3 (manifold dimension)

**Embedding Dimension**:
- 4 (quaternion representation)
- 9 (rotation matrix representation)

**Charts**:
- `quaternion`: Unit quaternion [w, x, y, z] with constraint w² + x² + y² + z² = 1
- `rotation_matrix`: 3×3 orthogonal matrix R with R^T R = I, det(R) = 1

**Tangent Space**: so(3) - angular velocity vectors ω ∈ R³

**Use Cases**: Attitude dynamics, gyroscope modeling, spacecraft orientation

### SE(3) - Special Euclidean Group in 3D

**Description**: 6-DOF rigid body poses (position + orientation)

**Dimension**: 6 (3 translation + 3 rotation)

**Embedding Dimension**: 7 (position R³ + quaternion S³) or 12 (position R³ + rotation matrix SO(3))

**Structure**: Semidirect product SE(3) = R³ ⋊ SO(3)

**Charts**:
- `position_quaternion`: [r, q] where r ∈ R³, q ∈ S³
- `homogeneous_matrix`: 4×4 transformation matrix

**Tangent Space**: se(3) - twist coordinates [v, ω] where v ∈ R³ (linear velocity), ω ∈ R³ (angular velocity)

**Use Cases**: Robot kinematics, rigid body dynamics, multi-body systems

### SE_2(3) - Extended Special Euclidean Group

**Description**: Extended pose for underactuated systems (position + orientation + velocity)

**Dimension**: 9 (3 position + 3 rotation + 3 velocity)

**Embedding Dimension**: 15 (5×3 matrix representation)

**Structure**: Extension of SE(3) with velocity coordinate

**Charts**:
- `extended_pose_matrix`: 5×3 matrix [p; R; v] where p, v ∈ R³, R ∈ SO(3)

**Tangent Space**: se_2(3) - [v_b; ω_skew; a_b] in body frame

**Use Cases**: UAV control, underactuated aerial vehicles, thrust-coupled systems

## Annotation Structure

Lie group information is stored in two places:

1. **Variable annotations**: Attached to individual variables
2. **Model metadata**: Global Lie group definitions in `metadata.lie_groups`

### Variable Annotations

```json
{
  "name": "q",
  "vartype": "Real",
  "shape": [4],
  "annotations": {
    "lie_group": "SO3",
    "manifold_chart": "quaternion",
    "convention": "scalar_first",
    "description": "Unit quaternion on SO(3) manifold"
  }
}
```

### Model Metadata

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
            "constraint": "q[1]^2 + q[2]^2 + q[3]^2 + q[4]^2 = 1"
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

## Variable Annotations

### Required Fields for Lie Group Variables

#### `lie_group` (string)
The name of the Lie group this variable belongs to. Must match a key in `metadata.lie_groups`.

**Example**: `"SO3"`, `"SE3"`, `"SE_2_3"`

#### `manifold_chart` (string)
The coordinate chart used to represent points on the manifold.

**Common charts**:
- SO(3): `"quaternion"`, `"rotation_matrix"`, `"axis_angle"`
- SE(3): `"position_quaternion"`, `"homogeneous_matrix"`
- SE_2(3): `"extended_pose_matrix"`

### Optional Fields

#### `convention` (string)
Specifies conventions for ambiguous representations.

**Quaternion conventions**:
- `"scalar_first"`: [w, x, y, z] (default for many libraries)
- `"scalar_last"`: [x, y, z, w] (used by some robotics frameworks)

#### `coordinate_frame` (string)
Reference frame for vector quantities.

**Values**: `"world"`, `"body"`, `"local"`, `"inertial"`

**Example**:
```json
{
  "name": "v",
  "annotations": {
    "coordinate_frame": "world",
    "description": "Linear velocity in world frame"
  }
}
```

#### `tangent_space` (string)
For velocity variables, specifies which Lie algebra they belong to.

**Example**:
```json
{
  "name": "omega",
  "annotations": {
    "tangent_space": "so3",
    "description": "Angular velocity (element of so(3))"
  }
}
```

## Model Metadata

The `metadata.lie_groups` section provides comprehensive information about each Lie group used in the model.

### Metadata Schema

```json
{
  "lie_groups": {
    "GROUP_NAME": {
      "description": "Human-readable description",
      "dimension": <int>,
      "embedding_dimension": <int>,
      "charts": {
        "CHART_NAME": {
          "coordinates": ["var1", "var2", ...],
          "constraint": "mathematical constraint expression",
          "description": "Chart description"
        }
      },
      "tangent_space": {
        "name": "lie_algebra_name",
        "coordinates": ["velocity_var1", ...],
        "description": "Tangent space description"
      }
    }
  }
}
```

### Field Descriptions

#### `description` (string, required)
Human-readable description of the Lie group and its physical interpretation.

#### `dimension` (integer, required)
Dimension of the manifold (degrees of freedom).

**Examples**:
- SO(3): 3 (three rotation angles)
- SE(3): 6 (three translations + three rotations)
- SE_2(3): 9 (three position + three rotation + three velocity)

#### `embedding_dimension` (integer, required)
Dimension of the ambient space where the manifold is embedded.

**Examples**:
- SO(3) quaternion: 4 (four quaternion components)
- SE(3) position+quaternion: 7 (three position + four quaternion)

#### `charts` (object, required)
Dictionary of coordinate charts (local parameterizations) for the manifold.

Each chart contains:
- `coordinates`: List of variable names using this chart
- `constraint`: Mathematical constraint defining the manifold (e.g., `"||q|| = 1"`)
- `description`: Human-readable chart description

#### `tangent_space` (object, required)
Description of the Lie algebra (tangent space at identity).

Contains:
- `name`: Name of the Lie algebra (e.g., `"so3"`, `"se3"`)
- `coordinates`: List of velocity variable names
- `description`: Physical interpretation of tangent vectors

### Optional Metadata Fields

#### `semidirect_product` (object)
For groups that are semidirect products (e.g., SE(3) = R³ ⋊ SO(3)).

```json
{
  "semidirect_product": {
    "translation": "R3",
    "rotation": "SO3"
  }
}
```

#### `extended_product` (object)
For extended groups like SE_2(3).

```json
{
  "extended_product": {
    "base_group": "SE3",
    "velocity_space": "R3"
  }
}
```

#### `applications` (object)
Use cases and benefits of this representation.

```json
{
  "applications": {
    "primary": "Quadrotor UAV control",
    "benefits": [
      "No gimbal lock",
      "Geometric integration",
      "Physically consistent constraints"
    ]
  }
}
```

#### `references` (array)
Academic references for the mathematical formulation.

```json
{
  "references": [
    {
      "title": "Paper title",
      "authors": "Author names",
      "year": 2020,
      "note": "Additional notes"
    }
  ]
}
```

## Examples

### Example 1: SO(3) Quaternion Variable

```json
{
  "variables": [
    {
      "name": "q",
      "vartype": "Real",
      "shape": [4],
      "start": [1.0, 0.0, 0.0, 0.0],
      "annotations": {
        "lie_group": "SO3",
        "manifold_chart": "quaternion",
        "convention": "scalar_first",
        "description": "Attitude quaternion [w, x, y, z]"
      }
    },
    {
      "name": "omega",
      "vartype": "Real",
      "shape": [3],
      "unit": "rad/s",
      "annotations": {
        "tangent_space": "so3",
        "coordinate_frame": "body",
        "description": "Angular velocity in body frame"
      }
    }
  ]
}
```

### Example 2: SE(3) Rigid Body

```json
{
  "variables": [
    {
      "name": "r",
      "vartype": "Real",
      "shape": [3],
      "unit": "m",
      "annotations": {
        "coordinate_frame": "world",
        "description": "Position in world frame (SE(3) translation component)"
      }
    },
    {
      "name": "q",
      "vartype": "Real",
      "shape": [4],
      "annotations": {
        "lie_group": "SO3",
        "manifold_chart": "quaternion",
        "convention": "scalar_first",
        "description": "Orientation (SE(3) rotation component)"
      }
    }
  ],
  "metadata": {
    "lie_groups": {
      "SE3": {
        "description": "6-DOF rigid body pose",
        "dimension": 6,
        "embedding_dimension": 7,
        "semidirect_product": {
          "translation": "R3",
          "rotation": "SO3"
        },
        "charts": {
          "position_quaternion": {
            "coordinates": ["r", "q"],
            "constraints": ["q[1]^2 + q[2]^2 + q[3]^2 + q[4]^2 = 1"]
          }
        }
      }
    }
  }
}
```

### Example 3: SE_2(3) UAV

See [se23_uav_base.json](../examples/se23_uav_base.json) for a complete example.

## Solver Considerations

### Constraint Manifold Integration

Solvers must handle constraint manifolds to maintain group structure:

1. **Projection Methods**: Project solution back to manifold after each step
   - For quaternions: normalize after integration
   - For rotation matrices: apply Gram-Schmidt orthogonalization

2. **Constrained Integration**: Use DAE solvers with algebraic constraints
   - Add constraint equations: `q[1]^2 + q[2]^2 + q[3]^2 + q[4]^2 - 1 = 0`
   - Solver maintains constraint during integration

3. **Geometric Integrators**: Specialized methods preserving Lie group structure
   - Runge-Kutta-Munthe-Kaas (RKMK) methods
   - Lie group exponential integrators
   - Crouch-Grossman methods

### Recommended Solver Settings

```python
# Example: DAE solver with quaternion constraint
from cyecca.solvers import DAESolver

solver = DAESolver(
    model=model,
    manifold_projection=True,  # Enable projection to constraint manifold
    projection_frequency=1,     # Project every step
    constraint_tolerance=1e-12  # Tight constraint tolerance
)
```

## Best Practices

### 1. Always Specify Conventions

Quaternion conventions vary across libraries. Always specify:

```json
{
  "annotations": {
    "lie_group": "SO3",
    "manifold_chart": "quaternion",
    "convention": "scalar_first"  // Explicit convention
  }
}
```

### 2. Document Coordinate Frames

Velocities and forces can be in different frames. Always document:

```json
{
  "name": "v",
  "annotations": {
    "coordinate_frame": "world",
    "description": "Linear velocity in world frame"
  }
},
{
  "name": "omega",
  "annotations": {
    "coordinate_frame": "body",
    "description": "Angular velocity in body frame"
  }
}
```

### 3. Link Variables to Charts

In metadata, explicitly list which variables use each chart:

```json
{
  "charts": {
    "quaternion": {
      "coordinates": ["q"],  // List all variables using this chart
      "constraint": "q[1]^2 + q[2]^2 + q[3]^2 + q[4]^2 = 1"
    }
  }
}
```

### 4. Provide Mathematical Constraints

State constraints clearly for solver configuration:

```json
{
  "charts": {
    "quaternion": {
      "constraint": "q[1]^2 + q[2]^2 + q[3]^2 + q[4]^2 = 1",
      "normalization_required": true
    }
  }
}
```

### 5. Include References

For non-standard Lie groups, provide references:

```json
{
  "metadata": {
    "references": [
      {
        "title": "Geometric Control of UAVs on SE(2,3)",
        "authors": "Lee, Leok, McClamroch",
        "year": 2013,
        "doi": "10.1109/TAC.2013.2258495"
      }
    ]
  }
}
```

### 6. Use Descriptive Names

Choose variable names that reflect their geometric meaning:

**Good**:
```json
{"name": "q_attitude", "annotations": {"lie_group": "SO3"}}
{"name": "omega_body", "annotations": {"tangent_space": "so3", "coordinate_frame": "body"}}
```

**Avoid**:
```json
{"name": "x", "annotations": {"lie_group": "SO3"}}
{"name": "v1", "annotations": {"tangent_space": "so3"}}
```

### 7. Structure Metadata for Reusability

Define reusable Lie group metadata that can be referenced across multiple models:

```json
{
  "metadata": {
    "lie_groups": {
      "SO3": {
        "description": "3D rotation group",
        "dimension": 3,
        "embedding_dimension": 4,
        "charts": {
          "quaternion": {
            "coordinates": ["q"],
            "constraint": "||q|| = 1",
            "normalization_required": true
          }
        },
        "tangent_space": {
          "name": "so3",
          "coordinates": ["omega"],
          "description": "Angular velocity vectors"
        }
      }
    }
  }
}
```

### 8. Validate Constraints

Include validation equations to check constraint satisfaction:

```json
{
  "equations": [
    {
      "eq_type": "simple",
      "lhs": {"op": "var", "name": "quaternion_norm_error"},
      "rhs": {
        "op": "-",
        "args": [
          {
            "op": "function_call",
            "name": "norm",
            "args": [{"op": "var", "name": "q"}]
          },
          {"op": "literal", "value": 1.0}
        ]
      },
      "comment": "Quaternion constraint violation (should be zero)"
    }
  ]
}
```

### 9. Distinguish Configuration and Velocity Variables

Use clear annotations to distinguish:
- **Configuration variables**: Elements of the Lie group (use `lie_group` annotation)
- **Velocity variables**: Elements of the Lie algebra (use `tangent_space` annotation)

```json
{
  "name": "q",
  "annotations": {
    "lie_group": "SO3",           // Configuration
    "manifold_chart": "quaternion"
  }
},
{
  "name": "omega",
  "annotations": {
    "tangent_space": "so3",        // Velocity
    "coordinate_frame": "body"
  }
}
```

### 10. Test Round-Trip Conversion

Verify that Lie group annotations survive import/export cycles:

```python
from cyecca.io import import_base_modelica, export_base_modelica

# Import model with Lie group annotations
model = import_base_modelica("so3_model.json")

# Export to new file
export_base_modelica(model, "so3_roundtrip.json")

# Re-import and verify
model2 = import_base_modelica("so3_roundtrip.json")

# Check annotations preserved
assert model2.get_variable("q").lie_group_type == "SO3"
assert model2.get_variable("q").manifold_chart == "quaternion"
```

## Tooling Support

### Cyecca Support

Cyecca provides full support for Lie group annotations:

1. **Import/Export**: Preserves all Lie group metadata through JSON round-trips
2. **Code Generation**: Generates manifold-aware simulation code
3. **Validation**: Checks constraint satisfaction and metadata consistency
4. **Geometric Integrators**: Optional solvers respecting Lie group structure

### Example Workflow

```python
from cyecca.io import import_base_modelica
from cyecca.codegen import generate_simulation_code
from cyecca.solvers import GeometricDAESolver

# Import model with Lie group annotations
model = import_base_modelica("se3_rigid_body.json")

# Generate simulation code with manifold awareness
code = generate_simulation_code(
    model,
    solver=GeometricDAESolver,
    manifold_constraints=True
)

# Run simulation
result = code.simulate(t_span=(0, 10), dt=0.01)
```

## See Also

- [Base Modelica Alignment Guide](../../cyecca/docs/BASE_MODELICA_ALIGNMENT.md)
- [SO(3) Example](../examples/so3_quaternion_base.json)
- [SE(3) Example](../examples/se3_rigid_body_base.json)
- [SE_2(3) Example](../examples/se23_uav_base.json)

## References

1. **Lie Groups for Computer Vision** - Ethan Eade (2013)
   - Excellent introduction to SO(3) and SE(3) for practitioners

2. **A micro Lie theory for state estimation in robotics** - Sola et al. (2018)
   - Practical guide to Lie group methods in robotics
   - arXiv:1812.01537

3. **Geometric Numerical Integration** - Hairer, Lubich, Wanner (2006)
   - Comprehensive treatment of structure-preserving integrators

4. **Geometric Control of Mechanical Systems** - Bullo & Lewis (2004)
   - Geometric mechanics and control on manifolds

5. **An Introduction to Lie Groups and Lie Algebras** - Kirillov (2008)
   - Mathematical foundations

6. **Geometric Tracking Control of a Quadrotor UAV on SE(3)** - Lee, Leok, McClamroch (2010)
   - Application to UAV control using SE(3)
