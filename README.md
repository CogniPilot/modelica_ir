# modelica_ir

JSON schemas for Modelica intermediate representations.

## Schemas

| Schema | Description | Use Case |
|--------|-------------|----------|
| **Explicit IR** | Causal ODE: `dx/dt = f(x,u,p)` | Real-time code generation, embedded systems |
| **DAE IR** | Implicit DAE with classified variables | Simulation with DAE solvers, analysis |
| **Base Modelica IR** | MCP-0031 compliant flat format | Tool interoperability |
| **Full Modelica IR** | Complete Modelica 3.7 representation | IDE tools, analysis |

## Pipeline

```
Modelica source → Rumoca → Explicit IR (causal) or DAE IR (implicit) → Cyecca/Solver
```

- **Explicit IR**: Result of BLT + causalization. No algebraic loops. Ready for C code generation.
- **DAE IR**: Implicit models with algebraic constraints. Requires DAE solver.

## Explicit IR Example

```json
{
  "ir_version": "explicit-0.1.0",
  "model_name": "MassSpring",
  "states": [
    {"name": "x", "index": 0, "start": 1.0},
    {"name": "v", "index": 1, "start": 0.0}
  ],
  "parameters": [
    {"name": "k", "index": 0, "value": 1.0}
  ],
  "derivatives": [
    {"state_index": 0, "expr": {"op": "state_ref", "ref_index": 1}},
    {"state_index": 1, "expr": {"op": "*", "args": [{"op": "literal", "value": -1}, {"op": "param_ref", "ref_index": 0}, {"op": "state_ref", "ref_index": 0}]}}
  ]
}
```

## DAE IR Example

```json
{
  "ir_version": "dae-0.1.0",
  "model_name": "BouncingBall",
  "variables": {
    "states": [{"name": "h", "state_index": 0, "start": 1.0}],
    "algebraic": [],
    "parameters": [{"name": "g", "start": 9.81}]
  },
  "equations": {
    "continuous": [
      {"eq_type": "simple", "lhs": {"op": "der", "args": [{"op": "component_ref", "parts": [{"name": "h"}]}]}, "rhs": {"op": "component_ref", "parts": [{"name": "v"}]}}
    ]
  }
}
```

## Validation

```bash
python tools/validate.py examples/bouncing_ball_dae.json
```

## Related Projects

- [Rumoca](https://github.com/CogniPilot/rumoca) - Modelica compiler (produces these IRs)
- [Cyecca](https://github.com/CogniPilot/cyecca) - Simulation/analysis tool (consumes these IRs)

## License

See [LICENSE](LICENSE)
