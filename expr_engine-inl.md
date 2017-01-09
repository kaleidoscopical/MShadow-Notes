
# Expr_engine-inl.h


`./expr_engine-inl.h` provides methods to evaluate all of the expressions.

- MakeTensorExp
- Plan
- MakePlan
- ExpInfo
- TypeCheck
- StreamInfo
- ShapeCheck
- ExpEngine

## MakeTensorExp


`: public Exp<MakeTensorExp<SubType, SrcExp, dim, DType>, DType, type::kChainer>`

`MakeTensorExp` is a base class inheritated by most functions in `./extension/`. Details will be provided
when those functions are discussed.

```c++
template<typename SubType, typename SrcExp, int dim, typename DType>
struct MakeTensorExp : public Exp<MakeTensorExp<SubType, SrcExp, dim, DType>, DType, type::kChainer> {
  Shape<dim> shape_;
  inline const SubType& real_self(void) const{
    return *static_cast<const SubType*>(this);
  }
};
```

## Plan

The class `Plan` is the class to provide `Eval()` function, which designs the way to do actual evaluation.



### Plan Declaration

The template `Plan` should include the `ExpType`, since its major purpose is to evaluate an expression, and `DType`
to guide the evaluation process.

```c++
template<typename ExpType, typename DType>
class Plan {
 public:
  MSHADOW_XINLINE DType Eval(index_t y, index_t x) const;
};
```

### Tensor Plan

The tensor plan includes two member variables `*dptr_` and `stride_` to make sure the program can fetch
expected value by `dptr_[y * stride_ + x]`.

Since we will never expect to change the evaluation result from `Eval()` function, it is return type always
include the key word `const`.

One exception is in the tensor plan, since it has chances to be the left hand side of the assignment. Thus, 
it owns a special function `REval()`, whose return type should not be `const`.

Moreover, bacause tensor value lies in a pre-allocated memory space, the return type of `REval()` function should be
a reference type. To keep consistency, the `Eval()` function in tensor plan also be made as a reference type but const.

```c++
template <typename Device, int dim, typename DType>
class Plan<Tensor<Device, dim, DType>, DType> {
 public:
  explicit Plan(const Tensor<Device, dim, DType> &t) : dptr_(t.dptr_), stride_(t.stride_) {}

  MSHADOW_XINLINE DType &REval(index_t y, index_t x) {
    return dptr_[y * stride_ + x];
  }

  MSHADOW_XINLINE const DType &Eval(index_t y, index_t x) const {
    return dptr_[y * stride_ + x];
  }

 private:
  DType  *dptr_;
  index_t stride_;
};
```

Because of the special case for 1d tensor (no stride_), the tensor plan for 1d case is specifically defined.

```c++
template <typename Device, typename DType>
class Plan<Tensor<Device, 1, DType>, DType> {
 public:
  explicit Plan(const Tensor<Device, 1, DType> &t) : dptr_(t.dptr_) {}
  MSHADOW_XINLINE DType &REval(index_t y, index_t x) {
    return dptr_[x];
  }
  MSHADOW_XINLINE const DType &Eval(index_t y, index_t x) const {
    return dptr_[x];
  }

 private:
  DType  *dptr_;
};
```

### Scalar Plan

The usage of scalar plan includes

- It has a explicit constructor that takes in a built-in scalar type value and assign the value with type `DType` to its private member variable
- It has a public member function `Eval()` that return its private member variable `scalar_`, no matter what the input is

```c++
template<typename DType>
class Plan<ScalarExp<DType>, DType> {
 public:
  explicit Plan(DType scalar) : scalar_(scalar) {}

  MSHADOW_XINLINE DType Eval(index_t y, index_t x) const {
    return scalar_;
  }

 private:
  DType scalar_;
};
```

### Typecast Plan

The usage of typecast expression includes

- it is used for the type cast expression `TypecastExp`, e.g. tcast<int>(x)
- it has an explicit constructor that takes in a `Plan` type as its private member variable. The reason of doing so 
  is it uses the original plan for evaluation, and only convert its returned type at last.
- it has a function called `Eval()`, where the type cast happens by calling `DstDType(src_.Eval(y, x))`

```c++
template<typename DstDType, typename SrcDType, typename EType, int etype>
class Plan<TypecastExp<DstDType, SrcDType, EType, etype>, DstDType> {
 public:
  explicit Plan(const Plan<EType, SrcDType> &src) : src_(src) {}

  MSHADOW_XINLINE DstDType Eval(index_t y, index_t x) const {
    return DstDType(src_.Eval(y, x));
  }

 private:
  Plan<EType, SrcDType> src_;
};
```

### Ternary Plan

The usage of ternary plan includes

- it has an explicit constructor that takes in 3 expressions as its private member variables
- it has a function called `Eval()`, where the computation happens by calling 
  `OP::Map(item1_.Eval(y, x), item2_.Eval(y, x), item3_.Eval(y, x))`;

```c++
template<typename OP, typename TA, typename TB, typename TC, int etype, typename DType>
class Plan<TernaryMapExp<OP, TA, TB, TC, DType, etype>, DType> {
 public:
  explicit Plan(const Plan<TA, DType> &item1, const Plan<TB, DType> &item2, const Plan<TC, DType> &item3)
      : item1_(item1), item2_(item2), item3_(item3) {}

  MSHADOW_XINLINE DType Eval(index_t y, index_t x) const {
    return OP::Map(item1_.Eval(y, x), item2_.Eval(y, x), item3_.Eval(y, x));
  }

 private:
  Plan<TA, DType> item1_;
  Plan<TB, DType> item2_;
  Plan<TC, DType> item3_;
};
```

### Binary Plan

The usage of binary plan includes

- it has an explicit constructor that takes in 2 expressions as its private member variables
- it has a function called `Eval()`, where the computation happens by calling 
  `OP::Map(lhs_.Eval(y, x), rhs_.Eval(y, x));`

```c++
template<typename OP, typename TA, typename TB, int etype, typename DType>
class Plan<BinaryMapExp<OP, TA, TB, DType, etype>, DType> {
 public:
  explicit Plan(const Plan<TA, DType> &lhs, const Plan<TB, DType> &rhs)
      : lhs_(lhs), rhs_(rhs) {}

  MSHADOW_XINLINE DType Eval(index_t y, index_t x) const {
    return OP::Map(lhs_.Eval(y, x), rhs_.Eval(y, x));
  }

 private:
  Plan<TA, DType> lhs_;
  Plan<TB, DType> rhs_;
};
```

### Unary Plan

The usage of unary plan includes

- it has an explicit constructor that takes in 1 expression as its private member variables
- it has a function called `Eval()`, where the computation happens by calling 
  `OP::Map(src_.Eval(y, x));`

```c++
template<typename OP, typename TA, int etype, typename DType>
class Plan<UnaryMapExp<OP, TA, DType, etype>, DType> {
 public:
  explicit Plan(const Plan<TA, DType> &src) : src_(src) {}

  MSHADOW_XINLINE DType Eval(index_t y, index_t x) const {
    return OP::Map(src_.Eval(y, x));
  }

 private:
  Plan<TA, DType> src_;
};
```

### MakeTensorExp Plan

Since the expression type `MakeTensorExp` is only a base class of several extension functions (will be discussed
in `./extensions/`), the MakeTensorExp plan also only consider its input `Plan` as its member variables, and use the 
`Eval()` function of input plan as `Eval()` function of itself.

```note
Since any expression should be composed by some types of calling, e.g. `MakeExp()`.
However, the `MakeTensorExp` lacks of it, since it is only a general base class.
Thus, I think this Plan will also never be called. Remain unproven!!!
```

```c++
template<typename SubType, typename SrcExp, int dim, typename DType>
struct Plan<MakeTensorExp<SubType, SrcExp, dim, DType>, DType> {
 public:
  Plan(const Plan<SubType, DType> &src) : src_(src) {}

  MSHADOW_XINLINE DType Eval(index_t y, index_t x) const {
    return src_.Eval(y, x);
  }

 private:
  Plan<SubType, DType> src_;
};
```


### Transpose Plan

The usage of transpose plan includes

- it has an explicit constructor that takes in a Plan `src` to initiate a same one as its private member variable.
  Since it is only a transpose expression, it still need the original `src` to do the evaluation, and only show the
  transpose effect in the special design in the `Eval()` function (the interchage of `y` and `x`).
- it has a member function called `Eval()`, which do the transpose in the computation by calling `src_.Eval(x, y)`, 
  instead of `src_.Eval(y, x)`;

```c++
template<typename EType, typename DType>
class Plan<TransposeExp<EType, DType>, DType> {
 public:
  explicit Plan(const Plan<EType, DType> &src) : src_(src) {}

  MSHADOW_XINLINE DType Eval(index_t y, index_t x) const {
    return src_.Eval(x, y);
  }

 private:
  Plan<EType, DType> src_;
};
```


## MakePlan

The major purpose of `MakePlan` is to transform an Expression to its corresponding `Plan`. 

### Tensor MakePlan

The input variable of Tensor MakePlan is `RValueExp<T, DType> &e`, which is the grandfather class of 
`Tensor`. The reason of such design choice is compatibility, since a father class can safely refer to 
its children class.

```c++
template<typename T, typename DType>
inline Plan<T, DType> MakePlan(const RValueExp<T, DType> &e) {
  return Plan<T, DType>(e.self());
}
```

### Scalar MakePlan

The Scalar MakePlan takes in a `ScalarExp`, and uses its member variable `scalar_` to initiate
`Plan<ScalarExp<DType>, DType>`.

```c++
template<typename DType>
inline Plan<ScalarExp<DType>, DType> MakePlan(const ScalarExp<DType> &e) {
  return Plan<ScalarExp<DType>, DType>(e.scalar_);
}
```

### Typecast MakePlan

Since the Typecast Plan only use the `Eval()` function of original plan to do `Eval()` of itself,
with a enforced typecast at the end, the only thing Typecast MakePlan needs to do is input a plan
of original expression

```c++
template<typename DstDType, typename SrcDType, typename EType, int etype>
inline Plan<TypecastExp<DstDType, SrcDType, EType, etype>, DstDType>
MakePlan(const TypecastExp<DstDType, SrcDType, EType, etype> &e) {
  return Plan<TypecastExp<DstDType, SrcDType, EType, etype>, DstDType>(MakePlan(e.exp));
}
```

### Ternary MakePlan

Since the Ternary Plan require three Plans to do the evaluation, the Ternary MakePlan just
provides the plan of its components.

```c++
// declaration
template<typename OP, typename TA, typename TB, typename TC, typename DType, int etype>
inline Plan<TernaryMapExp<OP, TA, TB, TC, DType, etype>, DType>
MakePlan(const TernaryMapExp<OP, TA, TB, TC, DType, etype> &e);

// definition
template<typename OP, typename TA, typename TB, typename TC, typename DType, int etype>
inline Plan<TernaryMapExp<OP, TA, TB, TC, DType, etype>, DType>
MakePlan(const TernaryMapExp<OP, TA, TB, TC, DType, etype> &e) {
  return Plan<TernaryMapExp<OP, TA, TB, TC, DType, etype>,
              DType>(MakePlan(e.item1_), MakePlan(e.item2_), MakePlan(e.item3_));
}
```

### Binary MakePlan

Since the Binary Plan require two Plans to do the evaluation, the Biary MakePlan just
provides the plan of its components.

```c++
// declaration
template<typename OP, typename TA, typename TB, typename DType, int etype>
inline Plan<BinaryMapExp<OP, TA, TB, DType, etype>, DType>
MakePlan(const BinaryMapExp<OP, TA, TB, DType, etype> &e);

// definition
template<typename OP, typename TA, typename TB, typename DType, int etype>
inline Plan<BinaryMapExp<OP, TA, TB, DType, etype>, DType>
MakePlan(const BinaryMapExp<OP, TA, TB, DType, etype> &e) {
  return Plan<BinaryMapExp<OP, TA, TB, DType, etype>,
              DType>(MakePlan(e.lhs_), MakePlan(e.rhs_));
}
```

### Unary MakePlan

Since the Unary Plan require one Plan to do the evaluation, the Unary MakePlan just
provides the plan of its component.


```c++
template<typename OP, typename TA, typename DType, int etype>
inline Plan<UnaryMapExp<OP, TA, DType, etype>, DType>
MakePlan(const UnaryMapExp<OP, TA, DType, etype> &e) {
  return Plan<UnaryMapExp<OP, TA, DType, etype>, DType>(MakePlan(e.src_));
}
```

### MakeTensorExp MakePlan

In `2.8`, we have claimed that the expression type MakeTensorExp is only a base class of several extension functions 
(will be discussed in ./extensions/), and use the `Eval()` function of input plan as `Eval()` function of itself.

Therefore, in MakeTensorExp MakePlan, the input `MakeTensorExp<T, SrcExp, dim, DType> &e` is kind of like 
`RValueExp<T, DType> &e`, since both of them are the father class of other classes. Same as Tensor MakePlan,
it returns `Plan<T, DType>(e.real_self());` as a Plan of its `SubType`.

```c++
template<typename T, typename SrcExp, int dim, typename DType>
inline Plan<T, DType>
MakePlan(const MakeTensorExp<T, SrcExp, dim, DType> &e) {
  return Plan<T, DType>(e.real_self());
}
```

### Transpose MakePlan

Since the Transpose Plan only use the `Eval()` function of original plan to do `Eval()` of itself,
with a interchage of `y` and `x`, the only thing Transpose MakePlan needs to do is input a plan
of original expression


```c++
template<typename T, typename DType>
inline Plan<TransposeExp<T, DType>, DType>
MakePlan(const TransposeExp<T, DType> &e) {
  return Plan<TransposeExp<T, DType>, DType>(MakePlan(e.exp));
}
```

## ExpInfo

`ExpInfo` is a template struct providing expression infomation to help the `TypeCheck`.

- `kDim`: is the dimension of expression related variables
- `kDevice`: is the device where the expression related variables stored

The default configuration is:

```c++
template<typename E>
struct ExpInfo {
  static const int kDim = -1;
  static const int kDevMask = 0;
};
```

For Tensor ExpInfo, information can be directly extracted from the `Tensor`:

```c++
template<typename Device, int dim, typename DType>
struct ExpInfo<Tensor<Device, dim, DType> > {
  static const int kDim = dim;
  static const int kDevMask = Device::kDevMask;
  // kDevMask = 1 for cpu (01), = 2 for gpu (10)
};
```

For Scalar ExpInfo, `kDim` is always `0`, and `kDevMask` is always `0xffff` to make sure no influence to other
variables caused by itself.

```c++
template<typename DType>
struct ExpInfo< ScalarExp<DType> > {
  static const int kDim = 0;
  static const int kDevMask = 0xffff;
};
```

For Typecast ExpInfo, the information can be directly copied from the information of original expresion

```c++
template<typename DstDType, typename SrcDType, typename EType, int etype>
struct ExpInfo<TypecastExp<DstDType, SrcDType, EType, etype> > {
  static const int kDim = ExpInfo<EType>::kDim;
  static const int kDevMask = ExpInfo<EType>::kDevMask;
};
```

For Ternary ExpInfo, its information is from three different expressions:

```c++
template<typename OP, typename TA, typename TB, typename TC, typename DType, int etype>
struct ExpInfo<TernaryMapExp<OP, TA, TB, TC, DType, etype> > {
  static const int kDimItem1 = ExpInfo<TA>::kDim;
  static const int kDimItem2 = ExpInfo<TB>::kDim;
  static const int kDimItem3 = ExpInfo<TC>::kDim;
  static const int kDim = kDimItem1;
  static const int kDevMask = ExpInfo<TA>::kDevMask & ExpInfo<TB>::kDevMask & ExpInfo<TC>::kDevMask;
};
```

For Binary ExpInfo, its information is from two different expressions:

```c++
template<typename OP, typename TA, typename TB, typename DType, int etype>
struct ExpInfo<BinaryMapExp<OP, TA, TB, DType, etype> > {
  static const int kDimLhs = ExpInfo<TA>::kDim;
  static const int kDimRhs = ExpInfo<TB>::kDim;
  static const int kDim = (kDimLhs >= 0 && kDimRhs >= 0) ?\
      (kDimLhs == 0 ?\
       kDimRhs :\
       ((kDimRhs == 0 || kDimLhs == kDimRhs) ? kDimLhs : -1)) : -1;
  static const int kDevMask = ExpInfo<TA>::kDevMask & ExpInfo<TB>::kDevMask;    
};
```

For Unary ExpInfo and Transpose ExpInfo, its information is also simply the copy of its variable:

```c++
template<typename OP, typename TA, typename DType, int etype>
struct ExpInfo<UnaryMapExp<OP, TA, DType, etype> > {
  static const int kDim = ExpInfo<TA>::kDim;
  static const int kDevMask = ExpInfo<TA>::kDevMask;
};

template<typename E, typename DType>
struct ExpInfo<TransposeExp<E, DType> > {
  static const int kDim = ExpInfo<E>::kDim;
  static const int kDevMask = ExpInfo<E>::kDevMask;
};
```

At last, the MakeTensorExp Info is determined by `SrcExp` as input expression variable of its `SubType`

```note
the reason of its design is unclear for now, remain to be checking !!!
```

```c++
template<typename T, typename SrcExp, int dim, typename DType>
struct ExpInfo<MakeTensorExp<T, SrcExp, dim, DType> > {
  static const int kDimSrc = ExpInfo<SrcExp>::kDim;
  static const int kDim = kDimSrc >= 0 ? dim : -1;
  static const int kDevMask = ExpInfo<SrcExp>::kDevMask;
};
```

## TypeCheck

`TypeCheck` is a template to do type checking. The checking happens in compile time.

- `kExpDim`: is the dimension of expression.
- `kDevPass`: checks whether the expression device type matches the provided type. 
  This is to make sure that the expression related variables are all in one specified device.
- `kMapPass`: checks whether the expression can be mapped to expression of dim
- `kRedPass`: checks whether the expression can be reduced to expression of dim

```warning
The usage of kRedPass is unclear yet!!!
```

```c++
template<typename Device, int dim, typename DType, typename E>
struct TypeCheck {
  static const int kExpDim = ExpInfo<E>::kDim; 
  static const bool kDevPass = (ExpInfo<E>::kDevMask & Device::kDevMask) != 0;
  static const bool kMapPass = (kExpDim == 0 || kExpDim == dim) && kDevPass; 
  static const bool kRedPass = (kExpDim > dim) && kDevPass;
};
```

The usage of TypeCheck is like 
`expr::TypeCheckPass<expr::TypeCheck<cpu, dim, DType, E>::kMapPass> ::Error_All_Tensor_in_Exp_Must_Have_Same_Type();`.
If the `TypeCheck<cpu, dim, DType, E>::kMapPass>` returns `false`, then the corresponding function `Error_All_Tensor_in_Exp_Must_Have_Same_Type()`
will not be found, which leads to the fail of compile.


```c++
template<bool kPass>
struct TypeCheckPass;
template<>
struct TypeCheckPass<false> {};
template<>
struct TypeCheckPass<true> {
  inline static void Error_All_Tensor_in_Exp_Must_Have_Same_Type(void) {}
  inline static void Error_TypeCheck_Not_Pass_For_Reduce_Exp(void) {}
  inline static void Error_Expression_Does_Not_Meet_Dimension_Req(void) {}
};
```

## StreamInfo

The `StreamInfo` returns the stream of a `Tensor` in its stored `Device`.

```warning
The usage of it is unclear yet !!!
```

```c++
template<typename Device, typename E>
struct StreamInfo {
  inline static Stream<Device> *Get(const E &t);
};

template<int dim, typename Device, typename DType>
struct StreamInfo<Device, Tensor<Device, dim, DType> > {
  inline static Stream<Device> *Get(const Tensor<Device, dim, DType> &t) {
    return t.stream_;
  }
};
```

## ShapeCheck

`ShapeCheck` is a runtime shape checking template to get the shape of an expression, 
reporting error if shape mismatch

### Declaration

```c++
template<int dim, typename E>
struct ShapeCheck {
  inline static Shape<dim> Check(const E &t);
};
```

### Tensor ShapeCheck

The Tensor ShapeCheck simply returns its shape.

```c++
template<int dim, typename Device, typename DType>
struct ShapeCheck<dim, Tensor<Device, dim, DType> > {
  inline static Shape<dim> Check(const Tensor<Device, dim, DType> &t) {
    return t.shape_;
  }
};
```

### Scalar ShapeCheck

The Scalar ShapeCheck returns a same `Shape` of provided `dim`, but all of the dimension equal to `0`.

```c++
template<int dim, typename DType>
struct ShapeCheck<dim, ScalarExp<DType> > {
  inline static Shape<dim> Check(const ScalarExp<DType> &exp) {
    Shape<dim> shape;
    for (int i = 0; i < dim; ++i) {
      shape[i] = 0;
    }
    return shape;
  }
};
```

### Typecast ShapeCheck

The Typecast ShapeCheck simply conducts in a way that calls the original `Check()` of the expression 
wishing to cast it type.

```c++
template<int dim, typename DstDType, typename SrcDType, typename EType, int etype>
struct ShapeCheck<dim, TypecastExp<DstDType, SrcDType, EType, etype> > {
  inline static Shape<dim>
  Check(const TypecastExp<DstDType, SrcDType, EType, etype> &exp) {
    return ShapeCheck<dim, EType>::Check(exp.exp);
  }
};
```

### Ternary ShapeCheck

The Ternary ShapeCheck first fetch the check results of its three variable expressions.
Then, it couducts a explicit checking to make sure all of the expressions have same shape.
At last, it returns one of the shape of expressions.

```c++
template<int dim, typename OP, typename TA, typename TB, typename TC, typename DType, int etype>
struct ShapeCheck<dim, TernaryMapExp<OP, TA, TB, TC, DType, etype> > {
  inline static Shape<dim>
  Check(const TernaryMapExp<OP, TA, TB, TC, DType, etype> &t) {
    Shape<dim> shape1 = ShapeCheck<dim, TA>::Check(t.item1_);
    Shape<dim> shape2 = ShapeCheck<dim, TB>::Check(t.item2_);
    Shape<dim> shape3 = ShapeCheck<dim, TC>::Check(t.item3_);
    bool same = (shape1 == shape2) && (shape2 == shape3);
    CHECK(same) << "TernaryMapExp: Shapes of operands are not the same, " <<
      "Shape1=" << shape1 << ", Shape2=" << shape2 << ", Shape3=" << shape3;
    return shape1;
  }
};
```

### Binary ShapeCheck

The Binary ShapeCheck first fetch the check results of its two variable expressions.
Then, it couducts a explicit checking to make sure all of the expressions have same shape.
At last, it returns one of the shape of expressions.

```c++
template<int dim, typename OP, typename TA, typename TB, typename DType, int etype>
struct ShapeCheck<dim, BinaryMapExp<OP, TA, TB, DType, etype> > {
  inline static Shape<dim>
  Check(const BinaryMapExp<OP, TA, TB, DType, etype> &t) {
    Shape<dim> shape1 = ShapeCheck<dim, TA>::Check(t.lhs_);
    Shape<dim> shape2 = ShapeCheck<dim, TB>::Check(t.rhs_);
    if (shape1[0] == 0) return shape2;
    if (shape2[0] == 0) return shape1;
    CHECK_EQ(shape1, shape2) << "BinaryMapExp: Shapes of operands are not the same, " <<
      "Shape1=" << shape1 << ", Shape2=" << shape2;
    return shape1;
  }
};
```

### Unary ShapeCheck

For Unary ShapeCheck, it just fetch the check result of its variable expressions and return it.

```c++
template<int dim, typename OP, typename TA, typename DType, int etype>
struct ShapeCheck<dim, UnaryMapExp<OP, TA, DType, etype> > {
  inline static Shape<dim> Check(const UnaryMapExp<OP, TA, DType, etype> &t) {
    Shape<dim> s = ShapeCheck<dim, TA>::Check(t.src_);
    return s;
  }
};
```

### MakeTensorExp ShapeCheck

MakeTensorExp ShapeCheck simply use the member variable `shape_` of `MakeTensorExp` as its return.

```warning
It is design strategy needs to be further examined.
```

```c++
template<int dim, typename SrcExp, typename T, typename DType>
struct ShapeCheck<dim, MakeTensorExp<T, SrcExp, dim, DType> > {
  inline static Shape<dim>
  Check(const MakeTensorExp<T, SrcExp, dim, DType> &t) {
    return t.shape_;
  }
};
```

### Transpose ShapeCheck

Transpose ShapeCheck fetchs the checked shape of expression to be transposed, and simply return the
swaped lowest two dimensions.

```warning
The strategy of Transpose needs to be further studied when it comes to application.
```

```c++
template<int dim, typename E, typename DType>
struct ShapeCheck<dim, TransposeExp<E, DType> > {
  inline static Shape<dim> Check(const TransposeExp<E, DType> &e) {
    // swap the lowest two dimensions
    Shape<dim> s = ShapeCheck<dim, E>::Check(e.exp);
    std::swap(s[0], s[1]);
    return s;
  }
};
```

## ExpEngine

`ExpEngine` is a struct with several overloaded `Eval()` functions, always called by the assignment
related functions in `RValueExp` to dispatch simple operations.

```c++
template<typename SV, typename RV, typename DType>
struct ExpEngine {
  template<typename E>
  inline static void Eval(RV *dst,
                          const Exp<E, DType, type::kMapper> &exp) {
    MapExp<SV>(dst, exp);
  }

  template<typename E>
  inline static void Eval(RV *dst,
                          const Exp<E, DType, type::kChainer> &exp) {
    MapExp<SV>(dst, exp);
  }

  template<typename E>
  inline static void Eval(RV *dst,
                          const Exp<E, DType, type::kRValue> &exp) {
    MapExp<SV>(dst, exp);
  }

  template<typename E>
  inline static void Eval(RV *dst,
                          const Exp<E, DType, type::kComplex> &exp) {
    ExpComplexEngine<SV, RV, E, DType>::Eval(dst->ptrself(), exp.self());
  }
};
```

The `ExpComplexEngine` to evaluate expression with `type::kComplex` majorly deals with
`dot()` expression and looks like:

```c++
template<typename SV, typename RV, typename E, typename DType>
struct ExpComplexEngine {
  inline static void Eval(RV *dst, const E &exp);
};

template<typename SV, typename Device, int dim, int ldim,
         int rdim, bool ltrans, bool rtrans, typename DType>
struct ExpComplexEngine<SV,
                        Tensor<Device, dim, DType>,
                        DotExp<Tensor<Device, ldim, DType>,
                               Tensor<Device, rdim, DType>,
                               ltrans, rtrans, DType>,
                        DType> {
  inline static void Eval(Tensor<Device, dim, DType> *dst,
                          const DotExp<Tensor<Device, ldim, DType>,
                                       Tensor<Device, rdim, DType>,
                                       ltrans, rtrans, DType> &exp) {
    DotEngine<SV, Device, dim, ldim, rdim,
              ltrans, rtrans, DType>::Eval(dst, exp.lhs_, exp.rhs_, exp.scale_);
  }
};
```

The DotEngine will be further discussed in `./dot_engine-inl.h`.










































