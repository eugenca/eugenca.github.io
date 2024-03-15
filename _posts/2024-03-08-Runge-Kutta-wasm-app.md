## Runge-Kutta online !

Fixed-step classic Runge-Kutta method solver for systems of differential equations

Click here to see it: [Click](https://eugenca.github.io/wasm_projects/runge-kutta/)

### Features:
* Robust calculation on the client side with web wasm application
* Almost unlimited number of functions / parameters
* Automatic parsing of functions (with some rules, see below)
* Optimization of function formulas - with [expression optimizer](https://github.com/Thorium/Linq.Expression.Optimizer)
* Almost native function performance compared to c# method (may be not totally true for web version, but true for desktop one)

### Rules:
f1', f2' ... - derivative functions of independent variable.
If we have 3 functions, then each we have:
```
f1'(t, x, y, z) or f1'(t, p1, p2, p3) or f1(IndependentVar, Param(1), Param(2), Param(3)) 
f2'(t, x, y, z) or f2'(t, p1, p2, p3) or f2(IndependentVar, Param(1), Param(2), Param(3)) 
f3'(t, x, y, z) or f3'(t, p1, p2, p3) or f3(IndependentVar, Param(1), Param(2), Param(3))
```
This shows how to write functions:
* We can access independent variable in function as t or IndependentVar
* We can access dependent variables as:\
    x, y, z (easy method for first three functions);\
    p1, p2, ... p20 (moderate method for first 20 functions);\
    Param(1), Param(2), ... Param(N) for any of N functions;
* We can write operations and even conditions (see [NCalc2](https://github.com/sklose/NCalc2/tree/master) on examples of usage)
