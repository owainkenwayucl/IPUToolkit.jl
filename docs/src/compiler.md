# Writing codelets in Julia

The `IPUToolkit.IPUCompiler` submodule allows you to write [codelets](https://docs.graphcore.ai/projects/poplar-user-guide/en/3.2.0/vertices_overview.html) for the IPU in Julia.
Codelets are defined with the [`@codelet`](@ref) macro, and then you can use them inside a program, written using the interface to the Poplar SDK described before.
This mechanism uses the [`GPUCompiler.jl`](https://github.com/JuliaGPU/GPUCompiler.jl) package, which is a generic framework for generating LLVM IR code for specialised targets, not limited to GPUs despite the historical name.

Examples of codelets written in Julia are shown in the files [`examples/main.jl`](https://github.com/giordano/IPUToolkit.jl/blob/main/examples/main.jl), [`examples/adam.jl`](https://github.com/giordano/IPUToolkit.jl/blob/main/examples/adam.jl), and [`examples/pi.jl`](https://github.com/giordano/IPUToolkit.jl/blob/main/examples/pi.jl).

The code inside a codelet has the same limitations as all the compilation models based on [`GPUCompiler.jl`](https://github.com/JuliaGPU/GPUCompiler.jl):

* the code has to be statically inferred and compiled, dynamic dispatch is not admitted;
* you cannot use functionalities which require the Julia runtime, most notably the garbage collector;
* you cannot call into any other external binary library at runtime, for example you cannot call into a BLAS library.

After defining a codelet with `@codelet` you can add a vertex calling this codelet to the graph with the function [`add_vertex`](@ref), which also allows controlling the tile mapping in a basic way.

```@docs
@codelet
VertexVector
add_vertex
IPUCompiler.KEEP_LLVM_FILES
IPUCompiler.POPC_FLAGS
IPUCompiler.PROGRESS_SPINNER
```

## IPU builtins

Inside codelets defined with [`@codelet`](@ref) all calls to random functions

* `rand(Float32)`
* `rand(UInt32)`
* `rand(UInt64)`

result to call to corresponding IPU builtins for [random number generation](https://docs.graphcore.ai/projects/poplar-api/en/latest/ipu_intrinsics/ipu_builtins.html#random-number-generation), but with the general semantic of the Julia function `rand` (numbers uniformely distributed in the $[0, 1)$ range).

Additionally, you can use the [IPU builtins](https://docs.graphcore.ai/projects/poplar-api/en/latest/ipu_intrinsics/ipu_builtins.html) listed below.

```@docs
get_scount_l
get_tile_id
```

## Printing

Inside codelets you can print text and value of variables using the macros [`@ipuprintf`](@ref), [`@ipuprint`](@ref), [`@ipuprintln`](@ref), and [`@ipushow`](@ref).
These macros are useful for debugging purposes but printing inside a codelet might incur performance penalty.
To completely disable all printing and make these macros no-op you can set [`IPUCompiler.DISABLE_PRINT`](@ref):
```julia
IPUCompiler.DISABLE_PRINT[] = true
```

```@docs
@ipuprintf
@ipuprint
@ipuprintln
@ipushow
IPUCompiler.DISABLE_PRINT
```

## Benchmarking

To benchmark expressions inside codelets you can use the macros [`@ipucycles`](@ref), [`@ipushowcycles`](@ref), and [`@ipuelapsed`](@ref), which report the number of cycles spent in the wrapped expression.
They are similar to Julia's `@time`, `@showtime`, and `@elapsed` macros, but report the number of cycles, as the clockspeed of tiles cannot be easily obtained _inside_ a codelet.
The corresponding time can be obtained by dividing the number of cycles by the clock frequency of the the tile, which you can get with [`Poplar.TargetGetTileClockFrequency(target)`](https://docs.graphcore.ai/projects/poplar-api/en/latest/poplar/device/Target.html#_CPPv4NK6poplar6Target21getTileClockFrequencyEv) outside of the codelet, and should usually be 1.330 GHz or 1.850 GHz depending on the model of your IPU.
The printing macros `@ipucycles` and `@ipushowcycles` can be made completely no-op by setting [`IPUCompiler.DISABLE_PRINT`](@ref).

!!! warning

    Timing of expressions taking longer than `typemax(UInt32) / tile_clock_frequency` (about 2 or 3 seconds depending on your IPU model) is unreliable because the difference between the starting and the ending cycle counts would overflow.

```@docs
@ipucycles
@ipushowcycles
@ipuelapsed
```

## Passing non-constant variables from global scope

If your kernel references a non-constant (`const`) global variable, the generated code will result in a reference to a memory address on the host, and this will fatally fail at runtime because programs running on the IPU don't have access to the host memory.
Constant variables are not affected by this problem because their values are inlined when the function is compiled.
If you can't or don't want to make a variable constant you can interpolate its value with a top-level `@eval` when defining the codelet.
For example:

```julia
using IPUToolkit.IPUCompiler, IPUToolkit.Poplar
device = Poplar.get_ipu_device()
target = Poplar.DeviceGetTarget(device)
graph = Poplar.Graph(target)
tile_clock_frequency = Poplar.TargetGetTileClockFrequency(target)
@eval IPUCompiler.@codelet graph function test(invec::VertexVector{Float32, In}, outvec::VertexVector{Float32, Out})
    # We can use the intrinsic `get_scount_l` to get the cycle counter right
    # before and after some operations, so that we can benchmark it.
    cycles_start = get_scount_l()
	# Do some operations here...
    cycles_end = get_scount_l()
    # Divide the difference between the two cycle counts by the tile frequency
    # clock to get the time.
    time = (cycles_end - cycles_start) / $(tile_clock_frequency)
	# Show the time spent doing your operations
    @ipushow time
end
```

The use of `@eval` allows you not to have to pass an extra argument to your kernel just to use the value of the variable inside the codelet.

## Domain-Specific Language: `@ipuprogram`

The `IPUCompiler.@ipuprogram` macro provides a very simple and limited DSL to automatically generate most of the boilerplate code needed when writing an IPU program.
You can do *very* little with this DSL, which is mainly a showcase of Julia's meta-programming capabilities.
A fully commented examples of use of the `@ipuprogram` macro is available in the [`examples/dsl.jl`](https://github.com/giordano/IPUToolkit.jl/blob/main/examples/dsl.jl) file.