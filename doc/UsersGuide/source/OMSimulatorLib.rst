.. index:: OMSimulatorLib

OMSimulatorLib
=================

This library is the core of OMSimulator and provides a C interface that can
easily be utilized to handle co-simulation scenarios.

.. index:: OMSimulatorLib; C-API

C-API
#####

.. include:: OMSimulatorLib.inc


.. index:: OMSimulatorLib; Available Solver

Available Solver
################

OMSimulator will use an ODE solver as master method to solve a given system of components.

If the system contains strongly connected components, also called algebraic systems or loops,
an iterative method is used to solve these.

Master Solver
-------------

Bla bla for differences between weekly and strongly coupled systems.
From [SUNDIALS]_ CVODE with adaptive step size is used as central solver method.
I guess Eulers method is available as well?

Algebraic Loops
---------------

There are two different algebraic solvers available in OMSimulator:

  * A Fixed-Point-Iteration
  * KINSOL from [SUNDIALS]_

Both are iterative solvers and need a starting point close enough to the solution
to converge towards the solution.
While the fixed-point-iteraton doesn't have any overhead for allocation, initalization
and deallocation of the solver it can not solve difficult problems or find a
non-attractive solution.

For this reason KINSOL is used as default method in OMSimulator.
// TODO is it??

Example - Set KINSOL to solve strongly connected system of FMUs
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Generate model exchange FMUs `A.fmu` and `B.fmu` from modelica models `A` and `B`
with your favorit Modelica tool:

.. code-block:: modelica

  model A
    Modelica.Blocks.Interfaces.RealInput x_in;
    Modelica.Blocks.Interfaces.RealOutput x_out;
    Real y(fixed=true, start=2);
    parameter Real a = 2;
  equation
    der(y) = a*y;
    x_out = 2*x_in-y;
  end A;

.. code-block:: modelica

  model B
    Modelica.Blocks.Interfaces.RealInput x_in;
    Modelica.Blocks.Interfaces.RealOutput x_out;
  equation
    x_out = x_in;
  end B;

When connecting `A` and `B` together an strongly connected component spanning over
FMUs A and B is formed that needs to be solved at each step of the master integration
method.

.. code-block:: lua

  -- loopsOverFMUs.lua

  oms_setTempDirectory("./temp/")
  oms_newModel("loopsOverFMUs")
  oms_addSystem("loopsOverFMUs.root", oms_system_sc)

  -- instantiate FMUs
  oms_addSubModel("loopsOverFMUs.root.A", "A.fmu")
  oms_addSubModel("loopsOverFMUs.root.B", "B.fmu")

  -- connections: A <-> B
  oms_addConnection("loopsOverFMUs.root.A.x_out", "loopsOverFMUs.root.B.x_in")
  oms_addConnection("loopsOverFMUs.root.A.x_in", "loopsOverFMUs.root.B.x_out")

  -- simulation settings
  oms_setResultFile("loopsOverFMUs", "loopsOverFMUs.mat")
  oms_setStartTime("loopsOverFMUs", 0)
  oms_setStopTime("loopsOverFMUs", 0.1)

  -- instantiation
  oms_instantiate("loopsOverFMUs")

  oms_initialize("loopsOverFMUs")
  oms_simulate("loopsOverFMUs")
  oms_terminate("loopsOverFMUs")
  oms_delete("loopsOverFMUs")

While the loop is not complicated, it can not be solved by an fixed-point iteration
since the solution is a repelling fixed-point.
So one needs to use the flag `algLoopSolver` from :ref:`OMSimulator Flags <oms-flags>` to
use the KINSOL solver as algebraic loop solver.

.. code-block:: bash

  OMSimulator loopsOverFMUs.lua --algLoopSolver=kinsol


.. [SUNDIALS] Alan C. Hindmarsh, Peter N. Brown, Keith E. Grant, Steven L. Lee, Radu Serban, Dan E. Shumaker, and Carol S. Woodward.
  2005.
  SUNDIALS: Suite of nonlinear and differential/algebraic equation solvers.
  ACM Trans. Math. Softw. 31, 3 (September 2005), 363–396.
  DOI:https://doi.org/10.1145/1089014.1089020
