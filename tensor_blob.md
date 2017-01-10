
# Tensor_blob.h

`./tensor_blob.h` consists of common representation of arbitrary dimension tensor, which can be used to 
transforme to normal fixed dimension tensor.

- TShape
- TBlob


## TShape

`TShape` is a dynamic shape class (not a template class) that can hold shape of arbitrary dimension.

The shape will be stored in `data_stack_` when dimension is smaller than `kStackCache`.
When it is bigger, it will be stored in `data_heap_`.

### Member Variables

- `kStackCache`: size of space in stack.
- `ndim_`: number of dimensions of the shape.
- `num_heap_allocated_`: number of cells allocated in `data_heap_`.
- `data_stack_[kStackCache]`: stack space used to store shape when dimension is small.
- `*data_heap_`: space to store shape when dimension is big.

```c++
static const index_t kStackCache = 4;
index_t ndim_;
index_t num_heap_allocated_;
index_t data_stack_[kStackCache];
index_t *data_heap_;
```

### Constructors

#### Default Constructor

The default constructor can be a good way to know which variable is required 
to be initialized. 

In this case, we have to initiate `ndim_` (kep variable
in the context), `num_heap_allocated_` (to decide whether uses heap), and
`data_heap_`. 

Since `kStackCache` is a static const, and `data_stack_` is 
automatically initialized, we can ignore them.

```c++
TShape()
    : ndim_(0),
      num_heap_allocated_(0),
      data_heap_(NULL) {}
```

#### Explicit Constructor

The explicit constructor constructs an "all-one" `TShape` with given dimensions.

```c++
explicit TShape(index_t ndim)
    : ndim_(ndim) {
  if (ndim_ <= kStackCache) {
    data_heap_ = NULL;
    num_heap_allocated_ = 0;
    std::fill_n(data_stack_, ndim_, 1);
  } else {
    data_heap_ = new index_t[ndim_];
    num_heap_allocated_ = ndim_;
    std::fill_n(data_heap_, ndim_, 1);
  }
}
```

#### Constructor from TShape

The idea is same as the explicit constructor. The only difference is it is an itentical
copy by using `std::copy()`.

```c++
TShape(const TShape &s)
    : ndim_(s.ndim_) {
  if (ndim_ <= kStackCache) {
    data_heap_ = NULL;
    num_heap_allocated_ = 0;
    std::copy(s.data_stack_, s.data_stack_ + ndim_, data_stack_);
  } else {
    data_heap_ = new index_t[ndim_];
    num_heap_allocated_ = ndim_;
    std::copy(s.data_heap_, s.data_heap_ + ndim_, data_heap_);
  }
}
```

#### Constructor from RandomAccessIterator

The beginning of this constructor is same as a default constructor, the 
difference lies in the calling of member function `CopyFrom()`.

```c++
template<typename RandomAccessIterator>
TShape(RandomAccessIterator begin,
       RandomAccessIterator end)
    : ndim_(0),
      num_heap_allocated_(0),
      data_heap_(NULL) {
  this->CopyFrom(begin, end);
}
```

The `CopyFrom()` function first set the dimension of shape by calling `SetDim()`.
Then, copy the data from range `[first,end)` to `data()`, which is determined by
`ndim_` compared to `kStackCache`.

```c++
template<typename RandomAccessIterator>
inline void CopyFrom(RandomAccessIterator begin,
                     RandomAccessIterator end) {
  this->SetDim(end - begin);
  std::copy(begin, end, data());
}
```

`SetDim()` also choose to initiate `data_heap_` or not by the value of `ndim_`.

```c++
inline void SetDim(index_t dim) {
  if (dim > kStackCache &&
      dim > num_heap_allocated_) {
    // data_heap_ can be NULL
    delete [] data_heap_;
    data_heap_ = new index_t[dim];
    num_heap_allocated_ = dim;
  }
  ndim_ = dim;
}

inline index_t *data() {
  return ndim_ <= kStackCache ? data_stack_ : data_heap_;
}
```

#### Move Constructor from TShape

According to the property of move constructor, we first transfer the member variables
from input arguments to the new `TShape`. Then, the original pointer of `s.data_heap_`
is reset to `NULL`.

```c++
TShape(TShape &&s)
    : ndim_(s.ndim_),
      num_heap_allocated_(s.num_heap_allocated_),
      data_heap_(s.data_heap_) {
  if (ndim_ <= kStackCache) {
    std::copy(s.data_stack_, s.data_stack_ + ndim_, data_stack_);
  }
  // remove data heap space from s
  s.data_heap_ = NULL;
}
```

#### Move Constructor from Shape

Similar to constructor from `RandomAccessIterator`, the beginning of this constructor 
is same as a default constructor, the difference lies in the calling of member function 
`CopyFrom()`.

The `CopyFrom()` function takes in two iterators as its input. It first set the dimension 
of shape by calling `SetDim()`. Then, copy the data from range `[first,end)` to `data()`, 
which is determined by `ndim_` compared to `kStackCache`.

```c++
template<int dim>
TShape(Shape<dim> &&s)
    : ndim_(0),
      num_heap_allocated_(0),
      data_heap_(NULL) {
  this->CopyFrom(s.shape_, s.shape_ + dim);
}
```

#### Destructor

Since `data_heap_` is the only member variables with a pointer-related type, we explicitly
delete it in destructor.

```c++
~TShape() {
  // data_heap_ can be NULL
  delete [] data_heap_;
}
```

### Overloaded Operators

#### Overloaded `=`

##### Overloaded from `Tshape`

It first sets the shape of itself to `shape.ndim_`, then assigns the first address of data
(`data_stack_` or `data_heap_`) to a temp variable `src`. At last, it copies the contents
pointed by `src` to the `data()` of itself.

```c++
inline TShape &operator=(const TShape &shape) {
  this->SetDim(shape.ndim_);
  const index_t *src = shape.data();
  std::copy(src, src + ndim_, data());
  return *this;
}
```

##### Overloaded from `std::vector`

The `CopyFrom()` function takes in two iterators as its input, which is suitable for 
the two variables `shape.begin()` and `shape.end()`. It first set the dimension 
of shape by calling `SetDim()`. Then, copy the data from range `[first,end)` to `data()`, 
which is determined by `ndim_` compared to `kStackCache`.

```c++
inline TShape &operator=(const std::vector<index_t> &shape) {
  this->CopyFrom(shape.begin(), shape.end());
  return *this;
}
```

##### Overloaded from `Shape`

It first sets the shape of itself to `dim`. Then, it definite a pointer pointing to 
`data_stack_` or `data_heap_` by comparing `dim` to `kStackCache`. At last, it just
do a element-wise assignment from `shape` to itself.

```c++
template<int dim>
inline TShape &operator=(const Shape<dim> &shape) {
  this->SetDim(dim);
  index_t *d = dim <= kStackCache ? data_stack_ : data_heap_;
  for (int i = 0; i < dim; ++i) {
    d[i] = shape[i];
  }
  return *this;
}
```

#### Overloaded `[]`

It returns the dimension by calling `data()` to fetch the first
address, and uses operator `[]` to reach the value.

```warning
Is it safe to leave the bound check out ???
```

```c++
inline index_t &operator[](index_t i) {
  return data()[i];
}

inline const index_t &operator[](index_t i) const {
  return data()[i];
}
```

#### Overloaded `==`

##### Overloaded from `TShape`

It returns whether two shape equals.

```c++
inline bool operator==(const TShape &s) const {
  if (ndim_ != s.ndim_) return false;
  if (ndim_ <= kStackCache) {
    for (index_t i = 0; i < ndim_; ++i) {
      if (data_stack_[i] != s.data_stack_[i]) return false;
    }
  } else {
    for (index_t i = 0; i < ndim_; ++i) {
      if (data_heap_[i] != s.data_heap_[i]) return false;
    }
  }
  return true;
}
```

##### Overloaded from `Shape`

It returns whether two shape equals.

```c++
template<int dim>
inline bool operator==(const Shape<dim> &s) const {
  if (ndim_ != dim) return false;
  const index_t *d = dim <= kStackCache ? data_stack_ : data_heap_;
  for (index_t i = 0; i < dim; ++i) {
    if (d[i] != s.shape_[i]) return false;
  }
  return true;
}
```

#### Overloaded `!=`

##### Overloaded from `TShape`

It returns whether two shape not equals.

```c++
inline bool operator!=(const TShape &s) const {
  return !(*this == s);
}
```

##### Overloaded from `Shape`

It returns whether two shape not equals.

```c++
template<int dim>
inline bool operator!=(const Shape<dim> &s) const {
  return !(*this == s);
}
```

#### Overloaded from `<<`

It outputs a python-style tuple as in the class `Shape`.

```c++
inline std::ostream &operator<<(std::ostream &os, const TShape &shape) {
  os << '(';
  for (index_t i = 0; i < shape.ndim(); ++i) {
    if (i != 0) os << ',';
    os << shape[i];
  }
  // python style tuple
  if (shape.ndim() == 1) os << ',';
  os << ')';
  return os;
}
```

#### Overloaded from `>>`

It reads the input `shape` from the istream and assigns it to a `TShape` type.

There are two `while` loops in the overloaded function for 1-D shape and multi-dimensional
shape respectively.

in the first `while` loop, we first peek the next character in 
a input stream. Then, we judge whether it is a digit or not.

if it is a digit, it means we have a shape with only one dimension.
Thus, we input it to a variable named `idx`, set the `ndim_` of 
`shape` to be `1`, and copy the value of `idx` to `data()` (Refer
to the definition of `CopyFrom()`).

Otherwise, we use `is.get()` to extract the character, but do not 
assign it to anywhere. Again, we judge whether the input is a `(` to support
multi-dimension. If it is, we break the `while` loop and move to the
second part. If not, we judge whether it is a `space`. If true, we 
continue the process to see what is the next character. Otherwise,
we set the `failbit` to disable the following `istream` and return.

If we receive a `(` in the first part, meaning we will receive the shape of a 
multi-dimensional tensor, we move to the next part. 

In the beginning, we define a `index_t` type `idx` to receive the elements of 
each dimension, and a `std::vector` to represent the complete shape.

In the second `while` loop, we first fetch the next character by `is>>idx`.
It automatically neglects any `space` and check the success of read. It may set a 
error bit if read fails and break the `while` loop, e.g. input a char `a` to `idx`.

If the input is correct `index_t` type, we push it into the vector, and recursively
get the next character until it is not a `space`. 

If next is a `L` as the representation of `long` type in Python, we do the fetch again
to receive next character.

If next is `,`, then we recursively peek the next character until it is not a space.
If the peeked variable is `)`, we break the whole `while` loop. Otherwise we return to 
the start of `while` loop.

At last, we copy `tmp` to `shape`, even there is a false input, e.g. `(3,4,a)`. We will 
receive a `tmp` with value `(3,4)` and similarly copy it to `shape`.

```c++
inline std::istream &operator>>(std::istream &is, TShape &shape) {
  // first part
  while (true) {
    char ch = is.peek();
    if (isdigit(ch)) {
      index_t idx;
      if (is >> idx) {
        shape.CopyFrom(&idx, &idx + 1);
      }
      return is;
    }
    is.get();
    if (ch == '(') break;
    if (!isspace(ch)) {
      is.setstate(std::ios::failbit);
      return is;
    }
  }

  // second part
  index_t idx;
  std::vector<index_t> tmp;
  while (is >> idx) {
    tmp.push_back(idx);
    char ch;
    do {
      ch = is.get();
    } while (isspace(ch));
    if (ch == 'L') {
      ch = is.get();
    }
    if (ch == ',') {
      while (true) {
        ch = is.peek();
        if (isspace(ch)) {
          is.get(); continue;
        }
        if (ch == ')') {
          is.get(); break;
        }
        break;
      }
      if (ch == ')') break;
    } else if (ch == ')') {
      break;
    } else {
      is.setstate(std::ios::failbit);
      return is;
    }
  }
  shape.CopyFrom(tmp.begin(), tmp.end());
  return is;
}
```

Usage:

```c++
int main() {
  TShape a;
  cout << (cin >> a) << endl;
  return 0;
}

// correct example
// 3          ->    (3,)
// (3,5)      ->    (3,5)
// (3 , 5)    ->    (3,5)
// (3, 4L, 5) ->    (3,4,5)
// incorrect example
// a          ->    wrong!
// (3,4,a)    ->    (3,4)
```

### Member Functions

#### CopyFrom

The `CopyFrom()` function first set the dimension of shape by calling `SetDim()`
with argument equaling the difference between iterators `begin` and `end`.
Then, copy the data from range `[begin,end)` to `data()`, which is determined by
`ndim_` compared to `kStackCache`.

```c++
template<typename RandomAccessIterator>
inline void CopyFrom(RandomAccessIterator begin,
                     RandomAccessIterator end) {
  this->SetDim(end - begin);
  std::copy(begin, end, data());
}
```

#### data

It returns the data content of the `TShape`. 

The reason that these two functions can be overloaded
is they are called according to the constness of their
corresponding variables. If a variable is `const`, then 
the `const` version will be called, and vice versa.

Actually, it may just according to the implicit `this`
argument, by checking `this` is a `const` type or not.

```c++
inline const index_t *data() const {
  return ndim_ <= kStackCache ? data_stack_ : data_heap_;
} 

inline index_t *data() {
  return ndim_ <= kStackCache ? data_stack_ : data_heap_;
}
```

#### ndim

It simply returns the private member variable `ndim_`.

```c++
inline index_t ndim(void) const {
  return ndim_;
}
``` 

#### Size

It returns the multiplication of the values from all dimensions.

Its return type is `size_t`, which is a machine-related type that is large enough 
to hold any size can be stored in memory.

```c++
inline size_t Size(void) const {
  size_t size = 1;
  const index_t *d = this->data();
  for (index_t i = 0; i < ndim_; ++i) {
    size *= d[i];
  }
  return size;
}
```

#### FlatTo2D

It flattens the higher dimension of `TShape` to second dimension, returns a 2D `shape`.

```c++
inline Shape<2> FlatTo2D(void) const {
  Shape<2> s;
  if (ndim_ == 0) return Shape2(0, 0);
  const index_t *d = this->data();
  s.shape_[1] = d[ndim_ - 1];
  index_t ymax = 1;
  for (index_t i = 1; i < ndim_; ++i) {
    ymax *= d[i - 1];
  }
  s.shape_[0] = ymax;
  return s;
}
```

#### FlatTo3D

It flattens the shape into three parts: `[0, axis_begin)`, `[axis_begin, axis_end]` 
, and `(axis_end, ndim)`.

```c++
inline Shape<3> FlatTo3D(index_t axis_begin, index_t axis_end) const {
  CHECK(axis_end >= axis_begin);
  Shape<3> s;
  if (ndim_ == 0) return Shape3(0, 0, 0);
  const index_t *d = this->data();
  s.shape_[0] = 1;
  s.shape_[1] = 1;
  s.shape_[2] = 1;

  for (index_t i = 0; i < axis_begin; ++i) {
    s.shape_[0] *= d[i];
  }
  for (index_t i = axis_begin; i <= axis_end; ++i) {
    s.shape_[1] *= d[i];
  }
  for (index_t i = axis_end + 1; i < ndim_; ++i) {
    s.shape_[2] *= d[i];
  }
  return s;
}
```

It flattens the axis before and after the specified axis, so it becomes 3D tensor.

```c++
inline Shape<3> FlatTo3D(index_t axis) const {
  return FlatTo3D(axis, axis);
}
```

### ProdShape

It returns product of shape in `[dimstart,dimend)`.

```c++
inline index_t ProdShape(int dimstart, int dimend) const {
  index_t num = 1;
  const index_t *d = this->data();
  for (int i = dimstart; i < dimend; ++i) {
    num *= d[i];
  }
  return num;
}
```

### get

It returns the class `shape`, which is a component of `Tensor`.

```c++
template<int dim>
inline Shape<dim> get(void) const {
  CHECK_EQ(dim, ndim_) << "dimension do not match target dimension " << dim << " vs " << ndim_;
  const index_t *d = this->data();
  Shape<dim> s;
  for (int i = 0; i < dim; ++i) {
    s[i] = d[i];
  }
  return s;
}
```

### SetDim (`private`)

Not only does it set the value of `ndim_`, but it decides the usage of `data_stack_` or 
`data_heap_`.

```c++
inline void SetDim(index_t dim) {
  if (dim > kStackCache &&
      dim > num_heap_allocated_) {
    // data_heap_ can be NULL
    delete [] data_heap_;
    data_heap_ = new index_t[dim];
    num_heap_allocated_ = dim;
  }
  ndim_ = dim;
}
```


## TBlob

Tensor blob class (`TBlob`) that can be used to hold tensor of any dimension,
any device and any data type.
This is only a weak type that can be used to transfer data through interface.
`TBlob` itself do not involve any arithmetic operations,
but it can be converted to `Tensor` of fixed dimension for further operations.

Like `Tensor`, this data structure is like a pointer class and do not
implicit allocated, de-allocate space. 
This data structure can be helpful to hold tensors of different dimensions
and wait for further processing.

### Member Variables

There are five member variables belonging to `TBlob`.

- `dptr_`: pointer to the data
- `shape_`: shape of the tensor `TShape`
- `stride_`: storing the stride information in x dimension
- `dev_mask_`: device mask of the corresponding device
- `type_flag_`: typ flag of the tensor blob

```c++
void *dptr_;
TShape shape_;
index_t stride_;
int dev_mask_;
int type_flag_;
```

### Constructor

#### Default Constructor

Default `TBlob` is set with `dptr_` to be `NULL`, `dev_mask_` to be the mask 
of `cpu`, and `type_flag_` to be the flag of `default_real_t`, which is actually
`float` type.

```c++
TBlob(void)
    : dptr_(NULL), dev_mask_(cpu::kDevMask),
      type_flag_(DataType<default_real_t>::kFlag) {}
```

#### Constructor from `TShape`

It constructs `TBlob` from contiguous memory by setting the `stride_` to be the
highest dimension of `TShape`.

```c++
template<typename DType>
TBlob(DType *dptr,
      const TShape &shape,
      int dev_mask)
    : dptr_(dptr), shape_(shape),
      stride_(shape[shape.ndim() - 1]),
      dev_mask_(dev_mask),
      type_flag_(DataType<DType>::kFlag) {}
```

#### Constructor from `TShape` with type

It constructs `TBlob` from contiguous memory by setting the `stride_` to be the
highest dimension of `TShape`, and provides a user-defined `type_flag`.

```c++
TBlob(void *dptr,
      const TShape &shape,
      int dev_mask,
      int type_flag)
    : dptr_(dptr), shape_(shape),
      stride_(shape[shape.ndim() - 1]),
      dev_mask_(dev_mask),
      type_flag_(type_flag) {}
```

#### Constructor from `Tensor`

It uses the overloaded assignment operator `=` to construct `TBlob` from `Tensor`. The
detail of `=` can be referred to following notes.

```c++
template<typename Device, int dim, typename DType>
TBlob(const Tensor<Device, dim, DType> &src) {
  *this = src;
}
```

### Overloaded Operators

Only the assignment operator is overloaded in class `TBlob`. The `rhs` should only be 
the `Tensor` type.

```c++
template<typename Device, int dim, typename DType>
inline TBlob
&operator=(const Tensor<Device, dim, DType> &src) {
  dptr_ = src.dptr_;
  shape_ = src.shape_;
  stride_ = src.stride_;
  dev_mask_ = Device::kDevMask;
  type_flag_ = DataType<DType>::kFlag;
  return *this;
}
```

### Member Functions

#### CheckContiguous

It checks whether the `stride_` equals to the highest dimension of `shape_`.

```c++
inline bool CheckContiguous(void) const {
  return shape_[shape_.ndim() - 1] == stride_;
}
```

#### FlatTo2D

It first checks the consistency of device and type between the desired return `Tensor` and 
`TBlob` itself. Then, it calls the constructor of `Tensor`.

```c++
template<typename Device, typename DType>
inline Tensor<Device, 2, DType> FlatTo2D(Stream<Device> *stream = NULL) const {
  CHECK(Device::kDevMask == dev_mask_)
    << "TBlob.get: device type do not match specified type";
  CHECK(DataType<DType>::kFlag == type_flag_)
    << "TBlob.get_with_shape: data type do not match specified type."
    << "Expected: " << type_flag_ << " v.s. given " << DataType<DType>::kFlag;
  return Tensor<Device, 2, DType>(static_cast<DType*>(dptr_),
                                  shape_.FlatTo2D(), stride_, stream);
}
```

#### ndim

It simply calls the `ndim()` function, which is a member function of its member variable
`shape_` with type `TShape`.

```c++
inline int ndim(void) const {
  return shape_.ndim();
}
```

#### size

It returns the size of `i`-th dimension.

```c++
inline index_t size(index_t idx) const {
  return shape_[idx];
}
```

#### Size

It returns the total number of elements in `Tensor`.

```c++
inline index_t Size(void) const {
  return shape_.Size();
}
```

#### get

It fetches the tensor, with respect to specific dimension `dim`. if it do not
match the stored dimension, an error will be issued by the function `get()` of 
class `TBlob`.

```c++
template<typename Device, int dim, typename DType>
inline Tensor<Device, dim, DType> get(Stream<Device> *stream = NULL) const {
  CHECK(Device::kDevMask == dev_mask_)
    << "TBlob.get: device type do not match specified type";
  CHECK(DataType<DType>::kFlag == type_flag_)
    << "TBlob.get_with_shape: data type do not match specified type."
    << "Expected: " << type_flag_ << " v.s. given " << DataType<DType>::kFlag;
  return Tensor<Device, dim, DType>(static_cast<DType*>(dptr_),
                                     shape_.get<dim>(),
                                     stride_, stream);
}
```

#### get_with_shape

It fetches a tensor in given shape. If size do not match the stored size, 
an error will be issued. 

It first checks the consistency of device and type between the desired return `Tensor` and 
`TBlob` itself. Then, it checks whether the `TBlob` is contiguous by comparing the `stride_` 
to the highest dimension of `shape_`. Moreover, it also compares the sizes of two shapes to 
make sure the change of shape will not cause memory leak. At last, it calls the constructor 
of `Tensor` to create a new one according to the given `shape`.

```c++
template<typename Device, int dim, typename DType>
inline Tensor<Device, dim, DType> get_with_shape(const Shape<dim> &shape,
                                                 Stream<Device> *stream = NULL) const {
  CHECK(Device::kDevMask == dev_mask_)
    << "TBlob.get: device type do not match specified type";
  CHECK(DataType<DType>::kFlag == type_flag_)
    << "TBlob.get_with_shape: data type do not match specified type."
    << "Expected: " << type_flag_ << " v.s. given " << DataType<DType>::kFlag;
  CHECK_EQ(this->CheckContiguous(), true) << "TBlob.get_reshape: must be contiguous";
  CHECK_EQ(this->shape_.Size(), shape.Size())
    << "TBlob.get_with_shape: new and old shape do not match total elements";
  return Tensor<Device, dim, DType>(static_cast<DType*>(dptr_),
                                    shape,
                                    shape[dim - 1],
                                    stream);
}
```

#### FlatTo3D

It flattens the tensor to `3` dimension by calling its member function `FlatTo3D()`.
Its first version collapses the dimension before and after specified axis.

```c++
template<typename Device, typename DType>
inline Tensor<Device, 3, DType> FlatTo3D(int axis, Stream<Device> *stream = NULL) const {
  return this->get_with_shape<Device, 3, DType>(
      this->shape_.FlatTo3D(axis), stream);
}
```

Its second version collapses the dimension: `[0, axis_begin)`, `axis_begin, axis_end]`, 
and `(axis_end, ndim)`.

```c++
template<typename Device, typename DType>
inline Tensor<Device, 3, DType> FlatTo3D(int axis_begin, int axis_end,
  Stream<Device> *stream = NULL) const {
  return this->get_with_shape<Device, 3, DType>(
      this->shape_.FlatTo3D(axis_begin, axis_end), stream);
}
```


## Missing Explanation

### Algorithm: `std::fill_n`

The `std::fill_n` is defined in `<algorithm>`.

```c++
template<sclass OutputIt, class Size, class T >
OutputIt fill_n( OutputIt first, Size count, const T& value );
```

a possible implementation:

```c++
template<class OutputIt, class Size, class T>
OutputIt fill_n(OutputIt first, Size count, const T& value)
{
    for (Size i = 0; i < count; i++) {
        *first++ = value;
    }
    return first;
}
```

Assigns the given `value` to the number of `count` elements in the `first`
with the range beginning at `first` if `count>0`. Does nothing otherwise.

### Algorithm: `std::copy`

The `std::copy` is defined in `<algorithm>`.

```c++
template< class InputIt, class OutputIt >
OutputIt copy( InputIt first, InputIt last, OutputIt d_first );
```

a possible implementation:

```c++
template<class InputIt, class OutputIt>
OutputIt copy(InputIt first, InputIt last, 
              OutputIt d_first)
{
    while (first != last) {
        *d_first++ = *first++;
    }
    return d_first;
}
```

Copies all elements in the range `[first, last)` to `d_first`.

### Move Constructor

The move constructor must ensure that the moved-from
object is left in a state such that destroying that 
object will be harmless. In particular, once its 
resources are moved, the original object must no 
longer point to those moved resources—responsibility 
for those resources has been assumed by the newly 
created object.

Unlike the copy constructor, the move constructor 
does not allocate any new memory; it takes over the 
memory in the given argument. Having taken over the
memory from its argument, the constructor body sets 
the pointers in the given object to nullptr.

### IOStream: `istream::peak`

```
int peak();
```
It returns the ASCII code of next character in the input sequence, without extracting it: 
The character is left as the next character to be extracted from the stream.

### IOStream: `istream::get`

```
int get();
```

It extracts a single character from the stream.

### IOStream: `istream::setstate`

```c++
void setstate (iostate state);
```

It modifies the current internal error state flags by combining the current 
flags with those in argument state (as if performing a bitwise OR operation).

### IOStream: `std::ios::failbit`

`std::ios::failbit` represents logical error on i/o operation

Once an error has occurred, subsequent IO operations on that stream will fail.

### Condition State of `istream`

The easiest way to determine the state of a stream object is to use that 
object as a condition.

The `while` condition checks the state of the stream returned from the `>>` expression.
If that input operation succeeds, the state remains valid and the condition will
succeed. For example, 

```c++
while (is >> idx) { // `>>` input a value to `idx`
    do_some_thing();
}
```


## Missing Explanation

### Load and Save

These two functions are related to the functions in `./io.h`, and we leave their 
explanations to the related parts.

```c++
template<typename TStream>
inline void Save(TStream *strm) const {
  strm->Write(&ndim_, sizeof(ndim_));
  strm->Write(data(), sizeof(index_t) * ndim_);
}


template<typename TStream>
inline bool Load(TStream *strm) {
  if (strm->Read(&ndim_, sizeof(ndim_)) != sizeof(ndim_)) return false;
  this->SetDim(ndim_);
  size_t nread = sizeof(index_t) * ndim_;
  if (strm->Read(data(), nread) != nread) return false;
  return true;
}
```
