---
layout: post
title:  "Speeding Up .NET Code with Vector Intrinsics"
date:   2021-02-27 14:48:00 +0100
categories: programming
math: yes
---

## Introduction

Many of today's applications are not computationally heavy in the sense of
number crunching. Most of them are more line-of-business oriented, mostly
relying on good and fast databases, network protocols, etc. If there is
number crunching to be done, it is well hidden away as an implementation
detail of a ready-to-use component. The wizards writing the database engine,
network stacks, JSON parsers, etc. often heavily rely on computationally
(i.e. CPU-intensive) operations. Even more pronounced, the new resurgence of
artificial intelligence and machine learning involves a lot of
computationally intensive work. Not to forget the whole financial industry
with crypto-mining and high frequency trading.

For such demands, the CPU vendors created special instructions that are
capable of applying the same operation to multiple operands at once.
Collectively, they are referred to as [SIMD] (single instruction, multiple
data) instructions. In this blog post I will explore how these instructions
can be leveraged in the context of .NET to speed up a computationally
demanding algorithm.

As diverse and ubiquitous computers are, their original application of their
computational power was in science and engineering. And still today, the
biggest and most powerful super computers are used to investigate phenomena
of turbulence, combustion, create weather forecasts, solve protein folding
problems, etc. pp. Many of these applications involve the solution of
so-called ordinary differential equations (a.k.a. ODE's), in particular
[initial value problems] (IVP's) of the form

$$
  \begin{aligned}
  \frac{\mathrm{d}\,\mathbf{y}(t)}{\mathrm{d}\,t} 
                  &= \mathbf{f}(t, \mathbf{y}(t)) \quad,\\
  \mathbf{y}(t_0) &= \mathbf{y_0} \quad,
  \end{aligned}
$$

where \\(\mathbf{y}(t)\\) is a time-dependent state vector,
\\(\mathbf{f}(t, \mathbf{y}(t))\\) it's derivative with respect to time,
\\(t\\) the time variable and \\(\mathrm{y}_0\\) is a known state at time \\(t_0\\).

A wide range of phenomena can be described with this type of equation, e.g.

* exponential decay or growth (such as the spread of a disease),
* oscillators, such as pendulums or spring-mass systems,
* planetary systems,
* chemical reactions.

For very simple problems, analytical solutions can be found. Quickly,
however, practical applications become intractable and approximate solutions
using numerical methods need to be found. How these methods are developed,
the mathematics around ODE's, the trade-offs between the many algorithms,
their applicability to various problems, etc. are beyond the scope of this
blog post. Instead, I will present the very popular and well known
[Runge-Kutta-Fehlberg 4(5)], a.k.a _RK4(5)_, method as an example.

The full code for the examples developed below is in my [GitHub repository].

# The Runge-Kutta 4(5) Method

The method uses discrete time steps of size \\(h\\) to march along the slope
of the function \\(\mathbf{y}(t)\\). The effective increment is a weighted
average of six different estimates of the increment. The formulas for the
increments are given by

$$
\begin{aligned}
  \mathrm{k}_1 &= h \cdot \mathrm{f}(t + A(1)\cdot h,\, \mathbf{y}) \quad,\\
  \mathrm{k}_2 &= h \cdot \mathrm{f}(t + A(2)\cdot h,\, B(2,1)\cdot\mathbf{k}_1) \quad,\\
  \mathrm{k}_3 &= h \cdot \mathrm{f}(t + A(3)\cdot h,\, B(3,1)\cdot\mathbf{k}_1 + B(3,2)\cdot\mathbf{k}_2) \quad,\\
  \mathrm{k}_4 &= h \cdot \mathrm{f}(t + A(4)\cdot h,\, B(4,1)\cdot\mathbf{k}_1 + B(4,2)\cdot\mathbf{k}_2 + B(4,3)\cdot\mathbf{k}_3) \quad,\\
  \mathrm{k}_5 &= h \cdot \mathrm{f}(t + A(5)\cdot h,\, B(5,1)\cdot\mathbf{k}_1 + B(5,2)\cdot\mathbf{k}_2 + B(5,3)\cdot\mathbf{k}_3) + B(4,4)\cdot\mathbf{k}_4 \quad,\\
  \mathrm{k}_6 &= h \cdot \mathrm{f}(t + A(6)\cdot h,\, B(6,1)\cdot\mathbf{k}_1 + B(6,2)\cdot\mathbf{k}_2 + B(6,3)\cdot\mathbf{k}_3) + B(5,4)\cdot\mathbf{k}_4 + B(5,6)\cdot\mathbf{k}_5 \quad.\\
\end{aligned}
$$

Using the weighted average to find the value of the function at the new time
step is then

$$
  \mathbf{y}(t + h) = \mathbf{y}(t) + \sum_{i=1}^6 C_h(i)\cdot \mathbf{k}_i \quad .
$$

An important benefit of this method is, that the estimated increments
\\(\mathbf{k}_i\\) can also be used to estimate the error made by using
an estimated slope across a finite time step \\(h\\):

$$
  \mathbf{T}_{err} = \left| \sum_{i=1}^6 C_t(i)\cdot\mathbf{k}_i \right| \quad .
$$

This error estimate can be used to correct the time step size \\(h\\) in case
the the solution accuracy is either too low or too high. Yes, even a too high
accuracy can be bad because the solution quality will not significantly
improve, however a lot of CPU cycles will go to waste.

Hence, the algorithm rejects the increment if \\(T_{err,i} > \varepsilon_i\\)
for any \\(i\\), where \\(\mathbf{\varepsilon}\\) is a user-defined maximum
permissible error for each component of the function \\(\mathbf{y}(t)\\).
Alternatively, a scalar value \\(\varepsilon\\) can be given which is then
compared against the maximum value of vector \\(\mathbf{T}_{err}\\).

In any event, a new time step size \\(h_n\\) is calculated as

$$
  h_n = 0.9 \cdot \left( \min_i\frac{\varepsilon}{T_{err,i}} \right)^{1/5} \quad.
$$

If the increment has been rejected, it is recomputed using the new, smaller
time step size \\(h_n\\). Otherwise the next increment is computed with
\\(h_n\\).

By using this adaptive time step size, the algorithm ensures that the
estimated error remains just below \\(\mathbf{\varepsilon}.\\) The
coefficients used in the formulas above are given by the below table:

<table>
<thead>
<tr>
    <th rowspan="2">i</th>
    <th rowspan="2">A(i)</th>
    <th colspan="5">B(i, j)</th>
    <th rowspan="2">C<sub>h</sub>(i)</th>
    <th rowspan="2">C<sub>t</sub>(i)</th>
</tr>
<tr>
    <th>j=1</th>
    <th>j=2</th>
    <th>j=3</th>
    <th>j=4</th>
    <th>j=5</th>
</tr>
</thead>
<tbody>
    <tr>
        <td>1</td>
        <td>0</td>
        <td></td>
        <td rowspan="2"></td>
        <td rowspan="3"></td>
        <td rowspan="4"></td>
        <td rowspan="5"></td>
        <td>16/135</td>
        <td>1/360</td>
    </tr>
    <tr>
        <td>2</td>
        <td>1/4</td>
        <td>1/4</td>
        <td>0</td>
        <td>0</td>
    </tr>
    <tr>
        <td>3</td>
        <td>3/8</td>
        <td>3/32</td>
        <td>9/32</td>
        <td>6656/12825</td>
        <td>-128/4275</td>
    </tr>
    <tr>
        <td>4</td>
        <td>12/13</td>
        <td>1932/2197</td>
        <td>-7200/2197</td>
        <td>7296/2197</td>
        <td>28561/56430</td>
        <td>-2187/75240</td>
    </tr>
    <tr>
        <td>5</td>
        <td>1</td>
        <td>439/216</td>
        <td>-8</td>
        <td>3680/513</td>
        <td>-845/4104</td>
        <td>-9/50</td>
        <td>1/50</td>
    </tr>
    <tr>
        <td>6</td>
        <td>1/2</td>
        <td>-8/27 </td>
        <td>2</td>
        <td>-3544/2565</td>
        <td>1859/4104</td>
        <td>-11/40</td>
        <td>2/55</td>
        <td>2/55</td>
    </tr>
</tbody>
</table>

That's a whole lot of crazy magic numbers. Welcome to the world of numerical
mathematics. However, the numbers are not arbitrary, or guessed, they are a
consequence of rigorous analytical deductions involving Taylor series
expansions and polynomial fitting.

![Maths GIF](https://media.giphy.com/media/DHqth0hVQoIzS/giphy.gif)

# Naïve .NET Implementation

Looking at the algorithmic description above, transferring it to an
implementation in C# appears to be quite straight forward. Below code skips
important parts, such as imports, argument validation and output generation.
The full code is in the [GitHub repository].

```cs
public static class OdeNaive
{
  public static IEnumerable<(double, double[])> Rk45(
    Func<double, IEnumerable<double>, IEnumerable<double>> func,
    double[] y0,
    double t0,
    double tEnd,
    double h,
    double hMax,
    double[] epsilon)
  {
    // Define A(i)
    const double A1 = 0.0    , A2 = 1.0/4, A3 = 3.0/8,
                 A4 = 12.0/13, A5 = 1.0  , A6 = 1.0/2;

    // Define B(i, j)
    const double
        B21 =  1.0/4      ,
        B31 =  3.0/32     , B32 =  9.0/32     ,
        B41 =  1932.0/2197, B42 = -7200.0/2197, B43 =  7296.9/2197,
        B51 =  439.0/216  , B52 = -8.0        , B53 =  3680.0/513 ,
                            B54 = -845.0/4104 ,

        B61 = -8.0/27     , B62 =  2.0        , B63 = -3544.0/2565,
                            B64 =  1859.0/4104, B65 = -11.0/40;

    // Define Ch(i)
    const double Ch1 = 16.0/135     , Ch2 =  0.0   , Ch3 = 6656.0/12825,
                 Ch4 = 28561.0/56430, Ch5 = -9.0/50, Ch6 = 2.0/55      ;

    // Define Ct(i)
    const double Ct1 =  1.0/360      , Ct2 = 0.0   , Ct3 = -128.0/4275,
                 Ct4 = -2187.0/75240 , Ct5 = 1.0/50, Ct6 =  2.0/55    ;

    // SNIP -- PARAMETER VALIDATION

    // Initialization
    var y = y0.ToArray();
    var t = t0;

    // Output at t0
    yield return (t, y);

    // Perform iterations
    do
    {
      // Clamp h to not exceed hmax and not overshoot tEnd
      h = Math.Min(Math.Min(h, hMax), tEnd - t);

      // Calculate k1 through k6
      var k1 = func(t + h * A1, y).Mul(h).ToArray();
      var k2 = func(t + h * A2, y.Add(k1.Mul(B21))).Mul(h).ToArray();
      var k3 = func(t + h * A3, y.Add(k1.Mul(B31)).Add(
            k2.Mul(B32))).Mul(h).ToArray();
      var k4 = func(t + h * A4, y.Add(k1.Mul(B41)).Add(
            k2.Mul(B42)).Add(k3.Mul(B43))).Mul(h).ToArray();
      var k5 = func(t + h * A5, y.Add(k1.Mul(B51)).Add(
            k2.Mul(B52)).Add(k3.Mul(B53)).Add(k4.Mul(B54))).Mul(h).ToArray();
      var k6 = func(t + h * A6, y.Add(k1.Mul(B61)).Add(
            k2.Mul(B62)).Add(k3.Mul(B63)).Add(k4.Mul(B64)).Add(
              k5.Mul(B65))).Mul(h).ToArray();

      // Calculate error estimate for each component
      var te = k1.Mul(Ct1).Add(
        k2.Mul(Ct2)).Add(
          k3.Mul(Ct3)).Add(
            k4.Mul(Ct4)).Add(
              k5.Mul(Ct5)).Add(
                k6.Mul(Ct6)).Select(Math.Abs);

      // If only single epsilon given, reduce
      if (epsilon.Length == 1)
      {
        te = new[]{ te.Max() };
      }

      // If error criterion is fulfilled, perform increment
      if (te.Zip(epsilon).All(x => x.First <= x.Second))
      {
        y = y.Add(
          k1.Mul(Ch1)).Add(
            k2.Mul(Ch2)).Add(
              k3.Mul(Ch3)).Add(
                k4.Mul(Ch4)).Add(
                  k5.Mul(Ch5)).Add(
                    k6.Mul(Ch6)).ToArray();
          t += h;
      }

      // Adapt step size
      h *= 0.9 * Math.Pow(epsilon.Div(te).Min(), 1.0/5);
    } while (t < tEnd);

    // Output at tEnd
    yield return (t, y);
  }
}
```

Above code uses a few helper extension methods to make the mathematics a bit
simpler to write:

```cs
// Helper extension methods
public static class MathExtensions
{
  // Element-wise addition of two sequences
  public static IEnumerable<double> Add(
    this IEnumerable<double> @this, IEnumerable<double> other)
    => @this.Zip(other).Select(v => v.First + v.Second);

  // Multiplies every element of a sequence by a scalar value
  public static IEnumerable<double> Mul(
    this IEnumerable<double> @this, double other)
    => @this.Select(v => v * other);

  // Element-wise division of two sequences
  public static IEnumerable<double> Div(
    this IEnumerable<double> @this, IEnumerable<double> other)
    => @this.Zip(other).Select(v => v.First / v.Second);
}
```

Now that we have the necessary pieces in place, let's take this ODE solver
for a spin and solve a very simple exponential decay problem of the form

$$
\begin{aligned}
  \frac{\mathrm{d}\,\mathbf{y}(t)}{\mathrm{d}\,t} &= -\frac{1}{2}\mathbf{y} \quad,\\
  \mathbf{y}(t0) &= \mathbf{y}_0 \quad.
\end{aligned}
$$

For this problem the analytical solution is very simple and we can use it to
validate our implementation

$$
\mathbf{y}(t) = \mathbf{y}_0 e^{-\frac{1}{2}t} \quad.
$$

Again, for brevity's sake, I'll skip the niceties:

```cs
class Program
{
  static void Main(string[] args)
  {
    // Warmup
    var result = SolveExponentialDecay();

    // Run for 20 rounds
    const double nRounds = 20;
    var total = 0L;
    System.Diagnostics.Stopwatch timer = new();

    for (var i = 0; i < nRounds; ++i)
    {
        timer.Start();
        _ = SolveExponentialDecay();
        timer.Stop();
        total += timer.ElapsedMilliseconds;
        timer.Reset();
    }

    // Print out results
    Console.Error.WriteLine($"Average wall clock time: {total / nRounds} ms");

    var yNames = Enumerable.Range(0, result[0].Item2.Length).Select(
        i => $"y[{i}]");
    Console.WriteLine($"t\t{string.Join('\t', yNames)}");

    foreach (var (i, row) in result.Select((r, i) => (i, r)))
    {
      var yValues = row.Item2.Select(v => v.ToString("G"));
      Console.WriteLine($"{row.Item1:G}\t{string.Join('\t', yValues)}");
    }
  }

  static IEnumerable<(double, double[])> SolveExponentialDecay()
    => OdeNaive.Rk45(
      // Right-hand-side function: f(t, y) = -0.5 y
      func: (double t, IEnumerable<double> y) => y.Mul(-0.5),
      y0: new[] { 1.0, 100, 51, 32 },
      t0: 0,
      tEnd: 10,
      h: 1e-5,
      hMax: 1e-2,
      epsilon: new[] { 1e-7 }).ToList();
}
```

Running this results in the following output (with the number of decimals
truncated for better visual display):

```sh
$ dotnet run -c Release
Average wall clock time: 1748 ms
t       y[0]         y[1]          y[2]          y[3]
0       1            100           51            35
10      0.006738     0.673795      0.343635      0.235828
```

Using the analytical solution from above, let's also verify the results
with Python:

```py
>>> import math
>>> [y0*math.exp(-0.5*10) for y0 in (1, 100, 51, 35)]
[0.006738, 0.673795, 0.343635, 0.235828]
```

As can be seen, up to the displayed six decimal digits, the results match
perfectly. But an average of 1.7 seconds for such a simple problem? Can we
do better? In the algorithm's implementation I took care not to execute
too many loops by using Linq to compute chained arithmetic operations.
However, the arithmetic operations themselves I didn't pay any attention
to.

You might have heard though, that CPU's are capable of carrying out the
same operation on multiple values with a single instructions. This feature
is called SIMD (single instructions, multiple data). It is also known as
vectorization. So, instead of e.g. adding the items of two arrays element
by element, the CPU would actually be able to do it in chunks. This
technology started out as [MMX] (multimedia extensions, primarily integer math
used for video and audio encoding and decoding) on the Intel Pentium CPU, but
evolved via [SSE1] to SSE4 to [AVX], AVX2 and AVX-512. Intel's competitor, AMD,
also supports many of those, but also has it's own, incompatible, SIMD
instructions. Itanium and ARM again have their own instruction sets. In the
next section I'll try to squeeze some more performance out of the algorithm
by using vector extensions that should be present on most modern x86/x64
CPU's.

# Vectorizing the RK-4(5) Algorithm

This article is not about what SIMD instructions are, how they relate to
compiler intrinsics, and the nitty gritty details on how to use them. There's
plenty of excellent resources available for that. Let's just dive in and see
how they can help, instead. A few remarks, however, are in place:

* There are, depending on the CPU capabilities, operations for different sizes
  of vectors. Old CPU's only supported vectors of 64-bits and 128-bits size.
  The latter data package can be filled with e.g. 2 `double` values, 4
  `float` or `int` values, 8 `Int16` values or 16 `byte`'s. Newer CPU's also
  have vectors with 256 or 512 bits. Microsoft has nice wrappers for these
  vectors in the form of the [`Vector64<T>`], [`Vector128<T>`] and
  [`Vector256<T>`]. 512-bit support is not widespread or homogeneous, hence
  there is no `Vector512<T>`. The non-templated classes [`Vector64`],
  [`Vector128`] and [`Vector256`] contain various factory methods for creating,
  casting (reinterpreting the bits!) and extraction of individual elements or
  the lower and upper half of a vector (i.e. decomposing e.g. a
  `Vector256<T>` into two `Vector128<T>` values).
* The intrinsic instructions that operate on these vectors are contained
  in various classes that derive from [`X86Base`]. For ARM CPU's there's also
  a hierarchy starting from [`ArmBase`], which I'm not going to further deal
  with. For each of the commonly supported SIMD instruction sets there is
  a class with static methods wrapping these intrinsics. The type inheritance
  is modelled after the _history_ of SIMD instruction sets, i.e. the type
  [`Avx`] inherits from [`Sse42`] which inherits from [`Sse41`], all the way up
  to [`Sse`] because a CPU supporting AVX also supports SSE 4.2, SSE 4.1
  up to the original SSE. Each of these types has a very important static
  property called `IsSupported` that indicates at runtime whether the
  wrapped instruction set is supported by the CPU the program is running
  on. A well written program should check these flags and provide suitable
  fallbacks in case a particular instruction set is not supported. As this
  complicates production code considerably, this blog post omits this.
* The number of SIMD instructions is quite overwhelming. Learning and
  understanding them is essential to writing efficient SIMD code. This
  article focuses on a very small subset of arithmetic floating point
  operations. There is no doubt that the implementation could be tuned
  for higher performance by leveraging other, more arcane, instructions.
  Doing so, however, would make the code much harder to understand. There is
  a hint of this obscurity in the `SimdExtensions.Abs()` and
  `SimdExtensions.All()` methods below.
* Good resources to learn about the SIMD intrinsics are the
  [Intel Intrinsics Guide] and the section about the intrinsics in the
  [Intel C++ Compiler Classic Developer Guide and Reference][Intel C++ Reference].
  Of course, it is also worthwhile to browse the [Microsoft intrinsics documentation],
  however the SIMD wrapper methods lack any descriptive text and I find it often
  more useful to refer to the Intel documentation instead.

Back to the implementation of the Runge-Kutta algorithm. First, a few helper
methods are created that simplify the actual algorithm. As in the previous
code, `using`'s are omitted. However, because they are unfamiliar to many
readers, note that `System.Runtime.Intrinsics` and
`System.Runtime.Intrinsics.X86` are required. Also, I make the bold
assumption that the CPU supports FMA, which was introduced back in 2013.
Hence, the code will be hard-wired to use `Vector256<double>`.

```cs
public static class SimdMathExtensions
{
  // Vector of negative zero, used to implement Abs()
  private static readonly Vector256<double> NegZero =
    Vector256.Create(-0.0);

  // Value returned by MoveMask() applied to a vector with all bits set.
  // Used by All()
  private static readonly int TrueMask =
    Avx.MoveMask(Vector256<double>.AllBitsSet);

  // Loads an array of doubles into an array of vectors.
  public static IEnumerable<Vector256<double>> Load(
    this double[] @this, double padding = 0)
  {
    var c = Vector256<double>.Count;
    var n = @this.Length - c + 1;
    var i = 0;
    for (; i < n; i += c)
    {
      yield return LoadInternal(@this, i);
    }

    // add one last vector with padding if necessary
    if (i < @this.Length)
    {
      var tail = new double[c];
      var j = 0;
      for (; i < @this.Length; ++i, ++j)
      {
        tail[j] = @this[i];
      }

      for (; j < Vector256<double>.Count; ++j)
      {
        tail[j] = padding;
      }

      yield return LoadInternal(tail, 0);
    }
  }

  private unsafe static Vector256<double> LoadInternal(double[] a, int i)
  {
    fixed (double* ap = a)
    {
      return Avx.LoadVector256(ap + i);
    }
  }

  // Reverse of Load()
  public static IEnumerable<T> Unpack<T>(this IEnumerable<Vector256<T>> @this)
    where T: struct
  {
    foreach (var v in @this)
    {
      for (var i = 0; i < Vector256<T>.Count; ++i)
      {
        yield return v.GetElement(i);
      }
    }
  }

  // Element-wise addition.
  public static IEnumerable<Vector256<double>> Add(
    this IEnumerable<Vector256<double>> @this,
    IEnumerable<Vector256<double>> other)
    => @this.Zip(other).Select(ab => Avx.Add(ab.First, ab.Second));

  // Element-wise multiplication.
  public static IEnumerable<Vector256<double>> Mul(
    this IEnumerable<Vector256<double>> @this,
    Vector256<double> other)
    => @this.Select(v => Avx.Multiply(v, other));

  // Multipyly-Add: a*@this[i]+c[i]
  public static IEnumerable<Vector256<double>> MulAdd(
    this IEnumerable<Vector256<double>> @this,
    Vector256<double> a,
    IEnumerable<Vector256<double>> c)
    => @this.Zip(c).Select(v => Fma.MultiplyAdd(a, v.First, v.Second));

  // Element-wise division.
  public static IEnumerable<Vector256<double>> Div(
    this IEnumerable<Vector256<double>> @this,
    IEnumerable<Vector256<double>> other)
    => @this.Zip(other).Select(ab => Avx.Divide(ab.First, ab.Second));

  // Element-wise absolute value.
  // See https://stackoverflow.com/a/5987631/159834
  public static Vector256<double> Abs(this Vector256<double> v)
    => Avx.AndNot(NegZero, v);

  // True if all values have all bits set (SIMD-variant of true), false
  // otherwise. See https://habr.com/en/post/467689/
  public static bool All(this IEnumerable<Vector256<double>> @this)
    => @this.Select(v => Avx.MoveMask(v)).All(i => i == TrueMask);

  // Minimum value.
  public static double Min(this IEnumerable<Vector256<double>> @this)
  {
    var tmp = @this.Aggregate((x, y) => Avx.Min(x, y));
    var tmp1 = Avx.Min(tmp.GetLower(), tmp.GetUpper());
    var result = Math.Min(tmp1.GetElement(0), tmp1.GetElement(1));
    return result;
  }

  // Maximum value.
  public static double Max(this IEnumerable<Vector256<double>> @this)
  {
    var tmp = @this.Aggregate((x, y) => Avx.Max(x, y));
    var tmp1 = Avx.Max(tmp.GetLower(), tmp.GetUpper());
    var result = Math.Max(tmp1.GetElement(0), tmp1.GetElement(1));
    return result;
  }
}
```

Next follows the actual implementation of the algorithm. Structurally it is
absolutely the same as before, however some changes were necessary to
accommodate the requirements of vector processing. One such thing is that
SIMD instructions can only ever operate on multiple packed values. Hence,
before a calculation can take place, the values to operate on need to be
loaded into the vectors. Because the vectors are of fixed size, special
measures have to be taken in case the number of values to operate on is not
an integer multiple of the vector length. Also, a bit of bit-fiddling is
required when Boolean operations come into play. Most of these idiosyncrasies
have been wrapped in the above `SimdMathExtensions` class. One exception is
the special treatment of the `epsilon` argument. If only a single value is
given by the caller, it is broadcast to every element of a new vector. Also,
the calculation order of some of the chained formulae has been reversed to
enable the use of the new `MulAdd()` extension method which performs a
multiplication and addition in a single instruction.

In order to stay [DRY], the algorithms coefficients have been extracted to a
common base class, `OdeBase` (not shown here).

```cs
public abstract class OdeSimd : OdeBase
{
  // helper for brevity
  private static Vector256<double> V(double d) => Vector256.Create(d);

  // Broadcast coefficients to vectors
  static readonly Vector256<double>
    VB21 = V(B21),
    VB31 = V(B31), VB32 = V(B32),
    VB41 = V(B41), VB42 = V(B42), VB43 = V(B43),
    VB51 = V(B51), VB52 = V(B52), VB53 = V(B53),
    VB54 = V(B54),
    VB61 = V(B51), VB62 = V(B52), VB63 = V(B53),
    VB64 = V(B64), VB65 = V(B65);

  static readonly Vector256<double>
    VCh1 = V(Ch1), VCh2 = V(Ch2), VCh3 = V(Ch3),
    VCh4 = V(Ch4), VCh5 = V(Ch5), VCh6 = V(Ch6);

  static readonly Vector256<double>
    VCt1 = V(Ct1), VCt2 = V(Ct2), VCt3 = V(Ct3),
    VCt4 = V(Ct4), VCt5 = V(Ct5), VCt6 = V(Ct6);

  public static IEnumerable<(double, double[])> Rk45(
    Func<
      Vector256<double>,
      IEnumerable<Vector256<double>>,
      IEnumerable<Vector256<double>>> func,
    double[] y0,
    double t0,
    double tEnd,
    double h,
    double hMax,
    double[] epsilon)
  {
    // SNIP -- PARAMETER VALIDATION

    // Initialization
    var y = y0.Load().ToArray();
    var t = t0;
    var veps = epsilon.Load(epsilon[0]).ToArray();

    // Output at t0
    yield return (t, y.Unpack().ToArray());

    // Perform iterations
    do
    {
      // Clamp h to not exceed hmax and not overshoot tend
      h = Math.Min(Math.Min(h, hMax), tEnd - t);

      var vh = Vector256.Create(h);

      // Calculate k1 through k6
      var k1 = func(V(t + h * A1), y).Mul(vh).ToArray();
      var k2 = func(V(t + h * A2), k1.MulAdd(
            VB21, y)).Mul(vh).ToArray();
      var k3 = func(V(t + h * A3), k2.MulAdd(
            VB32, k1.MulAdd(VB31, y))).Mul(vh).ToArray();
      var k4 = func(V(t + h * A4), k3.MulAdd(
            VB43, k2.MulAdd(VB42, k1.MulAdd(VB41, y)))).Mul(vh).ToArray();
      var k5 = func(V(t + h * A5), k4.MulAdd(
            VB54, k3.MulAdd(VB53, k2.MulAdd(VB52, k1.MulAdd(
              VB51, y))))).Mul(vh).ToArray();
      var k6 = func(V(t + h * A6), k5.MulAdd(
            VB65, k4.MulAdd(VB64, k3.MulAdd(VB63, k2.MulAdd(
              VB62, k1.MulAdd(VB61, y)))))).Mul(vh).ToArray();

      // Calculate error estimate for each component
      var te = k1.MulAdd(VCt1,
        k2.MulAdd(VCt2,
          k3.MulAdd(VCt3,
            k4.MulAdd(VCt4,
              k5.MulAdd(VCt5,
                k6.Mul(VCt6)))))).Select(v => v.Abs());

      // If only single epsilon given, reduce
      if (epsilon.Length == 1)
      {
        te = new[]{ V(te.Max()) };
      }

      // If error criterion is fulfilled, perform increment
      if (te.Zip(veps).Select(x =>
          Avx.CompareLessThanOrEqual(x.First, x.Second)).All())
      {
        y = k6.MulAdd(VCh6,
          k5.MulAdd(VCh5,
            k4.MulAdd(VCh4,
              k3.MulAdd(VCh3,
                k2.MulAdd(VCh2,
                  k1.MulAdd(VCh1, y)))))).ToArray();
        t += h;
      }

      // Adapt step size
      h *= 0.9 * Math.Pow(veps.Div(te).Min(), 1.0/5);
    } while (t < tEnd);

    // Output at tend
    yield return (t, y.Unpack().ToArray());
  }
}
```

![Piece of cake](https://media.giphy.com/media/3ohs80hRvix8UjrSSY/giphy.gif)

It is important to note that the interface of this solver is not exactly the
same as it was for the `OdeNaive` class. In particular, the delegate the caller
provides for the right-hand-side calculation needs to be vectorized as well and
operate on an array of `Vector256<T>`. Instead of repeating the `Program` class,
here just the vectorized function:

```cs
class Program
{
  // ... SNIP ...

  private static readonly oneHalf = Vector256.Create(-0.5);

  private static IEnumerable<Vector256<double>> RhsFunc(
    Vector256<double> t,
    IEnumerable<Vector256<double>> y) =>
      y.Select(yi => Avx.Multiply(oneHalf, yi))

  // ... SNAP ...
}
```

The results of running the SIMD implementation are shown below:

```sh
$ dotnet run -c Release
Average wall clock time: 1050.7 ms
t       y0           y1            y2            y3
0       1            100           51            35
10      0.006738     0.673810      0.343643      0.235834
```

First thing we notice is that there are slight deviations in the results.
This is to be expected for two reasons:

1. The order of evaluation of some of the expressions has been changed. While
   mathematically equivalent, floating point math can result in differences
   due to elimination and rounding.
2. Often the CPU uses higher precision internally when performing calculations
   than what is stored back in memory. If the CPU now carries out a chain of
   calculations in its registers only without round-tripping to memory, the
   end result will be of higher accuracy. When using SIMD, the calculations
   are scheduled differently on the CPU, and hence the results are expected
   to deviate.

But most importantly, did all the effort pay off? Looking at the average
wall clock time, a **40%** reduction can be observed. For more complex
simulations, real-time or low-latency applications, this would be a very
significant improvement indeed.

# A More Interesting Problem

The exponential decay problem is nice because it allows for easy validation.
However, it is also quite boring and pointless, exactly because the
analytical solution is so simple. So, what could be solved that is more
interesting? One problem that probably all mechanical engineering students
around the globe get confronted with is that of two coupled oscillators.
Imagine two masses that are connected by a spring and one of the masses is
additionally connected to a fixed wall. In this hypothetical problem, the
masses can only move in the direction that stretches or compresses the
springs, but not laterally. When the masses are released from positions
where the springs are not relaxed, they will start to oscillate. Because
the masses are coupled to each other, the oscillations of one mass will
influence the oscillation of the other mass and vice versa. Depending
on the ratio between the stiffness of the springs and the masses, the
oscillations will either be very regular or quite chaotic. Below is a
sketch of the system:

![Sketch of the oscillator system](/assets/images/simd/coupled_masses.svg)

\\(m_1\\) and \\(m_2\\) are the masses, \\(x_1\\) and \\(x_2\\) their
position (i.e. distance from the fixed wall) and \\(L_1\\) and \\(L_2\\)
are the positions at which the springs are relaxed (i.e. no forces are acting
on them).

The mathematical equations describing the system are quite simple:

$$
\begin{aligned}
  \frac{\mathrm{d} x_1}{\mathrm{d} t} &= v_1 \quad, \\
  \frac{\mathrm{d} x_2}{\mathrm{d} t} &= v_2 \quad, \\
  \frac{\mathrm{d} v_1}{\mathrm{d} t} &= \frac{-b_1 v_1 - k_1 (x_1 - L_1) + k_2 (x_2 - x_1 - L_2)}{m_1} \quad, \\
  \frac{\mathrm{d} v_2}{\mathrm{d} t} &= \frac{-b_2 v_2 - k_2 (x_2 - x_1 - L_2)}{m_2} \quad.
\end{aligned}
$$

The reasoning is also straight forward:

* The change in the mass' position \\(x\\) is given by its velocity
  \\(v\\).
* The change in velocity (i.e. the acceleration) is given by the forces acting
  on the mass, following Newton's law: \\(a = F / m \\).
* The force of the spring is proportional to the stretching or
  compression of the springs. It is assumed that the springs are linear
  according to Hooke's law: \\(F = k (x - L)\\), where \\(L\\) is the length at
  which the spring is relaxed.
* Finally, damping forces which are proportional to the velocity \\(v\\), such
  as friction, are given as \\(F = b v\\), where \\(b\\) is the friction
  coefficient.

![Trust me, I'm an Engineer](https://media.giphy.com/media/LqW9dLVjQm3cs/giphy.gif)

Note that the above set of four equations could actually be written as a set
of two equations by noting that the second derivative of the position is the
acceleration. However, numerical solvers for initial value problems can only
deal with differential equations of the first order. Higher order equations
can always be split into a larger set of first order equations such that this
restriction can be easily circumvented.

## The Vectorized Right-Hand-Side Function

Programming the right-hand-side function for above problem in a naïve manner
is straight forward and is skipped here. The vectorized SIMD version,
however, requires a bit more consideration. Conveniently, the
`Vector256<double>` type holds four values, exactly what is required to
represent our state vector. However, looking at the equations, we see that we
have two pairs of very similar equations and it would be easier to implement
them separately. Luckily, there is also `Vector128<double>` which holds two
values. The first pair of equations is trivial, however, the second pair
needs a bit more thinking:

* The equation for \\(v_2\\) is very similar to the one for \\(v_1\\).
  In order to vectorize the computation, we need to find a way to express
  them as one.
* The last equation has no term for the force created by the first spring.
  However, if we set the value for \\(k_1=0\\) in the last equation, the
  only difference that remains is that the third equation adds the force
  created by the second spring, while the fourth equation subtracts it. 
* As a work-around we could just set \\(k_2\\) to a positive value in
  the third equation and the corresponding negative value in the fourth
  equation, but that is hackish. Luckily, there is the
  [`Fma.MultiplySubtractAdd()`][MultiplySubtractAdd] method (a.k.a.
  [`_mm_fmsubadd_pd()`] or `VFMSUBADDPD`) which does exactly what we need by
  using addition on the first element and subtraction on the second.

Problems solved, let's get to it. First the coefficients are defined. I
tinkered around with them until I got a satisfactorily random behavior.
You might also want to try out different values.

```cs
// Frictionless system
Vector128<double> b = Vector128.Create(0.0, 0.0);

// First mass has 5kg, second mass has 1.2kg
Vector128<double> m = Vector128.Create(5.0, 1.2);

// The first spring has a stiffness of 10N/m, using the trick of setting the
// coefficient to 0 for the second equation.
Vector128<double> k1 = Vector128.Create(10, 0.0 /* always zero */);

// The second spring has a stiffness 3N/m
Vector128<double> k2 = Vector128.Create(3.0);

// The first spring is 3m long, the second 6m, hence L2=3m+6m=9m.
Vector128<double> L1 = Vector128.Create(3.0),
                  L2 = Vector128.Create(9.0);
```

Next, the right-hand-side (RHS) function is defined. I used a number of
temporary variables for intermediate expression to improve readability and
simplify the commenting:

```cs
IEnumerable<Vector256<double>>
GetCoupledMassesRhs(IEnumerable<Vector256<double>> y)
{
  // Extract the state vector and split it into position and velocity
  var y0 = y.First();
  var x = y0.GetLower();
  var v = y0.GetUpper();

  // Compute x1 - L1
  var dx1 = Avx.Subtract(Vector128.Create(x.GetElement(0)), L1);
  // Compute negative value of (x2 - x1 - L2) as (x1 - x2 + L2)
  // Avx.HorizontalSubtract() subtracts the second from the first element in the
  // first argument and assigns it to the first element in the destination. The
  // difference of the second argument is assigned to the second destination
  // element ==> the result here is [(x1 - x2), (x1 - x2)].
  var mdx2 = Avx.Add(Avx.HorizontalSubtract(x, x), L2);

  // Compute k2 * (x1 - x2 + L2)
  var tmp1 = Avx.Multiply(k2, mdx2);
  // Compute k1 * (x1 - L1) ∓ k2 * (x1 - x2 + L2)
  var tmp2 = Fma.MultiplySubtractAdd(k1, dx1, tmp1);
  // Compute -b * v - k1 * (x1 - L1) ± k2 * (x1 - x2 + L2)
  var tmp3 = Fma.MultiplySubtractNegated(b, v, tmp2);
  // Compute (-b * v - k1 * (x1 - L1) ± k2 * (x1 - x2 + L2)) / m
  var tmp4 = Avx.Divide(tmp3, m);
  // Reassemble the four-element vector
  var result = Vector256.Create(v, tmp4);

  yield return result;
}
```

![Boom. Easy as that.](https://media.giphy.com/media/3o7btNa0RUYa5E7iiQ/giphy.gif)

Running this simulation with the initial conditions \\(x_1 = 0\\),
\\(x_2 = 11\\), \\(v_1 = v_2 = 0\\) for 50 seconds (simulated time) and
plotting the positions of the masses over time results in this interesting
chart:

![The positions plotted over time](/assets/images/simd/chart.svg)

Putting the code through its paces by benchmarking it I obtained an average
wall-clock-time of 4845.65 ms for the SIMD code and for the naïve
implementation 7817.5 ms. That's a hefty **38%** speedup, consistent with what
we saw with the much simpler exponential decay problem before.

# Conclusions

I hope I succeeded in showing that SIMD instructions are a very powerful tool
to squeeze the last bit of performance from your algorithm. But as always
with optimization, it should never be done prematurely. Unless you have
profiled your code and proven that there is a performance bottleneck that can
be optimized with SIMD instructions, it is better to keep the more
expressive, easier to maintain _naïve_ code.

In this blog post I focused on a few floating point SIMD instructions,
mainly due to the example problem I picked which was familiar to me with my
engineering background. However, there are many more instructions for integer
arithmetic, cryptographic operations and string manipulation.

[Runge-Kutta-Fehlberg 4(5)]: https://en.wikipedia.org/wiki/Runge%E2%80%93Kutta%E2%80%93Fehlberg_method
[SIMD]: https://en.wikipedia.org/wiki/SIMD
[Initial value problems]: https://en.wikipedia.org/wiki/Initial_value_problem
[GitHub repository]: https://github.com/wildmichael/rk45
[MMX]: https://en.wikipedia.org/wiki/MMX_(instruction_set)
[SSE1]: https://en.wikipedia.org/wiki/Streaming_SIMD_Extensions
[AVX]: https://en.wikipedia.org/wiki/Advanced_Vector_Extensions
[`Vector64<T>`]: https://docs.microsoft.com/en-us/dotnet/api/system.runtime.intrinsics.vector64-1
[`Vector128<T>`]: https://docs.microsoft.com/en-us/dotnet/api/system.runtime.intrinsics.vector128-1
[`Vector256<T>`]: https://docs.microsoft.com/en-us/dotnet/api/system.runtime.intrinsics.vector256-1
[`Vector64`]: https://docs.microsoft.com/en-us/dotnet/api/system.runtime.intrinsics.vector64
[`Vector128`]: https://docs.microsoft.com/en-us/dotnet/api/system.runtime.intrinsics.vector128
[`Vector256`]: https://docs.microsoft.com/en-us/dotnet/api/system.runtime.intrinsics.vector256
[`X86Base`]: https://docs.microsoft.com/en-us/dotnet/api/system.runtime.intrinsics.x86.x86base
[`ArmBase`]: https://docs.microsoft.com/en-us/dotnet/api/system.runtime.intrinsics.arm.armbase
[`Avx`]: https://docs.microsoft.com/en-us/dotnet/api/system.runtime.intrinsics.x86.avx
[`Sse42`]: https://docs.microsoft.com/en-us/dotnet/api/system.runtime.intrinsics.x86.sse42
[`Sse41`]: https://docs.microsoft.com/en-us/dotnet/api/system.runtime.intrinsics.x86.sse41
[`Sse`]: https://docs.microsoft.com/en-us/dotnet/api/system.runtime.intrinsics.x86.sse
[DRY]: https://en.wikipedia.org/wiki/Don%27t_repeat_yourself
[MultiplySubtractAdd]: https://docs.microsoft.com/en-us/dotnet/api/system.runtime.intrinsics.x86.fma.multiplysubtractadd
[`_mm_fmsubadd_pd()`]: https://software.intel.com/sites/landingpage/IntrinsicsGuide/#text=_mm_fmsubadd_pd
[Intel Intrinsics Guide]: https://software.intel.com/sites/landingpage/IntrinsicsGuide
[Intel C++ Reference]: https://software.intel.com/content/www/us/en/develop/documentation/cpp-compiler-developer-guide-and-reference/top/compiler-reference/intrinsics.html
[Microsoft intrinsics documentation]: https://docs.microsoft.com/en-us/dotnet/api/system.runtime.intrinsics.x86