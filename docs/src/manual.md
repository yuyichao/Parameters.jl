# Parameters manual

This is a manual by example
([examples/ex1.jl](https://github.com/mauro3/Parameters.jl/blob/master/examples/ex1.jl)).

# Types with default values & keyword constructors

Create a type which has default values using [`@with_kw`](@ref):
```julia
using Parameters

@with_kw immutable PhysicalPara{R}
    rw::R = 1000.
    ri::R = 900.
    L::R = 3.34e5
    g::R = 9.81
    cw::R = 4220.
    day::R = 24*3600.
end
```


Now the type can be constructed using the default values, or with
non-defaults specified with keywords:
```julia
# Create an instance with the defaults
pp = PhysicalPara()
pp_f32 = PhysicalPara{Float32}() # the type parameter can be chosen explicitly
# Make one with some non-defaults
pp2 = PhysicalPara(cw=77.0, day= 987.0)
# Make another one based on the previous one with some modifications
pp3 = PhysicalPara(pp2; cw=.11e-7, rw=100.)
# the normal positional constructor can also be used
# (and should be used in hot inner loops)
pp4 = PhysicalPara(1,2,3,4,5,6)
```


To enforce constraints on the values, it's possible to use `@assert`s
straight inside the type-def.  (As usual, for mutables these
asserts can be violated by updating the fields after type construction.)
```julia
@with_kw immutable PhysicalPara2{R}
    rw::R = 1000.; @assert rw>0
    ri::R = 900.
    @assert rw>ri # Note that the placement of assertions is not
                  # relevant. (They are moved to the constructor.
end
```


Parameter interdependence is possible:
```julia
@with_kw immutable Para{R<:Real}
    a::R = 5
    b::R
    c::R = a+b
end
pa = Para(b=7)
```


Often the bulk of fields will have the same type.  To help with this,
a default type can be set.  Using this feature, the last example (with
additional field `d`) can be written more compactly as:
```julia
@with_kw immutable Para2{R<:Real} @deftype R
    a = 5
    b
    c = a+b
    d::Int = 4 # adding a type overrides the @deftype
end
pa2 = Para2(b=7)

# or more pedestrian
@with_kw immutable Para3 @deftype Float64
    a = 5
    b
    c = a+b
    d::Int = 4
end
pa3 = Para3(b=7)
```


Custom inner constructors can be defined as long as:

- one defining all positional arguments is given
- no zero-positional arguments constructor is defined (as that
  would clash with the keyword constructor)
- no `@assert`s (as in above example) are used within the type body.

The keyword constructor goes through the inner positional constructor,
thus invariants or any other calculation will be honored.
```julia
@with_kw immutable MyS{R}
    a::R = 5
    b = 4
    MyS(a,b) = (@assert a>b; new(a,b)) #
    MyS(a) = MyS{R}(a, a-1) # For this provide your own outer constructor:
end
MyS{R}(a::R) = MyS{R}(a)

MyS{Int}() # MyS(5,4)
ms = MyS(3) # MyS(3,2)
MyS(ms, b=-1) # MyS(3,-1)
try
    MyS(ms, b=6) # this will fail the assertion
end
```
Note that two of the main reasons to have an inner constructor,
assertions and simple calculations, are more easily achieved with
`@assert`s and parameter interdependence.

The macro `@with_kw` defines a show-method which is, hopefully, more informative than the
standard one.  For example the printing of the first example is:
```julia
julia> PhysicalPara()
PhysicalPara{Float64}
  rw: Float64 1000.0
  ri: Float64 900.0
  L: Float64 334000.0
  g: Float64 9.81
  cw: Float64 4220.0
  day: Float64 86400.0
```

If this `show` method definition is not desired, for instance because of method
re-definition warnings, then use [`@with_kw_noshow`](@ref).

# (Un)pack macros

When working with parameters, or otherwise, it is often convenient to
unpack (and pack) some or all of the fields of a type.  This is often
the case when passed into a function.

The preferred to do this is using the [`@unpack`](@ref) and [`@pack`](@ref) macros
which are generic and also work with non-`@with_kw` types and
dictionaries (and can be customized for other types too).  Continuing
with the `Para` type defined above:

```julia
function fn2(var, pa::Para)
    @unpack a, b = pa # equivalent to: a,b = pa.a,pa.b
    out = var + a + b
    b = 77
    @pack pa = b # equivalent to: pa.b = b
    return out, pa
end

out, pa = fn1(7, pa)
```



Example with a dictionary:
```julia
d = Dict{Symbol,Any}(:a=>5.0,:b=>2,:c=>"Hi!")
@unpack a, c = d
a == 5.0 #true
c == "Hi!" #true

d = Dict{Symbol,Any}()
@pack d = a, c
d # Dict{Symbol,Any}(:a=>5.0,:c=>"Hi!")
```



## Customization of `@unpack` and `@pack`

What happens during the (un-)packing of a particular datatype is
determined by the functions [`Parameters.unpack`](@ref) and
[`Parameters.pack!`](@ref).

The `Parameters.unpack` function is invoked to unpack one entity of some
`DataType` and has signature:

`unpack(dt::Any, ::Val{field}) -> value of field`

Two definitions are included in the package to unpack a composite type
or a dictionary with Symbol or string keys:
```
@inline unpack{f}(x, ::Val{f}) = getfield(x, f)
@inline unpack{k}(x::Associative{Symbol}, ::Val{k}) = x[k]
@inline unpack{S<:AbstractString,k}(x::Associative{S}, ::Val{k}) = x[string(k)]
```

The `Parameters.pack!` function is invoked to pack one entity into some
`DataType` and has signature:

`pack!(dt::Any, ::Val{field}, value) -> value`

Two definitions are included in the package to pack into a composite
type or into a dictionary with Symbol or string keys:

```
@inline pack!{f}(x, ::Val{f}, val) = setfield!(x, f, val)
@inline pack!{k}(x::Associative{Symbol}, ::Val{k}, val) = x[k]=val
@inline pack!{S<:AbstractString,k}(x::Associative{S}, ::Val{k}, val) = x[string(k)]=val
```

More methods can be added to `unpack` and `pack!` to allow for
specialized packing of datatypes.

## The type-specific (un)pack macros (somewhat dangerous)

The `@with_kw` macro automatically produces type-specific (un-)pack
macros of form `@unpack_TypeName` which unpack all fields:
```julia
function fn(var, pa::Para)
    @unpack_Para pa # the macro is constructed during the @with_kw
                    # and called @unpack_*
    out = var + a + b
    b = 77
    @pack_Para pa
    return out, pa
end

out, pa = fn(7, pa)
```

However, note that the (un-)packing macros which unpack all fields
have a few pitfalls, as changing the type definition will change what
local variables are available in a function using `@unpack_*`.  Examples:

- adding a field `pi` to a type would hijack `Base.pi` usage in any
  function using `@unpack_*`
- the `@unpack_*` will shadow an input argument of the function with
  the same name as a type-fieldname.  This I found very perplexing at
  times.

Thus, in general, it is probably better to use the `@(un)pack` macros instead.
