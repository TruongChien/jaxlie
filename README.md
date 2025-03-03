# jaxlie

![build](https://github.com/brentyi/jaxlie/workflows/build/badge.svg)
![mypy](https://github.com/brentyi/jaxlie/workflows/mypy/badge.svg?branch=master)
![lint](https://github.com/brentyi/jaxlie/workflows/lint/badge.svg)
[![codecov](https://codecov.io/gh/brentyi/jaxlie/branch/master/graph/badge.svg)](https://codecov.io/gh/brentyi/jaxlie)

**[ [API reference](https://brentyi.github.io/jaxlie) ]** **[
[PyPI](https://pypi.org/project/jaxlie/) ]**

`jaxlie` is a library containing implementations of Lie groups commonly used for
rigid body transformations, targeted at computer vision &amp; robotics
applications written in JAX. Heavily inspired by the C++ library
[Sophus](https://github.com/strasdat/Sophus).

We implement Lie groups as high-level (data)classes:

<table>
  <thead>
    <tr>
      <th>Group</th>
      <th>Description</th>
      <th>Parameterization</th>
    </tr>
  </thead>
  <tbody valign="top">
    <tr>
      <td><code>jaxlie.<strong>SO2</strong></code></td>
      <td>Rotations in 2D.</td>
      <td><em>(real, imaginary):</em> unit complex (∈ S<sup>1</sup>)</td>
    </tr>
    <tr>
      <td><code>jaxlie.<strong>SE2</strong></code></td>
      <td>Proper rigid transforms in 2D.</td>
      <td><em>(real, imaginary, x, y):</em> unit complex &amp; translation</td>
    </tr>
    <tr>
      <td><code>jaxlie.<strong>SO3</strong></code></td>
      <td>Rotations in 3D.</td>
      <td><em>(qw, qx, qy, qz):</em> wxyz quaternion (∈ S<sup>3</sup>)</td>
    </tr>
    <tr>
      <td><code>jaxlie.<strong>SE3</strong></code></td>
      <td>Proper rigid transforms in 3D.</td>
      <td><em>(qw, qx, qy, qz, x, y, z):</em> wxyz quaternion &amp; translation</td>
    </tr>
  </tbody>
</table>

Where each group supports:

- Forward- and reverse-mode AD-friendly **`exp()`**, **`log()`**,
  **`adjoint()`**, **`apply()`**, **`multiply()`**, **`inverse()`**,
  **`identity()`**, **`from_matrix()`**, and **`as_matrix()`** operations. (see
  [./examples/se3_example.py](./examples/se3_basics.py))
- Helpers for optimization on manifolds (see
  [./examples/se3_optimization.py](./examples/se3_optimization.py),
  <code>jaxlie.<strong>manifold.\*</strong></code>).
- Compatibility with standard JAX function transformations. (see
  [./examples/vmap_example.py](./examples/vmap_example.py))
- (Un)flattening as pytree nodes.
- Serialization using [flax](https://github.com/google/flax).

We also implement various common utilities for things like uniform random
sampling (**`sample_uniform()`**) and converting from/to Euler angles (in the
`SO3` class).

---

#### Install (Python >=3.7)

```bash
# Python 3.6 releases also exist, but are no longer being updated.
pip install jaxlie
```

---

#### Example usage for SE(3)

```python
import numpy as onp

from jaxlie import SE3

#############################
# (1) Constructing transforms.
#############################

# We can compute a w<-b transform by integrating over an se(3) screw, equivalent
# to `SE3.from_matrix(expm(wedge(twist)))`.
twist = onp.array([1.0, 0.0, 0.2, 0.0, 0.5, 0.0])
T_w_b = SE3.exp(twist)

# We can print the (quaternion) rotation term; this is an `SO3` object:
print(T_w_b.rotation())

# Or print the translation; this is a simple array with shape (3,):
print(T_w_b.translation())

# Or the underlying parameters; this is a length-7 (quaternion, translation) array:
print(T_w_b.wxyz_xyz)  # SE3-specific field.
print(T_w_b.parameters())  # Helper shared by all groups.

# There are also other helpers to generate transforms, eg from matrices:
T_w_b = SE3.from_matrix(T_w_b.as_matrix())

# Or from explicit rotation and translation terms:
T_w_b = SE3.from_rotation_and_translation(
    rotation=T_w_b.rotation(),
    translation=T_w_b.translation(),
)

# Or with the dataclass constructor + the underlying length-7 parameterization:
T_w_b = SE3(wxyz_xyz=T_w_b.wxyz_xyz)


#############################
# (2) Applying transforms.
#############################

# Transform points with the `@` operator:
p_b = onp.random.randn(3)
p_w = T_w_b @ p_b
print(p_w)

# or `.apply()`:
p_w = T_w_b.apply(p_b)
print(p_w)

# or the homogeneous matrix form:
p_w = (T_w_b.as_matrix() @ onp.append(p_b, 1.0))[:-1]
print(p_w)


#############################
# (3) Composing transforms.
#############################

# Compose transforms with the `@` operator:
T_b_a = SE3.identity()
T_w_a = T_w_b @ T_b_a
print(T_w_a)

# or `.multiply()`:
T_w_a = T_w_b.multiply(T_b_a)
print(T_w_a)


#############################
# (4) Misc.
#############################

# Compute inverses:
T_b_w = T_w_b.inverse()
identity = T_w_b @ T_b_w
print(identity)

# Compute adjoints:
adjoint_T_w_b = T_w_b.adjoint()
print(adjoint_T_w_b)

# Recover our twist, equivalent to `vee(logm(T_w_b.as_matrix()))`:
twist_recovered = T_w_b.log()
print(twist_recovered)
```
