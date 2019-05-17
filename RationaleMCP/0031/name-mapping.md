# Name mangling

With Flat Modelica being a subset of full Modelica, variable names in Flat Modelica are constrained according to the identifier rules for Modelica.  At the same time, Flat Modelica shall support scalarization during reduction to Flat Modelica, implying that names must be able to encode the hierarchy of the original model.

Flat Modelica must also define a safe namespace that tools may use for automatically generated identifiers.
