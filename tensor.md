
# Tensor.h

`./tensor.h` consists of header file of tensor data structure and functions

- CPU and GPU
- Shape
- Stream
- TRValue `:public expr::RValueExp<Container, DType>`
- Tensor `:public TRValue<Tensor<Device, dimension, DType>, Device, dimension, DType>`
- Tensor<Device, 1, DType> `:public TRValue<Tensor<Device, 1, DType>, Device, 1, DType>`

## 1. CPU and GPU

The struct `cpu` and `gpu` here is to guarantee the success of checkings before final evaluation.

```c++
// device name CPU
struct cpu {
  // whether this device is CPU or not
  static const bool kDevCPU = true;
  // device flag number, identifies this device
  static const int kDevMask = 1 << 0;
};
// device name GPU
struct gpu {
  // whether this device is CPU or not 
  static const bool kDevCPU = false;
  // device flag number, identifies this device 
  static const int kDevMask = 1 << 1;
};
```

## 2. Shape

The struct `Shape` here is to provide a flexible component for the construction of class Tensor.

### 2.1 Member variables

The `kDimension` defines the dimension of current shape, while the `kSubdim` can be used to infer 
the lowest dimension in the Tensor.

```c++
static const int kDimension = dimension;
static const int kSubdim = dimension - 1;
```

With the information of `kDimension`, we can construct an unsigned array `shape_` to store the shape of Tensor.

```c++
// typedef unsigned index_t;
index_t shape_[kDimension]; 
```

### 2.2 Constructor

After the difinition of three member variables, we can write the constructor to create a struct `Shape`.

```c++
// default constructor, do nothing  
MSHADOW_XINLINE Shape(void) {}
// constuctor
MSHADOW_XINLINE Shape(const Shape<kDimension> &s) {
  #pragma unroll
  for (int i = 0; i < kDimension; ++i) {
	this->shape_[i] = s[i];
  }
}
```

The code ```const Shape<kDimension> &s``` as a variable of constructor makes sure that only struct `Shape` with
same `kDimension` can be allowed to initiate the new ones.

### 2.3 Overloaded Operator

Operator `[]` is overloaded to return the sub-dimension of required index.

```c++
MSHADOW_XINLINE index_t &operator[](index_t idx) {
  return shape_[idx];
}
// the returned value is constant
MSHADOW_XINLINE const index_t &operator[](index_t idx) const {
  return shape_[idx];
}
```

Operator `==` and `!=` are overloaded to check whether two struct `Shape` are same.

```c++
// Shape<kDimension> implicitly check whether the input variable has the same size
MSHADOW_XINLINE bool operator==(const Shape<kDimension> &s) const {
  #pragma unroll
  for (int i = 0; i < kDimension; ++i) {
    if (s.shape_[i] != this->shape_[i]) return false;
  }
  return true;
}
// return whether two shape not equal
// s:         the shape to compare against
MSHADOW_XINLINE bool operator!=(const Shape<kDimension> &s) const {
  return !(*this == s);
}
```

Operator `<<` is overloaded to output the `shape_` of `Shape`.   

```c++
template<int dim>
  friend std::ostream &operator<<(std::ostream &os, const Shape<dim> &shape);
```

Above is only a declaration, while its definition is in `./tensor_cpu-inl.h`.

```c++
template<int ndim>
inline std::ostream &operator<<(std::ostream &os, const Shape<ndim> &shape) {
  os << '(';
  for (int i = 0; i < ndim; ++i) {
    if (i != 0) os << ',';
    os << shape[i];
  }
  // python style tuple
  if (ndim == 1) os << ',';
  os << ')';
  return os;
}
```



### 2.4 Member Functions

#### 2.4.1 Size

The `size` function returns the product of all sub-dimensions. e.g. `(5,3,6) -> 90`

```c++
MSHADOW_XINLINE size_t Size(void) const {
  size_t size = this->shape_[0];
  #pragma unroll
  for (int i = 1; i < kDimension; ++i) {
    size *= this->shape_[i];
  }
  return size;
}
```

#### 2.4.2 FlatTo1D

The `FlatTo1D` function returns a `Shape` with `kDimension=1` and its dimension equals to 
the product of original `Shape`. e.g. `(5,3,6) -> (90)`

```c++
MSHADOW_XINLINE Shape<1> FlatTo1D(void) const {
  Shape<1> s;
  s[0] = this->Size();
  return s;
}
```

#### 2.4.3 FlatTo2D

The `FlatTo2D` function returns a `Shape` with `kDimension=2` and its first dimension equals
to the product of original `Shape` instead of the lowest dimension, which is put into its second
dimension.


```warning
the reason of doing so will be considered and explained later
```

```c++
MSHADOW_XINLINE Shape<2> FlatTo2D(void) const {
  Shape<2> s;
  s.shape_[1] = this->shape_[kDimension - 1];
  index_t ymax = 1;
  #pragma unroll
  for (int i = 0; i < kDimension - 1; ++i) {
    ymax *= this->shape_[i];
  }
  s.shape_[0] = ymax;
  return s;
}
```

#### 2.4.4 ProdShape

The function `ProdShape` returns the product of shape in range `[dimstart, dimend)`.

```c++
MSHADOW_XINLINE index_t ProdShape(int dimstart, int dimend) const {
  index_t num = 1;
  #pragma unroll
  for (int i = dimstart; i < dimend; ++i) {
    num *= this->shape_[i];
  }
  return num;
}
```

#### 2.4.5 SubShape

The function `SubShape` return a new shape, whose `kDimension` is the minus `1` of original
one. e.g. `(3,2,6,4) -> (2,6,4)`

```warning
Since it is majorly built for cuda, its effectiveness will be considered and explained later
``` 

```c++
MSHADOW_XINLINE Shape<kSubdim> SubShape(void) const {
  Shape<kSubdim> s;
  // for cuda
  #pragma unroll
  for (int i = 0; i < kSubdim; ++i) {
    s.shape_[i] = this->shape_[i + 1];
  }
  return s;
}
```

#### 2.4.6 Slice

The function `Slice` return a new shape, whose `kDimension` is the `dimend-dimstart` of original
one. 

```c++
template<int dimstart, int dimend>
MSHADOW_XINLINE Shape<dimend - dimstart> Slice(void) const {
  Shape<dimend - dimstart> s;
  #pragma unroll
  for (int i = dimstart; i < dimend; ++i) {
    s[i - dimstart] = this->shape_[i];
  }
  return s;
}
```

It can used in the following way, e.g. `(3,4,5,6,7) -> (5,6,7)`.

```c++
// usage
Shape<5> ss = Shape5(3,4,5,6,7);
Shape<3> sss = ss.Slice<2,5>();
cout << sss <<endl;
// output (5,6,7)
```

### 2.5 Useful Construction

According to the `usage` instruction above, we introduce several construction functions
for struct `Shape` as APIs. 

```c++
// useful construction functions to generate shape
MSHADOW_XINLINE Shape<1> Shape1(index_t s0) {
  Shape<1> s; s[0] = s0;
  return s;
}

MSHADOW_XINLINE Shape<2> Shape2(index_t s0, index_t s1) {
  Shape<2> s; s[0] = s0; s[1] = s1;
  return s;
}

MSHADOW_XINLINE Shape<3> Shape3(index_t s0, index_t s1, index_t s2) {
  Shape<3> s;
  s[0] = s0; s[1] = s1; s[2] = s2;
  return s;
}

MSHADOW_XINLINE Shape<4> Shape4(index_t s0, index_t s1,
                                index_t s2, index_t s3) {
  Shape<4> s;
  s[0] = s0; s[1] = s1; s[2] = s2; s[3] = s3;
  return s;
}

MSHADOW_XINLINE Shape<5> Shape5(index_t s0, index_t s1, index_t s2,
                                index_t s3, index_t s4) {
  Shape<5> s;
  s[0] = s0; s[1] = s1; s[2] = s2; s[3] = s3; s[4] = s4;
  return s;
}
```

## 3. Stream

The `Stream` here is only a dummy implementation for CPU, we left it for further discussion when
we run into the implementation of GPU.

```c++
template<typename Device>
struct Stream {
  // this is only a dummy implementation for CPU
  // for GPU, the actual implementation will be specialized in tensor_gpu-inl.h
  
  //wait for all the computation associated with this stream to complete
  inline void Wait(void) {}

  // query whether the the stream is idle
  // return true if the stream is idle and all the job have been completed
  inline bool CheckIdle(void) {
    return true;
  }

  // create a blas handle
  inline void CreateBlasHandle() {}
};
```

## 4. TRValue
`:public expr::RValueExp<Container, DType>`

This is Tensor RValue, which is also the super type of all kinds of possible tensors.

```warning
The meaning of its existence is essential for the understanding of MShadow framework, thus 
we postpone its discussion into the core part of this tutorial
```

```c++
template<typename Container, typename Device, int dimension, typename DType>
struct TRValue: public expr::RValueExp<Container, DType> {
};
```

## 5. Tensor
`:public TRValue<Tensor<Device, dimension, DType>, Device, dimension, DType>`

The struct `Tensor` is the key element in MShadow.

```c++
//in /mshadow/base.h

//#ifndef MSHADOW_DEFAULT_DTYPE
//#define MSHADOW_DEFAULT_DTYPE = default_real_t
//#endif

//typedef float default_real_t;
//as a result, the default data type of Tensor is float
template<typename Device, int dimension, typename DType MSHADOW_DEFAULT_DTYPE>
struct Tensor: public TRValue<Tensor<Device, dimension, DType>, Device, dimension, DType> {
};	
```

```c++
// Trival Usage
Tensor<cpu, 3> ts(data, Shape3(2,5,2));
```
### 5.1 Member Variables

The variable `kDevCPU` indicates in which type of device the data are stored. And the `kSubdim` is
same as in the struct `Shape`.

```c++
static const bool kDevCPU = Device::kDevCPU;
static const int  kSubdim = dimension - 1;
```

The pointer `dptr_` points to the data wherever it is stored. Besides, the shape of data is 
controlled by the struct `Shape` to make it flexible to handle.  
```c++
DType *dptr_;
Shape<dimension> shape_;
```

At last, the `stride_` variable is used to deal with pitch allocation in gpu or 
sse (align x dimension to 64bit) for efficiency.

```warning
the concept of stream is important for GPU devices, we leave the discussion to there.
```


```c++
index_t stride_;
// stream where the computation lies
// stream is a device dependency concept
Stream<Device> *stream_;
```

### 5.2 Constructor

```warning
It is worth noting that the `stride_` is default initialized to be the lowest dimension of 
`Shape`. The reason of doing it is remained to be examined and discussed.
```


```c++
// default constructor
MSHADOW_XINLINE Tensor(void) : stream_(NULL) {}

// constructor from shape
MSHADOW_XINLINE Tensor(const Shape<dimension> &shape): shape_(shape), stream_(NULL) {}

// constructor from data pointer and shape, without stride
MSHADOW_XINLINE Tensor(DType *dptr, const Shape<dimension> &shape)
	: dptr_(dptr), shape_(shape), stride_(shape[kSubdim]), stream_(NULL) {}

// constructor from data pointer, shape and stream, without stride
MSHADOW_XINLINE Tensor(DType *dptr, const Shape<dimension> &shape,Stream<Device> *stream)
    : dptr_(dptr), shape_(shape), stride_(shape[kSubdim]), stream_(stream) {}

// constructor from data pointer, shape, stride and stream
MSHADOW_XINLINE Tensor(DType *dptr, const Shape<dimension> &shape, index_t stride, Stream<Device> *stream)
    : dptr_(dptr), shape_(shape), stride_(stride), stream_(stream) {}
```

### 5.3 Member Functions

#### 5.3.1 MSize and MemSize

The function `MSize` returns the memory cost of specified tensor, including the aligned x dimension
(so it starts with the value of largest dimension of tensor). While the function `MemSize` returns the 
memory starting from the `startdim`.

```c++
MSHADOW_XINLINE size_t MSize(void) const {
  return this->MemSize<0>();
}

template<int startdim>
MSHADOW_XINLINE size_t MemSize(void) const {
  size_t memsz = this->stride_;
  #pragma unroll
  for (int i = startdim; i < kSubdim; ++i) {
    memsz *= this->shape_[i];
  }
  return memsz;
}
```

#### 5.3.2 size

The function `size` return the `shape` of the specified sub-dimension. 

```c++
MSHADOW_XINLINE index_t size(index_t idx) const {
  return shape_[idx];
}
```

#### 5.3.3 FlatTo1D and FlatTo2D

The functions `FlatTo1D` and `FlatTo2D` return a new tensor with same data (unchanged `dptr_`), 
but different shape (refer to `FlatTo1D` and `FlatTo2D` in the context of `Shape`).

```c++
MSHADOW_XINLINE Tensor<Device, 1, DType> FlatTo1D(void) const {
  return Tensor<Device, 1, DType>(dptr_, shape_.FlatTo1D(), stride_, stream_);
}

MSHADOW_XINLINE Tensor<Device, 2, DType> FlatTo2D(void) const {
  return Tensor<Device, 2, DType>(dptr_, shape_.FlatTo2D(), stride_, stream_);
}
```

#### 5.3.4 Slice

The function `Slice` returns a new Tensor, which is a subset of the highest dimension.
e.g. `(128,3,224,224) -> (64,3,224,224)`

```c++
MSHADOW_XINLINE Tensor<Device, dimension, DType>
Slice(index_t begin, index_t end) const {
  Shape<dimension> s = this->shape_;
  s[0] = end - begin;
  return Tensor<Device, dimension, DType>(dptr_ + this->MemSize<1>() * begin, s, stride_, stream_);
}
```

### 5.4 Overloaded Operators

#### 5.4.1 Operator `[]`
The operator `[]` is overloaded to return the corresponding index in the highest dimension of `Tensor`.

The code `dptr_ + this->MemSize<1>() * idx` is to fetch the `idx` sub-tensor in `Tensor`, e.g. `(128,3,224,224)[5] -> 5-th (3,224,224)`

```c++
MSHADOW_XINLINE Tensor<Device, kSubdim, DType> operator[](index_t idx) const {
  return Tensor<Device, kSubdim, DType>(dptr_ + this->MemSize<1>() * idx, shape_.SubShape(), stride_, stream_);
}
```

#### 5.4.2 Operator `=`
The operator `=` is overloaded to be assignment operator when the `rhs` (right hand side) is also a `Tensor` variable.

```c++
// implement the assignment of same type
inline Tensor<Device, dimension, DType> &
operator=(const Tensor<Device, dimension, DType> &exp) {
  dptr_ = exp.dptr_;
  shape_ = exp.shape_;
  stride_ = exp.stride_;
  stream_ = exp.stream_;
  return *this;
}
```

However, if the `rhs` is a `scalar`, e.g. `3.0f`, or a `Exp` (expression) type, the operator `=` is overloaded to 
trigger the computation, which calls the `__assign` function, defined in its grandfather class `RValueExp`.


```c++
// we trigger computation at here
template<typename E, int etype>
inline Tensor<Device, dimension, DType> &
operator=(const expr::Exp<E, DType, etype> &exp) {
  return this->__assign(exp);
}

inline Tensor<Device, dimension, DType> &
operator=(const DType &exp) {
  return this->__assign(exp);
}
```

It is worth noting that there are several other assignment related operators are overloaded, 
but in the grandfather class `RValueExp`. We will do analysis until reaching there.


## 6. Missing Explanations

### 6.1 the usage of ```#pragma unroll```:

if the following ```for loop``` has a constant number of loops, the ```for loop``` will be expanded
in the compile time to accelerate the process.
e.g. ```for(i = 1; i < 10; i++){...}```; will be expanded

Otherwise, if the number of loop is undetermined, it will keep itself same as common loop
e.g. ```for(i = 1; i < n; i++){...};``` will be the same. Since computer will not be able to 
know the exact value of `n` until the computation time.

## 7. Missing Component

### 7.1 Shape::ConvertLayout

`ConvertLayout` is left to the discussion of MxNet, which uses it to do `convolution`.

```c++
// Convert shape in src_layout to shape in dst_layout
inline Shape<4> ConvertLayout(const Shape<4>& src, int src_layout, int dst_layout) {
  Shape<4> dst;
  switch (src_layout) {
  case kNCHW:
    dst = src;
    break;
  case kNHWC:
    dst[0] = src[0];
    dst[2] = src[1];
    dst[3] = src[2];
    dst[1] = src[3];
    break;
  default:
    LOG(FATAL) << "Invalid layout for 4d shape " << src_layout;
  }
  Shape<4> dst2;
  switch (dst_layout) {
  case kNCHW:
    return dst;
  case kNHWC:
    dst2[0] = dst[0];
    dst2[1] = dst[2];
    dst2[2] = dst[3];
    dst2[3] = dst[1];
    break;
  default:
    LOG(FATAL) << "Invalid layout for 4d shape " << src_layout;
  }
  return dst2;
}
// Convert shape in src_layout to shape in dst_layout
inline Shape<5> ConvertLayout(const Shape<5>& src, int src_layout, int dst_layout) {
  Shape<5> dst;
  switch (src_layout) {
  case kNCDHW:
    dst = src;
    break;
  case kNDHWC:
    dst[0] = src[0];
    dst[2] = src[1];
    dst[3] = src[2];
    dst[4] = src[3];
    dst[1] = src[4];
    break;
  default:
    LOG(FATAL) << "Invalid layout for 5d shape " << src_layout;
  }
  Shape<5> dst2;
  switch (dst_layout) {
  case kNCDHW:
    return dst;
  case kNDHWC:
    dst2[0] = dst[0];
    dst2[1] = dst[2];
    dst2[2] = dst[3];
    dst2[3] = dst[4];
    dst2[4] = dst[1];
    break;
  default:
    LOG(FATAL) << "Invalid layout for 5d shape " << src_layout;
  }
  return dst2;
}
```

### 7.2 Tensor:set_stream and Tensor::CheckContiguous

These two functions are heavily related to GPU implementation, so they are left for further discussion.

```c++
inline void set_stream(Stream<Device> *stream) {
  this->stream_ = stream;
}

MSHADOW_XINLINE bool CheckContiguous(void) const {
  return this->shape_[dimension - 1] == stride_;
}
```

### 7.3 Tensor<Device, 1, DType>

We must respecialize struct `Tensor` in the `1D` situation, since the implementation of 
overloaded operator `[]` is different.

It can also be considered as a review of member variables and functions in original `Tensor`.

```c++
template<typename Device, typename DType>
struct Tensor<Device, 1, DType>:
      public TRValue<Tensor<Device, 1, DType>, Device, 1, DType> {
 public:
  DType *dptr_;
  Shape<1> shape_;
  index_t stride_;
  Stream<Device> *stream_;
  // constructor
  MSHADOW_XINLINE Tensor(void) : stream_(NULL) {}
  MSHADOW_XINLINE Tensor(const Shape<1> &shape)
      : shape_(shape), stream_(NULL) {}
  MSHADOW_XINLINE Tensor(DType *dptr, Shape<1> shape)
      : dptr_(dptr), shape_(shape), stride_(shape[0]), stream_(NULL) {}
  MSHADOW_XINLINE Tensor(DType *dptr, Shape<1> shape, Stream<Device> *stream)
      : dptr_(dptr), shape_(shape), stride_(shape[0]), stream_(stream) {}
  MSHADOW_XINLINE Tensor(DType *dptr, Shape<1> shape,
                         index_t stride, Stream<Device> *stream)
      : dptr_(dptr), shape_(shape), stride_(stride), stream_(stream) {}
  inline void set_stream(Stream<Device> *stream) {
    this->stream_ = stream;
  }
  MSHADOW_XINLINE Tensor<Device, 1, DType> FlatTo1D(void) const {
    return *this;
  }
  MSHADOW_XINLINE Tensor<Device, 2, DType> FlatTo2D(void) const {
    return Tensor<Device, 2, DType>(dptr_, shape_.FlatTo2D(), stride_, stream_);
  }
  MSHADOW_XINLINE Tensor<Device, 1, DType> Slice(index_t begin, index_t end) const {
    Shape<1> s;
    s[0] = end  - begin;
    return Tensor<Device, 1, DType>(dptr_ + begin, s, s[0], stream_);
  }
  MSHADOW_XINLINE bool CheckContiguous(void) const {
    return true;
  }
  MSHADOW_XINLINE size_t MSize(void) const {
    return shape_[0];
  }
  MSHADOW_XINLINE index_t size(index_t i) const {
    return shape_[0];
  }
  MSHADOW_XINLINE DType &operator[](index_t idx) {
    return dptr_[idx];
  }
  MSHADOW_XINLINE const DType &operator[](index_t idx) const {
    return dptr_[idx];
  }
  // implement the assignment of same type
  inline Tensor<Device, 1, DType> &
  operator=(const Tensor<Device, 1, DType> &exp) {
    dptr_ = exp.dptr_;
    shape_ = exp.shape_;
    stride_ = exp.stride_;
    stream_ = exp.stream_;
    return *this;
  }
  template<typename E, int etype>
  inline Tensor<Device, 1, DType> &
  operator=(const expr::Exp<E, DType, etype> &exp) {
    return this->__assign(exp);
  }
  inline Tensor<Device, 1, DType> &operator=(const DType &exp) {
    return this->__assign(exp);
  }
};
``` 