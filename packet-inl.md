
# Expr_engine-inl.h

`Packet` is designed to further speed up computation. 

- Alignment
- Packet
- Packet Operator
- Saver
- Packet Plan
- Packet Check
- Packet Align Check
- MapPacketPlan

A general `Packet` has two modes to choose:

```c++
enum PacketArch {
  kPlain,
  kSSE2,
};
```

- `kPlain` is just a plain version using standard C language in case SSE is not available
- `kSSE2` is a set of instructions allowing to perform computations on packets of `128` bits 
  at once. Since a float is `32` bits, this means that `SSE2` instructions can handle `4` 
  floats at once, which also means that, if correctly used, they can make our computation go 
  up to `4x` faster.

```note
However, in real applications, e.g. we have chosen `size=50`, so our vectors consist of `50` 
float, and `50` is not a multiple of `4`. This means that we cannot hope to do all 
of that computation using SSE2 instructions. The second best thing, to which we should aim, 
is to handle the `48` first coefficients with SSE2 instructions, since `48` is the biggest 
multiple of `4` below `50`, and then handle separately, without SSE2, the `49th` and `50th` 
coefficients.
```

As a result, we define a struct containing the required aligned bytes

```c++
template<PacketArch Arch>
struct AlignBytes {
  // typedef unsigned index_t;
  static const index_t value = 4;   
};
```

## Packet

`Packet` is a struct that is designed to handle the evaluation process of expression,
in a scalar level, by using SSE2 instructions set for better efficiency. 

It has three different basic type `<DType, kPlain`> for standard C computation,
`<float, kSSE2>` for float SSE2 optimization, and `<double, kSSE2>` for double
SSE2 optimization. All of them will be described in parallel.

### Member Variables

- `kSize`: number of float can be handled as a vector
- `data_`: the internal data that is loaded from a pointer pointing to a built-in scalar type

Definition in `<DType, kPlain>`:

```c++
// since kPlain is only defined as default C computation in case of the absense of SSE2
// it naturally can only handle 1 float a time
static const index_t kSize = 1;
DType data_;
```

Definition in `<float, kSSE2>` and `<double, kSSE2>`:

```c++
// since float size is 32, it can handle 4 floats a time
static const index_t kSize = 4;
// __m128 is a special type in SSE2
__m128 data_;
```

```c++
// since double size is 64, it can only handle 2 doubles a time
static const index_t kSize = 2;
__m128d data_;
```

The vectorization makes `4` floats or `2` doubles looks like a `scalar`.

### Constructor

Since `Packet` only has `2` member variables, and one of them is static const type, 
we only need initiate the `data_` in constructor.

Definition in `<DType, kPlain>`:

```c++
Packet(void) {}
explicit Packet(DType data) : data_(data) {}
```

Definition for `<float, kSSE2>` and `<double, kSSE2>` is same:

```c++
Packet(void) {}
explicit Packet(__m128d data) : data_(data) {}
```

### Member Functions

The member functions in `<DType, kPlain>` just use the simple C / C++ code to realize same 
function in `<float/double, kSSE2>`.

```c++
// create a fill with the target value s
MSHADOW_CINLINE static Packet<DType, kPlain> Fill(DType s) {
  return Packet<DType, kPlain>(s);
}

// load from address src, using *src for the value
MSHADOW_CINLINE static Packet<DType, kPlain> Load(const DType* src) {
  return Packet<DType, kPlain>(*src);
}

// load from address src, using *src for the value
MSHADOW_CINLINE static Packet<DType, kPlain> LoadUnAligned(const DType* src) {
  return Packet<DType, kPlain>(*src);
}

// store data into dst
MSHADOW_CINLINE void Store(DType* dst) const {
  *dst = data_;
}
```

Compared to plain implementation, the realization of `<float/double, kSSE2>` requires
the usage of `intrinsics`, included in file `<emmintrin.h>`.

- `_mm_set1_ps(w)`: set all the four floats to the value of `w`, e.g. `r0 := r1 := r2 := r3 := w`
- `_mm_load_ps(w)`: load the first four floats pointed by `w`, e.g. `r0 := p[0]`, `r1 := p[1]`, `r2 := p[2]`, `r3 := p[3]`.
                    The address must be 16-byte aligned.
- `_mm_loadu_ps(w)`: same as `_mm_load_ps`, but the address does not need to be 16-byte aligned.
- `_mm_store_ps(*p,a)`: transfer the `4` floats values in `a` to the address pointed by `p`. 
  e.g. `p[0] := a0`, `p[1] := a1`, `p[2] := a2`, `p[3] := a3`

```c++
MSHADOW_CINLINE static Packet<float, kSSE2> Fill(float s) {
  return Packet<float, kSSE2>(_mm_set1_ps(s));
}

MSHADOW_CINLINE static Packet<float, kSSE2> Load(const float* src) {
  return Packet<float, kSSE2>(_mm_load_ps(src));
} 

MSHADOW_CINLINE static Packet<float, kSSE2> LoadUnAligned(const float* src) {
  return Packet<float, kSSE2>(_mm_loadu_ps(src));
}

MSHADOW_CINLINE void Store(float* dst) const {
  _mm_store_ps(dst, data_);
}
```

For `<double, kSSE2>`, the process and instructions are all similar, instead of the 
final `ps` is changed to `pd` for dealing with `double`.

```c++
MSHADOW_CINLINE static Packet<double, kSSE2> Fill(double s) {
  return Packet<double, kSSE2>(_mm_set1_pd(s));
}

MSHADOW_CINLINE static Packet<double, kSSE2> Load(const double* src) {
  return Packet<double, kSSE2>(_mm_load_pd(src));
}

MSHADOW_CINLINE static Packet<double, kSSE2> LoadUnAligned(const double* src) {
  return Packet<double, kSSE2>(_mm_loadu_pd(src));
}

MSHADOW_CINLINE void Store(double* dst) const {
  _mm_store_pd(dst, data_);
}
``` 

```note
We intentionally leave the discussion of Sum() function, since its application is scarce in 
our context and has heave extra SSE2 instructions.
``` 


### Overloaded Operators

#### `=` operator

The `=` operator simply change the value of `data_` to the value in right hand side, and 
return itself by `*this`.

```c++
// plain
MSHADOW_CINLINE Packet<DType, kPlain>& operator=(DType s) {
  data_ = s;
  return *this;
}

// <float, kSSE2>
MSHADOW_CINLINE Packet<float, kSSE2>& operator=(float s) {
  data_ = _mm_set1_ps(s);
  return *this;
}

// <double, kSSE2>
MSHADOW_CINLINE Packet<double, kSSE2>& operator=(double s) {
  data_ = _mm_set1_pd(s);
  return *this;
}
```

#### `+` / `-` / `*` / `/` operators

For Plain version:

```c++
template<typename DType>
MSHADOW_CINLINE Packet<DType, kPlain> operator+(const Packet<DType, kPlain>& lhs,
                                                const Packet<DType, kPlain>& rhs) {
  return Packet<DType, kPlain>(lhs.data_ + rhs.data_);
}

template<typename DType>
MSHADOW_CINLINE Packet<DType, kPlain> operator-(const Packet<DType, kPlain>& lhs,
                                                const Packet<DType, kPlain>& rhs) {
  return Packet<DType, kPlain>(lhs.data_ - rhs.data_);
}
template<typename DType>
MSHADOW_CINLINE Packet<DType, kPlain> operator*(const Packet<DType, kPlain>& lhs,
                                                    const Packet<DType, kPlain>& rhs) {
  return Packet<DType, kPlain>(lhs.data_ * rhs.data_);
}

template<typename DType>
MSHADOW_CINLINE Packet<DType, kPlain> operator/(const Packet<DType, kPlain>& lhs,
                                                    const Packet<DType, kPlain>& rhs) {
  return Packet<DType, kPlain>(lhs.data_ / rhs.data_);
}
```

For SSE2 version:

```c++
MSHADOW_CINLINE Packet<float, kSSE2> operator+(const Packet<float, kSSE2>& lhs,
                                                    const Packet<float, kSSE2>& rhs) {
  return Packet<float, kSSE2>(_mm_add_ps(lhs.data_, rhs.data_));
}

MSHADOW_CINLINE Packet<double, kSSE2> operator+(const Packet<double, kSSE2>& lhs,
                                                     const Packet<double, kSSE2>& rhs) {
  return Packet<double, kSSE2>(_mm_add_pd(lhs.data_, rhs.data_));
}

MSHADOW_CINLINE Packet<float, kSSE2> operator-(const Packet<float, kSSE2>& lhs,
                                                    const Packet<float, kSSE2>& rhs) {
  return Packet<float, kSSE2>(_mm_sub_ps(lhs.data_, rhs.data_));
}

MSHADOW_CINLINE Packet<double, kSSE2> operator-(const Packet<double, kSSE2>& lhs,
                                                     const Packet<double, kSSE2>& rhs) {
  return Packet<double, kSSE2>(_mm_sub_pd(lhs.data_, rhs.data_));
}

MSHADOW_CINLINE Packet<float, kSSE2> operator*(const Packet<float, kSSE2>& lhs,
                                                    const Packet<float, kSSE2>& rhs) {
  return Packet<float, kSSE2>(_mm_mul_ps(lhs.data_, rhs.data_));
}

MSHADOW_CINLINE Packet<double, kSSE2> operator*(const Packet<double, kSSE2>& lhs,
                                                     const Packet<double, kSSE2>& rhs) {
  return Packet<double, kSSE2>(_mm_mul_pd(lhs.data_, rhs.data_));
}


MSHADOW_CINLINE Packet<float, kSSE2> operator/(const Packet<float, kSSE2>& lhs,
                                                    const Packet<float, kSSE2>& rhs) {
  return Packet<float, kSSE2>(_mm_div_ps(lhs.data_, rhs.data_));
}

MSHADOW_CINLINE Packet<double, kSSE2> operator/(const Packet<double, kSSE2>& lhs,
                                                     const Packet<double, kSSE2>& rhs) {
  return Packet<double, kSSE2>(_mm_div_pd(lhs.data_, rhs.data_));
}
```


## Alignment

To make fully use of SSE2 instructions set, Alignment is important in the context.

### Aligned and Aligned Free

`AlignedMallocPitch()` is an analog to `cudaMallocPitch()`, which allocates a aligned
space with `num_line * lspace` cells.

- `out_pitch`: output parameter, the actuall space allocated for each line
- `lspace`: number of cells required for each line
- `num_line`: number of lines to be allocated

First, according to `lspace`, `out_pitch` can be calculated as the smallest number that
is larger than `lspace` and can be divided by `16`. Then, the aligned space to be allocated
can be calculated by `out_pitch * num_line`.

```c++
inline void* AlignedMallocPitch(size_t *out_pitch,
                                size_t lspace,
                                size_t num_line) {
  const index_t bits = AlignBytes<MSHADOW_DEFAULT_PACKET>::value;   // = 4
  const index_t mask = (1 << bits) - 1;   // = 15

  // e.g. ((25 + 15) >> 4) << 4 = 32;
  size_t pitch = ((lspace + mask) >> bits) << bits; 
                                                      
  *out_pitch = pitch;
#ifdef _MSC_VER
  void *res = _aligned_malloc(pitch * num_line, 1 << bits);
#else
  void *res;
  int ret = posix_memalign(&res, 1 << bits, pitch * num_line);
  CHECK_EQ(ret, 0) << "AlignedMallocPitch failed";
#endif
  if (res == NULL) {
    LOG(FATAL) << "AlignedMallocPitch failed";
  }
  return res;
}
```

As usual, since we allocate a space, we should free it after using by

```c++
inline void AlignedFree(void *ptr) {
#ifdef _MSC_VER
  _aligned_free(ptr);
#else
  free(ptr);
#endif
}
```

### CheckAlign

`CheckAlign` is designed to make sure a value or the address of a pointer can be divided by 16.

```c++
// check whether the value can be divided by 16
template<PacketArch Arch>
inline bool CheckAlign(size_t pitch) {
  const index_t bits = AlignBytes<Arch>::value;
  // the aligned pitch is expected to hold 4 zeros in its lowest 4 bits, or just can be divided by 16.

  // thus, when it is & with 15, the result should be zero
  return !(pitch & ((1 << bits) - 1)); 
}

// check whether the stored address of a pointer can be divided by 16
template<PacketArch Arch>
inline bool CheckAlign(void *ptr) {
  // reinterpret_cast forces the complier thinking ptr as a size_t type
  return CheckAlign<Arch>(reinterpret_cast<size_t>(ptr)); 
}
```

### UpperAlign and LowerAlign

`UpperAlign` and `LowerAlign` return the required loop times to execute a string with `size`.

According to the example in the beginning, it is a common strategy to use `LowerAlign` and then
handle remaining separately. If `UpperAlign` is used, padding is required.

```c++
template<typename DType, PacketArch Arch>
inline index_t UpperAlign(index_t size) {
  const index_t bits = AlignBytes<MSHADOW_DEFAULT_PACKET>::value;   // = 4
  const index_t mask = (1 << bits) - 1;   // = 15
  const index_t fsize = sizeof(DType);    // = e.g. 4 for float or 8 for double
  return (((size * fsize + mask) >> bits) << bits) / fsize;
}

template<typename DType, PacketArch Arch>
inline index_t LowerAlign(index_t size) {
  const index_t bits = AlignBytes<MSHADOW_DEFAULT_PACKET>::value;   // = 4
  const index_t fsize = sizeof(DType);    // = e.g. 4 for float or 8 for double
  // e.g. (((50 * 4) >> 4) << 4) / 4 = 48 as claimed in the beginning example.
  return (((size * fsize) >> bits) << bits) / fsize;
}
```

## Packet Operator

`PacketOp` is a struct same as the operator defined in the `op` namespace. The different choice of 
`PacketOp` is trigger by the given typename `OP`.

Its main usage is to deal with the operation between differnt `Packet`. Each `+` / `-` / `*` / `/` is 
defined in the file `./packet/plain-inl.h` or `./packet/sse-inl.h`.

```c++
// specialization of operators
template<typename DType, PacketArch Arch>
struct PacketOp<op::plus, DType, Arch> {
  static const bool kEnabled = true;
  MSHADOW_CINLINE static Packet<DType, Arch> Map(const Packet<DType, Arch>& lhs,
                                                   const Packet<DType, Arch>& rhs) {
    return lhs + rhs;
  }
};

template<typename DType, PacketArch Arch>
struct PacketOp<op::minus, DType, Arch> {
  static const bool kEnabled = true;
  MSHADOW_CINLINE static Packet<DType, Arch> Map(const Packet<DType, Arch>& lhs,
                                                  const Packet<DType, Arch>& rhs) {
    return lhs - rhs;
  }
};
template<typename DType, PacketArch Arch>
struct PacketOp<op::mul, DType, Arch> {
  static const bool kEnabled = true;
  MSHADOW_CINLINE static Packet<DType, Arch> Map(const Packet<DType, Arch>& lhs,
                                                  const Packet<DType, Arch>& rhs) {
    return lhs * rhs;
  }
};

template<typename DType, PacketArch Arch>
struct PacketOp<op::div, DType, Arch> {
  static const bool kEnabled = true;
  MSHADOW_CINLINE static Packet<DType, Arch> Map(const Packet<DType, Arch>& lhs,
                                                  const Packet<DType, Arch>& rhs) {
    return lhs / rhs;
  }
};

template<typename DType, PacketArch Arch>
struct PacketOp<op::identity, DType, Arch> {
  static const bool kEnabled = true;
  MSHADOW_CINLINE static Packet<DType, Arch> Map(const Packet<DType, Arch>& src) {
    return src;
  }
};
```

if the operator is out of range of `op::plus`, `op::minus`, `op::mul`,
`op::div` and `op:identity`, the declaration will in its default setting, 
which leads it fail to pass the checking and back to original C++ implement.

```c++
template<typename OP, typename DType, PacketArch Arch>
struct PacketOp {
  static const bool kEnabled = false;
};
```



## Saver

`Saver` is the struct that calls its member function `Save()` to do actual evaluation and saving.

It has two formulations. One for evaluation and saving (e.g. `+=` / `-=` / `*=` / `/=`), 
and one only for saving (e.g. `=`) with operator `sv::saveto`. 

For the first `Save()`, 
we first change the left hand side `*dst` to be a `Packet`, then call `PacketOp::Map()`
to finish evaluation. At last, the result is stored to `*dst`.

```c++
template<typename SV, typename TFloat, PacketArch Arch>
struct Saver{
  MSHADOW_CINLINE static void Save(TFloat *dst, const Packet<TFloat, Arch>& src) {
    Packet<TFloat, Arch> lhs = Packet<TFloat, Arch>::Load(dst);   
    Packet<TFloat, Arch> ans = PacketOp<typename SV::OPType, TFloat, Arch>::Map(lhs, src);
    ans.Store(dst);
  }
};
```
For the second `Save()`, we simply store the result to `*dst`.

```c++
template<typename TFloat, PacketArch Arch>
struct Saver<sv::saveto, TFloat, Arch> {
  MSHADOW_CINLINE static void Save(TFloat *dst, const Packet<TFloat, Arch>& src) {
    src.Store(dst);
  }
};
```

## Packet Plan

Same as the class `Plan` defined in `expr_engine-inl.h`, the class `PacketPlan` is the class to provide
`Eval()` function for `Packet`, which is the way to do actual evaluation.

### Declaration

According to the declaration, a `PacketPlan` contains two member functions `EvalPacket()` which use `SSE` optimization,
and `Eval()` to do common computation as in `Plan`.

```c++
typedef packet::PacketArch PacketArch;

// same as plan, but use packet
template<typename ExpType, typename DType, PacketArch Arch>
class PacketPlan {
 public:
  MSHADOW_CINLINE packet::Packet<DType, Arch> EvalPacket(index_t y, index_t x) const;
  MSHADOW_CINLINE DType Eval(index_t y, index_t x) const;
};
```

### Tensor PacketPlan

The usage of `EvalPacket()` is to provide the data according to `dptr_[y * stride_ + x]`

```c++
template <typename Device, int dim, typename DType, PacketArch Arch>
class PacketPlan<Tensor<Device, dim, DType>, DType, Arch> {
 public:
  explicit PacketPlan(const Tensor<Device, dim, DType> &t)
      :dptr_(t.dptr_), stride_(t.stride_) {}
  MSHADOW_CINLINE packet::Packet<DType, Arch> EvalPacket(index_t y, index_t x) const {
    return packet::Packet<DType, Arch>::Load(&dptr_[y * stride_ + x]);
  }
  MSHADOW_CINLINE DType Eval(index_t y, index_t x) const {
    return dptr_[y * stride_ + x];
  }

 private:
  const DType  *dptr_;
  index_t stride_;
};
```

### Scalar PacketPlan

The usage of `EvalPacket()` is to provide `scalar_` for every `x` and `y`.

```c++
template<typename DType, PacketArch Arch>
class PacketPlan<ScalarExp<DType>, DType, Arch> {
 public:
  explicit PacketPlan(DType scalar) : scalar_(scalar) {}
  MSHADOW_CINLINE packet::Packet<DType, Arch> EvalPacket(index_t y, index_t x) const {
    return packet::Packet<DType, Arch>::Fill(scalar_);
  }
  MSHADOW_CINLINE DType Eval(index_t y, index_t x) const {
    return scalar_;
  }

 private:
  DType scalar_;
};
```

### Binary PacketPlan

The usage of `EvalPacket()` is to perform `EvalPacket()` for `lhs_` and `rhs_` respectively, and 
use the pre-defined `OP` to do the combinational evaluation.

```c++
template<typename OP, typename TA, typename TB, int etype, typename DType, PacketArch Arch>
class PacketPlan<BinaryMapExp<OP, TA, TB, DType, etype>, DType, Arch> {
 public:
  PacketPlan(const PacketPlan<TA, DType, Arch> &lhs, const PacketPlan<TB, DType, Arch> &rhs)
      : lhs_(lhs), rhs_(rhs) {}
  MSHADOW_CINLINE packet::Packet<DType, Arch> EvalPacket(index_t y, index_t x) const {
    return packet::PacketOp<OP, DType, Arch>::Map(lhs_.EvalPacket(y, x), rhs_.EvalPacket(y, x));
  }
  MSHADOW_CINLINE DType Eval(index_t y, index_t x) const {
    return OP::Map(lhs_.Eval(y, x), rhs_.Eval(y, x));
  }

 private:
  PacketPlan<TA, DType, Arch> lhs_;
  PacketPlan<TB, DType, Arch> rhs_;
};
```

### Unary PacketPlan

The usage of `EvalPacket()` is to perform `EvalPacket()` for `src_` first, and 
use the pre-defined `OP` to do the compositional evaluation.

```c++
template<typename OP, typename TA, int etype, typename DType, PacketArch Arch>
class PacketPlan<UnaryMapExp<OP, TA, DType, etype>, DType, Arch> {
 public:
  PacketPlan(const PacketPlan<TA, DType, Arch> &src) : src_(src) {}
  MSHADOW_CINLINE packet::Packet<DType> EvalPacket(index_t y, index_t x) const {
    return packet::PacketOp<OP, DType, Arch>::Map(src_.EvalPacket(y, x));
  }
  MSHADOW_CINLINE DType Eval(index_t y, index_t x) const {
    return OP::Map(src_.Eval(y, x));
  }

 private:
  PacketPlan<TA, DType, Arch> src_;
};
```

### MapPacketPlan

#### RValueExp

The `MapPacketPlan()` for `RValueExp` is only to return a `PacketPlan` initiated by the 
subtype of `RValueExp<T, DType>`, which is always a `Tensor`.


```c++
template<PacketArch Arch, typename T, typename DType>
inline PacketPlan<T, DType, Arch> MakePacketPlan(const RValueExp<T, DType> &e) {
  return PacketPlan<T, DType, Arch>(e.self());
}
```

#### ScalarExp

The `MapPacketPlan()` for `ScalarExp` is only to return a `PacketPlan` initiated by the 
member variable `scalar_`:

```c++
template<PacketArch Arch, typename DType>
inline PacketPlan<ScalarExp<DType>, DType, Arch> MakePacketPlan(const ScalarExp<DType> &e) {
  return PacketPlan<ScalarExp<DType>, DType, Arch>(e.scalar_);
}
```

#### BinaryMapExp

The `MapPacketPlan()` for `BinaryMapExp` is to recursively called `MapPacketPlan()` for its
`lhs` and `rhs` at first stage, and then return the `PacketPlan` initiated by the two results.

```c++

template<PacketArch Arch, typename OP, typename TA, typename TB, typename DType, int etype>
inline PacketPlan<BinaryMapExp<OP, TA, TB, DType, etype>, DType, Arch>
MakePacketPlan(const BinaryMapExp<OP, TA, TB, DType, etype> &e);

template<PacketArch Arch, typename OP, typename TA, typename TB, typename DType, int etype>
inline PacketPlan<BinaryMapExp<OP, TA, TB, DType, etype>, DType, Arch>
MakePacketPlan(const BinaryMapExp<OP, TA, TB, DType, etype> &e) {
  return PacketPlan<BinaryMapExp<OP, TA, TB, DType, etype>,
                    DType, Arch>(MakePacketPlan<Arch>(e.lhs_), MakePacketPlan<Arch>(e.rhs_));
}
```

#### UnaryMapExp

The `MapPacketPlan()` for `UnaryMapExp` is to call `MapPacketPlan()` of its input variable `src`,
then construct the `PacketPlan` for the return

```c++
template<PacketArch Arch, typename OP, typename TA, typename DType, int etype>
inline PacketPlan<UnaryMapExp<OP, TA, DType, etype>, DType, Arch>
MakePacketPlan(const UnaryMapExp<OP, TA, DType, etype> &e) {
  return PacketPlan<UnaryMapExp<OP, TA, DType, etype>, DType, Arch>(MakePacketPlan<Arch>(e.src_));
}
```

## Packet Check

The major usage of `PacketCheck` is to make sure every related `DType` is `float` or `double`,
and every related `operator` is defined in the context of `Packet`. Otherwise, the program
will back to the original C++ implement.

```c++
template<PacketArch Arch>
struct PacketCheck<float, Arch> {
  static const bool kPass = true;
};

template<PacketArch Arch>
struct PacketCheck<double, Arch> {
  static const bool kPass = true;
};

template<typename DType, PacketArch Arch>
struct PacketCheck<ScalarExp<DType>, Arch> {
  static const bool kPass = PacketCheck<DType, Arch>::kPass;
};

template<int dim, typename DType, PacketArch Arch>
struct PacketCheck<Tensor<cpu, dim, DType>, Arch> {
  static const bool kPass = PacketCheck<DType, Arch>::kPass;
};

// check both operator implementation and data type in Tensor
template<typename OP, typename TA, typename DType, int etype, PacketArch Arch>
struct PacketCheck<UnaryMapExp<OP, TA, DType, etype>, Arch> {
  static const bool kPass = PacketCheck<TA, Arch>::kPass &&
      packet::PacketOp<OP, DType, Arch>::kEnabled;
};

// check both operator implementation and data type in Tensor
template<typename OP, typename TA, typename TB, typename DType, int etype, PacketArch Arch>
struct PacketCheck< BinaryMapExp<OP, TA, TB, DType, etype>, Arch> {
  static const bool kPass = packet::PacketOp<OP, DType, Arch>::kEnabled &&
      PacketCheck<TA, Arch>::kPass && PacketCheck<TB, Arch>::kPass;
};
```

For some expressions, like `TransposeExp()`, and scalars, like `int` of built-in type
which is not implemented in `Packet`, the `PacketCheck` is automatically failed. As a result, 
commmon C++ code will be used to evaluate such expression.

The following code makes sure the default setting is `false`.

```c++
template<typename E, PacketArch Arch>
struct PacketCheck{
  static const bool kPass = false;
};
```

## Packet Align Check

`PacketAlignCheck` is to check whether the data is aligned and the expressions
has been implemented.

### Default

First, for some expressions that is not implemented, like `TransposeExp()`, the 
check should automatically fail and return the evaluation of expressions to their 
original implementations. So following codes guarantee the automatic switch.

```c++
template<int dim, typename E, PacketArch Arch>
struct PacketAlignCheck {
  inline static bool Check(const E &exp) {
    return false;
  }
};
```

### Tensor

In the `Check()` function, it checks whether the address is 16-byte aligned, and the `stride_`
is available for packet (can be divided by `4` for `float`).

```note
For now, I highly wonder the importance of `stride_` checking. Since we can always compute
the parts that can be divided by `4` and calculate the remaining as written by itself in 
function `MapPacketPlan()`. However, the author just ignores it obviously.

The reason is remained to be checking !!!
```

```c++
template<int dim, typename DType, PacketArch Arch>
struct PacketAlignCheck<dim, Tensor<cpu, dim, DType>, Arch> {
  inline static bool Check(const Tensor<cpu, dim, DType> &t) {
    return packet::CheckAlign<Arch>(t.dptr_) &&
        packet::CheckAlign<Arch>(t.stride_ * sizeof(DType));
  }
};
```

### ScalarExp

The `ScalarExp` is naturally aligned, so we always return `true`.

```c++
template<int dim, typename DType, PacketArch Arch>
struct PacketAlignCheck<dim, ScalarExp<DType>, Arch> {
  inline static bool Check(const ScalarExp<DType> &exp) {
    return true;
  }
};
```

### BinaryMapExp

It first recursively calls `Check()` function of its `lhs` and `rhs`, and combine them 
by `&&` in the end.

```c++
template<int dim, typename OP, typename TA, typename TB,
         typename DType, int etype, PacketArch Arch>
struct PacketAlignCheck<dim, BinaryMapExp<OP, TA, TB, DType, etype>, Arch> {
  inline static bool Check(const BinaryMapExp<OP, TA, TB, DType, etype> &t) {
    return PacketAlignCheck<dim, TA, Arch>::Check(t.lhs_) &&
        PacketAlignCheck<dim, TB, Arch>::Check(t.rhs_);
  }
};
```

### UnaryMapExp

It just does `Check()` for its variable `src`

```c++
template<int dim, typename OP, typename TA, typename DType, int etype, PacketArch Arch>
struct PacketAlignCheck<dim, UnaryMapExp<OP, TA, DType, etype>, Arch> {
  inline static bool Check(const UnaryMapExp<OP, TA, DType, etype> &t) {
    return PacketAlignCheck<dim, TA, Arch>::Check(t.src_);
  }
};
```

## MapPacketPlan

`MapPacketPlan` finishes the evaluation and final assignment.

```note
I wonder why the `Tensor` here is 2-D. One possible explanation is the `MapPlan()` 
function in `./tensor_cpu-inl.h` is 2-D for the usage of transpose. But the transpose
is not written in the context of Packet. So either the transpose can be added or 
just change it to 1-D ???
```

```c++
template<typename SV, typename E, int dim, typename DType, PacketArch Arch>
inline void MapPacketPlan(Tensor<cpu, dim, DType> _dst,
                          const expr::PacketPlan<E, DType, Arch>& plan) {
  Tensor<cpu, 2, DType> dst = _dst.FlatTo2D();
  const index_t xlen = packet::LowerAlign<DType, Arch>(dst.size(1));
  for (index_t y = 0; y < dst.size(0); ++y) {
    for (index_t x = 0; x < xlen; x += packet::Packet<DType, Arch>::kSize) {
      packet::Saver<SV, DType, Arch>::Save(&dst[y][x], plan.EvalPacket(y, x));
    }
    for (index_t x = xlen; x < dst.size(1); ++x) {
      SV::Save(dst[y][x], plan.Eval(y, x));
    }
  }
}
```

The code above implement a two-step evaluation strategy, but in current code, the 
second step can not happen. This is due to the check of `stride_`. Its necessary 
is under question and remains to be checking.

## Missing Explanations

### void* pointer

The type void* is a special pointer type that can hold the address of any object.

Like any other pointer, a void* pointer holds an address, but the type of the object at
that address is unknown.

### size_t keyword

`size_t` is a machine-specific unsigned type that is guaranteed to be large enough to hold 
the size of any object in memory. The `size_t` type is defined in the `cstddef` header.

## Missing Components

### Packet::Sum()

For Plain packet:

```c++
MSHADOW_CINLINE DType Sum() const {
  return data_;
}
```

For SSE2 packet:

```c++
MSHADOW_CINLINE float Sum() const {
  __m128 ans  = _mm_add_ps(data_, _mm_movehl_ps(data_, data_));
  __m128 rst  = _mm_add_ss(ans, _mm_shuffle_ps(ans, ans, 1));
#if defined(_MSC_VER) && (_MSC_VER <= 1500) && defined(_WIN64)
  return rst.m128_f32[0];
#else
  float rr = _mm_cvtss_f32(rst);
  return rr;
#endif
}

inline double Sum(void) const {
  __m128d tmp =  _mm_add_sd(data_, _mm_unpackhi_pd(data_, data_));
#if defined(_MSC_VER) && (_MSC_VER <= 1500) && defined(_WIN64)
  return tmp.m128d_f64[0];
#else
  double ans = _mm_cvtsd_f64(tmp);
  return ans;
#endif
}
```

### MapPacketPlan for MakeTensorExp

this function will never be called, since the `MakeTensorExp` type will not pass the Check() function.
So we tempararily ignore it.

```c++
template<PacketArch Arch, typename T, int dim, typename DType>
inline PacketPlan<T, DType, Arch>
MakePacketPlan(const MakeTensorExp<T, cpu, dim, DType> &e) {
  return PacketPlan<T, DType, Arch>(e.real_self());
}
```
