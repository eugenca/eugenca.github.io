# String to native function in runtime!

## Math introduction

Recently I came with idea to make a calculator for some math problems, specifically to optimal control, which uses integration routines on non-linear differential systems of equations.
We can integrate system of differential equation (Cauchy problem) by using different methods, one of which is [Runge-Kutta](https://en.wikipedia.org/wiki/Runge%E2%80%93Kutta_methods) 4th order method.
We set N of equations with N of initial values, start and end independent variable, and integration step.
For example, free fall (g ~ 9.81), without air resistance, with initial position (posX, posX) and speed (VelX, VelY) from t0 = 0 to tb = 51
```
dx(t)/dt = Vx(t)
dy(t)/dt = Vy(t)
dVx(t)/dt = 0
dVy(t)/dt = -g

x0 = posX
y0 = posY
Vx0 = VelX
Vy0 = VelY
```

## Programming part
Now to the interesting part:
When we have some equations, we must somehow put them to the program. When you program in C/C++, you may hard-code functions, for example dy(t)/dt will look like:
```cpp
double YDerivative (double t, double x, double y, double Vx, double Vy)
{
    (void)t; (void)x; (void)y; (void)Vx; // We don't use them but must have them as arguments
    return Vy;
}
```

If we want to let user to input funtions, we must somehow interpret them, which is not robust. Function with 6 parameters may call a lot of function in the expression tree with a lot of indirection
For example evaluation of "(p1 * p3 / p2) + ((9 * 2) % 8)" function would result in something like:
```
res1 = (p1*p3)
res2 = (res1 / p2)
res3 = (9 * 2)
res4 = res3 % 8
res5 = res2 + res4

                plus
           /           \
         div           mod
        /   \         /   \
      mul   p2      mul    8
     /  \          /   \
    p1  p3        9     2
```

In C# we have Expression which have similar representation of operations and data, which is being built in runtime, and we have ability to compile it in method (or lambda) by converting it to IL and then to machine instriction with JIT.
So... we come up with native function at runtime in result!
After short investigation, I found parser library that does parsing from string, containing equation to Expression and then compiling it to lambda - [NCalc2](https://github.com/sklose/NCalc2)
And now, to demonstrate it - I made webassembly app with Avalonia UI.
Internally we have ```Func<RK4FuncContext, double>``` which takes context object and returns double, for our integration routine.
RK4FuncContext is defined as (may be subject to change later):
```csharp
public struct RK4FuncContext
{
    public double t { get => IndependentVar; }

    public double x { get { return Params[0]; } }
    public double y { get { return Params[1]; } }
    public double z { get { return Params[2]; } }

    public double p1 { get { return Params[0]; } }
    public double p2 { get { return Params[1]; } }
    public double p3 { get { return Params[2]; } }
    public double p4 { get { return Params[3]; } }
    public double p5 { get { return Params[4]; } }
    public double p6 { get { return Params[5]; } }
    public double p7 { get { return Params[6]; } }
    public double p8 { get { return Params[7]; } }
    public double p9 { get { return Params[8]; } }
    public double p10 { get { return Params[9]; } }

    public double p11 { get { return Params[10]; } }
    public double p12 { get { return Params[11]; } }
    public double p13 { get { return Params[12]; } }
    public double p14 { get { return Params[13]; } }
    public double p15 { get { return Params[14]; } }
    public double p16 { get { return Params[15]; } }
    public double p17 { get { return Params[16]; } }
    public double p18 { get { return Params[17]; } }
    public double p19 { get { return Params[18]; } }
    public double p20 { get { return Params[19]; } }

    public double IndependentVar;
    public double[] Params;

    public double Param(int index)
    {
        return Params[index];
    }
}
```

So we can use x, y, z as first three dependent arguments, p1, p2, ... , p20 as first 20 dependent arguments, t as independent variable, and Param(0) ... Param(99999) as custom dependent (to use more than 20 provided by p1-p20)
so function string ```"y + p2 + Param[1]"``` is the same as ```"y + y + y"``` or ```"p2 + p2 + p2"```
Also we can use any math functions provided in NCalc2, such as sin, cos, sqrt.

According to behavior, tests and benchmarks, running compiled ```"(p1 * p3 / p2) + ((9 * 2) % 8)"``` function is equivalent to running next c# method:
```csharp
public double Function(RK4FuncContext context)
{
    //                                              (optimized to constant)
    return (context.p1 * context.p3 / context.p2) + (     (9 * 2) % 8     )
}
```

You can try it online: [Click](https://eugenca.github.io/2024/03/08/Runge-Kutta-wasm-app.html)

P.S.
Actually, in wasm application I don't know clearly how Expression objects are compiled into method, and performance is different between desktop application and web one. It would be topic of my further investigation, and if you know something about it - welcome to comments!
