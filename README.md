![](/img/logo.png)

<p align="center">
    <img title="license" src="https://img.shields.io/badge/license-Apache_2.0-blue.svg">
    <img title="python" src="https://img.shields.io/badge/python-≥3.8-blue.svg">
</p>

<p align="center">
    <img title="test" src='https://github.com/SimoneGasperini/qiskit-symbolic/actions/workflows/test.yml/badge.svg?branch=master'>
    <img title="lint" src='https://github.com/SimoneGasperini/qiskit-symbolic/actions/workflows/lint.yml/badge.svg?branch=master'>
    <img title="coverage" src='https://coveralls.io/repos/github/SimoneGasperini/qiskit-symbolic/badge.svg?branch=master'>
</p>

***

# Table of contents
- [Introduction](#introduction)
- [Installation](#installation)
    - [User-mode](#user-mode)
    - [Dev-mode](#dev-mode)
- [Usage examples](#usage-examples)
    - [_Sympify_ a Qiskit circuit](#sympify-a-qiskit-circuit)
    - [_Lambdify_ a Qiskit circuit](#lambdify-a-qiskit-circuit)
- [Contributors](#contributors)


# Introduction
The `qiskit-symbolic` package is meant to be a Python tool to enable the symbolic evaluation of parametric quantum states and operators defined in [Qiskit](https://github.com/Qiskit/qiskit-terra) by parameterized quantum circuits.

A Parameterized Quantum Circuit (PQC) is a quantum circuit where we have at least one free parameter (e.g. a rotation angle $\theta$). PQCs are particularly relevant in Quantum Machine Learning (QML) models, where the values of these parameters can be learned during training to reach the desired output.

In particular, `qiskit-symbolic` can be used to create a symbolic representation of a parametric quantum statevector, density matrix, or unitary operator directly from the Qiskit quantum circuit. This has been achieved through the re-implementation of some basic classes defined in the [`qiskit/quantum_info/`](https://github.com/Qiskit/qiskit-terra/tree/main/qiskit/quantum_info) module by using [sympy](https://github.com/sympy/sympy) as a backend for symbolic expressions manipulation.


# Installation
### User-mode
To start using `qiskit-symbolic`, you can install the package directly from GitHub running the following command:
```
pip install git+https://github.com/SimoneGasperini/qiskit-symbolic.git
```
### Dev-mode
To install the package in development mode, first you have to clone locally the GitHub repository; then, move to the repo directory to install the develop dependencies and to launch the editable-mode installation running the following commands:
```
pip install -r requirements-dev.txt
pip install -e .
```


# Usage examples

### _Sympify_ a Qiskit circuit
Let's get started on how to use `qiskit-symbolic` to get the symbolic representation of a given Qiskit circuit. In particular, in this first basic example, we consider the following quantum circuit:
```python
from qiskit import QuantumCircuit
from qiskit.circuit import Parameter, ParameterVector

y = Parameter('y')
p = ParameterVector('p', length=2)

pqc = QuantumCircuit(2)
pqc.ry(y, 0)
pqc.cx(0, 1)
pqc.u(0, *p, 1)

pqc.draw('mpl')
```
![](/img/example_circuit.png)

To get the *sympy* representation of the unitary matrix corresponding to the parameterized circuit, we just have to create the symbolic `Operator` instance and call the `to_sympy()` method:
```python
from qiskit_symbolic import Operator

op = Operator.from_circuit(pqc)
op.to_sympy()
```
```math
\left[\begin{matrix}\cos{\left(\frac{y}{2} \right)} & - \sin{\left(\frac{y}{2} \right)} & 0 & 0\\0 & 0 & \sin{\left(\frac{y}{2} \right)} & \cos{\left(\frac{y}{2} \right)}\\0 & 0 & e^{i \left(p[0] + p[1]\right)} \cos{\left(\frac{y}{2} \right)} & - e^{i \left(p[0] + p[1]\right)} \sin{\left(\frac{y}{2} \right)}\\e^{i \left(p[0] + p[1]\right)} \sin{\left(\frac{y}{2} \right)} & e^{i \left(p[0] + p[1]\right)} \cos{\left(\frac{y}{2} \right)} & 0 & 0\end{matrix}\right]
```

If you want then to assign a value to some specific parameter, you can use the `subs(<dict>)` method passing a dictionary that maps each parameter to the desired corresponding value:
```python
params2value = {p: [-1, 2]}
new_op = op.subs(params2value)
new_op.to_sympy()
```
```math
\left[\begin{matrix}\cos{\left(\frac{y}{2} \right)} & - \sin{\left(\frac{y}{2} \right)} & 0 & 0\\0 & 0 & \sin{\left(\frac{y}{2} \right)} & \cos{\left(\frac{y}{2} \right)}\\0 & 0 & e^{i} \cos{\left(\frac{y}{2} \right)} & - e^{i} \sin{\left(\frac{y}{2} \right)}\\e^{i} \sin{\left(\frac{y}{2} \right)} & e^{i} \cos{\left(\frac{y}{2} \right)} & 0 & 0\end{matrix}\right]
```

### _Lambdify_ a Qiskit circuit
Given a Qiskit circuit, `qiskit-symbolic` also allows to generate a Python lambda function with actual arguments matching the Qiskit unbounded parameters.
Let's consider the following example starting from a `ZZFeatureMap` circuit, commonly used as a data embedding ansatz in QML applications:
```python
from qiskit.circuit.library import ZZFeatureMap

pqc = ZZFeatureMap(feature_dimension=3, reps=1)
pqc.draw('mpl')
```
![](/img/zzfeaturemap_circuit.png)

To get the Python lambda function representing, for instance, the final parameterized statevector, we just have to create the symbolic `Statevector` instance and call the `to_lambda()` method:
```python
from qiskit_symbolic import Statevector

sv = Statevector.from_circuit(pqc)
sv_func = sv.to_lambda()
```

We can now call the generated lambda function passing the actual values we want to assign to each free parameter (in alphabetical order, same convention used in `qiskit-terra`). The returned object will be a *numpy* 2D-array (with `shape=(8,1)` in this case) representing the final output statevector.
```python
values = [1.24, 2.27, 0.29]
statevec = sv_func(*values)
```

**_REMARK_** \
*When the PQC has to be evaluated on a large number of different sets of parameters values (typical case in QML), this `qiskit-symbolic` feature can help to significantly improve the (full-statevector) simulation performace. Indeed, the symbolic evalutation of the circuit and the lambda generation take place only once; then, the simulation only consists in executing multiple times the returned function passing a different set of parameters values for each iteration. For relatively shallow PQCs with a limilted number of qubits (e.g. Quantum Kernels evaluation), this can reduce the execution time up to two order of magnitudes (depending on the number of iterations) compared to the standard Qiskit simulation based on the [Aer Simulators](https://qiskit.org/documentation/tutorials/simulators/1_aer_provider.html) or the [Sampler](https://qiskit.org/documentation/stubs/qiskit.primitives.Sampler.html) primitive.*


# Contributors

<table>
  <tr>
    <td align="center"><a href="https://github.com/SimoneGasperini"><img src="https://avatars2.githubusercontent.com/u/71086758?s=400&v=4" width="120px;"/><br/><b>Simone Gasperini</b></a></td>
  </tr>
</table>