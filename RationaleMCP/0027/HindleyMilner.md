# Unit checking

Unit checking is the process of inferring the units of Modelica variables, and ensuring the variables are used consistently in expressions and equations.

It takes place after any evaluation of evaluable parameters and no evaluable parameters shall be evaluated during this process.
(This must be true as long as tools allow the users to opt-out from unit-checking.)

It is performed by using a type inference algorithm, albeit adapted to work with units.
The algorithm proposed here is based on the Hindley-Milner algorithm.
At a high level the algorithm collects a set of constraints on the units of expressions.
It extracts a set of substitutions from these constraints which once applied determine whether a variable has a unit and what that unit would be.
Any inconsistency discovered during this process is a unit error, and any unit expression that contains remaining unknowns at the end of the process is considered unknown.

## The unit of a Modelica variable

The unit of a Modelica variable is the unit associated with the variable after unit-checking has been performed, possibly remaining unknown.

Note that the unit of a variable is not the same concept as the `unit`-attribute of the variable, but the unit will be equal to the `unit`-attribute when present.

## Units of Modelica expressions

The unit of an arbitrary Modelica expression is represented by a unit meta-expression.
This section outlines the different constructs it comprises.

### Variables
The unit of the Modelica variable `var` is represented by the unit variable `var.unit`.
In practice, these are the unknowns for the inference algorithm to determine.

### Literals

#### `well-formed` unit
The `well-formed` units corresponds to unit expressions.

For the purposes of unit checking, they can be uniquely represented by their base unit factorization, offset and scaling, up to the order of the base units.

#### `empty` unit
An expression with `empty` unit won't contribute to the unit of a parent expression, and will meet any constraints imposed on it.
In practice, these are used to represent the unit of literal values.
They provide a more flexible solution than considering any literal value to have unit `"1"`.
As well as a stricter solution than letting the unit of literals be inferred like the unit of a variable would, which would effectively make literals wildcard from the perspective of the unit system.

#### `undefined` unit
An expression with `undefined` unit is used to represent unspecified or error cases, and will meet any constraints imposed on it.

### Operators

#### *
Consider the meta-expression `e1 * e2`.
* If both `e1` and `e2` are `well-formed`, it evaluates to a `well-formed` unit resulting from multiplying `e1` by `e2`.
* If one of them is `empty`, it evaluates to the other one. (In effect, the `empty` unit becomes unit `"1"`)
* If one of them is `undefined`, it evaluates to `undefined`.

#### ^
Consider the meta-expression `b ^ e`, where `e` is a rational.
* If `b` is `well-formed`, it evaluates to a `well-formed` unit resulting from raising `b` to the power `e`.
* If `b` is `empty`, it evaluates to `empty`.
* If `b` is `undefined`, it evaluates to `undefined`.

#### der
Consider the meta-expression `der(e)`.
* If `e` is `well-formed`, it evaluates to `e * ("s" ^ -1)`.
* If `e` is `empty`, it evaluates to `empty`.
* If `e` is `undefined`, it evaluates to `undefined`.

This definition prevents `der` from introducing `"s"` in a model that does not contain any units.
It could lead to unwanted unit errors otherwise.

### Evaluation
Unit variables can't be found to be `empty` or `undefined` unit.
This means that after evaluation, a unit meta-expression is either `undefined`, `empty` or does not contain any `undefined` or `empty` unit.

## Unit equivalence
Two well-formed units are said to be equivalent if all of the following are true:
* Their base unit factorizations are equal.
* Their base unit offsets are equal.
* Their base unit scalings are equal.

Examples:
- `"N"` and `"m/s2"` have different base unit factorizations, they are not equivalent.
- `"K"` and `"degC"` have different base unit offsets, they are not equivalent.
- `"s"` and `"ms"` have different base unit scaling, they are not equivalent.
- `"kN"` and `"W.s/mm"` are equivalent.

It is a quality of implementation how equality of offsets and scalings are checked.

`empty` units and `undefined` units are always equivalent to each other, themselves and any `well-formed` units.

## Unit convertibility
A well-formed unit `u1` is said to be convertible to another well-formed unit `u2` if:
* Their base unit factorizations are equal.

Examples:
- `"N"` is not convertible to `"m/s2"`, since they have different base unit factorizations.
- `"K"` is convertible to `"degC"`, since they differ by base unit offsets.
- `"s"` is convertible to `"ms"`, since they differ by base unit scaling.
- `"kN"` is convertible to `"W.s/mm"`.

`empty` units and `undefined` units are never convertible to each other, themselves or any `well-formed` units.

## Unit inference

### Modified Hindley-Milner

The original Hindley-Milner algorithm is designed to infer types, and consists of building a set of substitutions that returns the type of any expression of a given program.
Applying this algorithm to perform unit inference in Modelica comes with two important differences:
  * Not all variables must have a unit in a given model.
  * There are operations on units, that is, not all constraints are of the form  _`var1.unit` is equivalent to `var2.unit`_.

#### Conditional constraints
One notion that doesn't exists in the original Hindley-Milner algorithm is conditional constraints.
In Modelica, errors that are in unused part of a simulation model should typically be ignored (i.e removed conditional components).
In the context of unit inference, some modelica constructs generate constraints that are predicated on certain expressions having been evaluated or not.
The constraint conditions are conjunctive, in other words, if even a single condition evaluates to `false` the constraint should be discarded.

#### The algorithm

The Hindley-Milner algorithm modified to perform unit inference in the Modelica language:
* Traverse the flattened equation system and collect constraints before any evaluation has occured.
* After evaluable parameters have been evaluated, and expressions have been reduced to values when possible, evaluate both sides of the constraints, as well as their conditions. (This propagates `empty` and `undefined` units to the top of the constraints.)
* Discard any constraints of the form _`u` must be equivalent to empty unit_, or _`u` must be equivalent to undefined unit_, as they are trivially satisfied.
* Discard any constraints having at least a condition that was evaluated to `false`.
* For all constraints:
    * If the constaint is of the form _`well-formed` unit `u1` must be equivalent to `well-formed` unit `u2`_, check that this is the case, it is an error otherwise.
    * Otherwise:
        * Rewrite the constraint to be of the form `var.unit` is equivalent to `u` where `u` is a unit meta-expression that doesn't contain references to the unit variable `var.unit`, if it is not possible, discard the constraint.
        * Apply the substitution, _`var.unit` => `u`_, to the right-hand side of the substitutions already in the set.
        * Add the substitution to the set of substitutions
        * Apply the substitution to all remaining constraints.

* For every variable `var`, retrieve the substitution whose left-hand side is `var.unit`.
    * If the right-hand side of the substitution is a `well-formed` unit, then this is the unit of `var`. (Given that no unit errors have been detected, a variable with a `unit`-attribute will get the given unit.)
    * Otherwise, the unit of `var` is unknown.

The following rules describe the unit meta-expression of a given Modelica expression, where it makes sense.
They also provide the constraints that arise from different Modelica constructs.

When it is specified that _`u1` must be equivalent to `u2`_, the corresponding constraint should be collected.

In this section, the unit meta-expression of expression `e` is called `e.unit` and conditions of conditional constraints are highlighted with [].

### Variables
Consider the expression `e` of the form `var`, where `var` is a Modelica variable.
* If `var` doesn't have a declared unit attribute, and has constant component variablity, `e.unit` is equal to `empty` unit.
* Otherwise, `e.unit` is equal to `var.unit`.

If `var` has a declared unit attribute, `var.unit` must be equivalent to it.

### Literals
Literals have `empty` unit.

### Binary operations
Consider an expression `e` of the form `e1 op e2`.

#### Additive operators
* `e.unit` must be equivalent to `e1.unit`.
* `e.unit` must be equivalent to `e2.unit`.

Note that this requires introducing a temporary variable to represent `e.unit`.
This allows unit propagation when one of the operand has `empty` unit.

#### Multiplication
`e.unit` is equal to `e1.unit * e2.unit`.

#### Division
`e.unit` is equal to `e1.unit * (e2.unit ^ -1)`.

#### Relational operators
The unit of `e1` and the unit of `e2` must be equivalent.

#### Power operator
* `e.unit` must be equivalent to `e1.unit ^ e2`.
* If [`e2` wasn't evaluated or is a Real expression], `e1.unit` must be equivalent to `"1"` and .

Note that this requires introducing a temporary variable to represent `e.unit`.
Technically, `e2` is a modelica expression rather than a rational number, which means that `e1.unit ^ e2` cannot be represented as a unit meta-expression.
In practice, if the condition of the second constraint is verified, `e1.unit ^ e2` is equal to `"1"` and if it isn't verified, then it can be represented as a meta-expression.

(If the exponent was not evaluated or is a Real, the unit of the base should be `"1"`.)
(It is up for discussion, when `e2` is Real, whether `e2.unit` should be equivalent to `"1"`. It could be handled in a similar way as transcendental functions. See below.)

### Function calls
Consider the expression `e` of the form `f(e1, e2, …, en)`.
* If the output of the function has a declared unit, `e.unit` is the declared unit.
* `e.unit` is `undefined` otherwise.

For each argument `ei`, `ei.unit` must be equivalent to the declared unit of the corresponding function input.

### Transcendental functions
As examples of transcendental functions, consider `sin`, `sign`, and `atan2`.
Other transcendental functions can be handled similarly.

Consider expression `e` of the form `sin(e1)`.
* If `e1.unit` is `empty`, then so is `e.unit`.
* If `e1.unit` is not `empty`, then it must be equivalent to `"1"` and `e.unit` is `"1"`.

Note that some of these constraints can only be processed once `e1.unit` has been determined.
This avoids introducing a unit `"1"` in the inference algorithm.

Implementation note: The rule above can be implemented as making `e.unit` equal to `e1.unit`, and if `e1.unit` is `well-formed` after completed unit inference, then verify that it is equivalent to `"1"`.

Consider an expression `e` of the form `sign(e1)`.
* If `e1.unit` is `empty`, then so is `e.unit`.
* If `e1.unit` is not `empty`, then `e.unit` is `"1"`.

Consider an expression `e` of the form `atan2(e1, e2)`.
* If `e1.unit` must be equivalent to `e2.unit`.
* If `e1.unit` and `e2.unit` are `empty`, then so is `e.unit`.
* If neither `e1.unit` nor `e2.unit` is `empty`, then `e.unit` is `"1"`.

### Operator der
The unit of the expression `der(e)` is `der(e.unit)`.

### If expressions
Consider the expression `e` of the form `if cond then e1 else e2`.
If [`cond` was not evaluated]:
* `e.unit` must be equivalent to `e1.unit`.
* `e.unit` must be equivalent to `e2.unit`.

If [`cond` was evaluated], `e.unit` must be equivalent to the unit of the selected branch.

Note that this requires introducing a temporary variable to represent `e.unit`.

### Equations
Both sides of binding equations, equality equations and connect equations must be unit equivalent.

## Unit operators

### `withUnit($value$, $unit$)`
Create an expression with unit `$unit$`, and value `$value$`.
Argument $value$ needs to be a `Real` expression, with `empty` unit.

This operator makes it possible to attach a unit to a literal.

### `withoutUnit($value$, $unit$)`
Create an expression with empty unit, whose value is the numerical value of `$value$` expressed in `$unit$`.
Argument `$value$` needs to be a `Real` expression, with a well-formed unit.
The unit of `$value$` must be convertible to `$unit$`.

### `inUnit($value$, $unit$)`
Create an expression with unit `$unit$`, whose value is the conversion of `$value$` to `$unit$`.
Argument `$value$` needs to be a `Real` expression, with a well-formed unit.
The unit of `$value$` must be convertible to `$unit$`.

## Possible extensions and current limits

### Builtin operators and functions
The current state of this proposal does not cover the unit checking of the every Modelica builtin operators and functions.

### Arrays
We don't try to specify how this would work an arrays, although the current proposal work straight forwardly with element-wise operators.
We are confident that it could be extended to matrix computation too, but haven't a proof of concept for that part.

### Constants
This proposal takes a stance regarding how the unit of constants should be handled but we recognise that this is still very much up for discussion.
It is also somewhat orthogonal to the rest of the proposal.

### Functions
The Hindler-Milner algorithm, provides a way to infer unit attributes of the inputs and outputs of functions.
Concretely, it would imply computing the bundles of constraints from the body of each functions and adding them to the algorithm, after adequate substitutions, when processing a function call.

### Variable initialisation pattern
A common pattern used in the MSL, is the following:
```
model M
  Real d(unit = "s") = 12;
  Real v(unit = "m/s") = 0.1 / (3 * d);
end M;
```
Where a modeler knows how to express a variable with unit in function of another, with help of literal values that should really carry a unit.
The way to fix that with this proposal is to attach, a unit to the literal values with a call to `withUnit`.

### Syntax extension
We could extend the Modelica language to add syntactic sugar for calls to `withUnit`.
The following variants have all been successfully test implemented:
* `0.1[m]`
* `0.1 'm'` (or as `0.1'm'` to show more clearly that the quoted unit belongs to the literal)
* `0.1."m"`

### unsafeUnit operator
Another way to deal with the pattern described above would be to provide an unsafe operator that effectively drops any constraints generated in its subexpression.
The whole expression would have `undefined` unit.
