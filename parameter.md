# /dmlc-core/parameter.h

## Overview

[TOC]

`parameter.h` provides lightweight utilities to do parameter setup and checking.

- Predefined enum and struct 
- FieldAccessEntry 
- FieldEntryBase
- FieldEntryNumeric
- FieldEntry
- ParamManager
- ParamManagerSingleton
- Parameter
- MACRO


## 1. Predefined enum and struct 

### 1.1 ParamInitOption

`ParamInitOption` is the option in parameter initialization.

- `kAllowUnknown`: allow unknown parameters
- `kAllMatch`: need to match exact parameters
- `kAllowHidde`: allow unmatched hidden field with format `__*__`

```c++
enum ParamInitOption {
  kAllowUnknown,
  kAllMatch,
  kAllowHidde
};
```

### 1.2 ParamFieldInfo

`ParamFieldInfo` provides information about a parameter field in string representations.

- `name`: name of the field
- `type`: type of the field in string format
- `type_info_str`: This includes the default value, enum constrain and typename
- `description`: detailed description of the type

```c++
struct ParamFieldInfo {
  std::string name;
  std::string type;
  std::string type_info_str;
  std::string description;
};
```

## 2. FieldAccessEntry

`FieldAccessEntry` is an internal interface to help managge the parameters.
Each entry can be used to access one parameter in the `struct` inherited from `Parameter`.

Following is an overview of its member variables and functions, which are all defined as virtual
function 

```c++
class FieldAccessEntry {
 public:
  FieldAccessEntry()
      : has_default_(false) {}
  virtual ~FieldAccessEntry() {}
  virtual void SetDefault(void *head) const = 0;
  virtual void Set(void *head, const std::string &value) const = 0;
  virtual void Check(void *head) const {}
  virtual std::string GetStringValue(void *head) const = 0;
  virtual ParamFieldInfo GetFieldInfo() const = 0;

 protected:
  bool has_default_;
  size_t index_;
  std::string key_;
  std::string type_;
  std::string description_;
  virtual void PrintDefaultValueString(std::ostream &os) const = 0; 
  friend class ParamManager;
};
```

We add some brief introduction of the concepts in above code.

### 2.1 Pure Virtual Function

Pure virtual function is a special type of virtual function by adding `= 0` in the back.

A class containing one or more pure virtual functions is called `abstract class`. We cannot
declare instances from this class. It only serve as a `base class` for its `derived class`.
Moreover, unless all pure virtual functions are defined in the `derived class`, we can still
not declare instances using the `derived class`.

### 2.2 Virtual Destructor

When the `base class` has virtual function, its `destructor` must be 
defined to be `virtual`. If not, when deconstruct its child classes,
the system will call the destructor of `base class`, which will lead
to the memory leak, since the variables owned by child classes
will not be deleted.


## 3. FieldEntryBase
```c++
template<typename TEntry, typename DType>
class FieldEntryBase : public FieldAccessEntry
```

`FieldEntryBase` is a base class of all `FieldEntry` class, but itself is inherited from 
`FieldAccessEntry` with all pure virtual function defined.

### 3.1 Member Variables

```c++
ptrdiff_t offset_;
DType default_value_;

// together with inherited ones
bool has_default_;
size_t index_;
std::string key_;
std::string type_;
std::string description_;
```

`ptrdiff_t`, like `size_t` (`unsigned`), is a machine-related type (`signed`), defined in `cstddef` head file.

In the context of `dmlc-core`, we use it to identify the offset of the address between the `head` and
the `target instance`.


### 3.2 Member Functions

#### 3.2.1 `public:` Init

`Init()` function sets values for member variables `key_`, `type_`, and `offset_`.

```c++
inline void Init(const std::string &key,
                 void *head, DType &ref) { 
  this->key_ = key;
  if (this->type_.length() == 0) {
    this->type_ = dmlc::type_name<DType>();
  }
  this->offset_ = ((char*)&ref) - ((char*)head); 
}
```

The code `this->type_.length() == 0` means the `type_` is only set once and cannot change.

Moreover, the function `dmlc::type_name<DType>()` is defined in `./type_traits.h`.

```c++
#define DMLC_DECLARE_TYPE_NAME(Type, Name)            \
  template<>                                          \
  inline const char* type_name<Type>() {              \
    return Name;                                      \
  }

DMLC_DECLARE_TYPE_NAME(float, "float");
DMLC_DECLARE_TYPE_NAME(double, "double");
DMLC_DECLARE_TYPE_NAME(int, "int");
DMLC_DECLARE_TYPE_NAME(uint32_t, "int (non-negative)");
DMLC_DECLARE_TYPE_NAME(uint64_t, "long (non-negative)");
DMLC_DECLARE_TYPE_NAME(std::string, "string");
DMLC_DECLARE_TYPE_NAME(bool, "boolean");
```

#### 3.2.2 `protected:` Get

`Get()` function fetches the internal representation of parameters.

For example, if this entry corresponds field `param.learning_rate`
then `Get(&param)` will return reference to `param.learning_rate`.
Its realization thanks to the `offset_` between `head` and the target
instance.

```c++
inline DType &Get(void *head) const {
  return *(DType*)((char*)(head) + offset_);
}
```

First, it converts `head` wity type `void*` to `char*`.
Second it adds the `offset_` to `head`.
Third, it converts `head` added by `offset_` to `DType*`.
Last, it returns the value of `head` with type `DType*`.

Note, `protected` variables or functions can be accessed by `friend` or `hild class`.

#### 3.2.3 `public:` Set

`Set()` function set the target instance with `value`. In its implementation, 
it explicitly avoids `space` in the value.

Important Note: the value returned by `Get()` is `DType`. Thanks to the overloaded operator
`>>` in `istringstream`, we can convert the `string` type of `is` to the corresponding `DType`.

```c++
istream& operator>> (bool& val);
istream& operator>> (short& val);
istream& operator>> (unsigned short& val);
istream& operator>> (int& val);
istream& operator>> (unsigned int& val);
istream& operator>> (long& val);
istream& operator>> (unsigned long& val);
istream& operator>> (float& val);
istream& operator>> (double& val);
istream& operator>> (long double& val);
istream& operator>> (void*& val);
```

```c++
virtual void Set(void *head, const std::string &value) const {
  std::istringstream is(value);
  is >> this->Get(head);
  // the following codes is designed for avoiding `space` in the value
  if (!is.fail()) {
    while (!is.eof()) {
      int ch = is.get(); // `.get()` returns the next character of istringstream
      if (ch == EOF) {
        is.clear(); break;  // `.clear()` clears all error flags in default
      }
      if (!isspace(ch)) {   // `isspace()` is a C function to check space
        is.setstate(std::ios::failbit); break;  // if we run into space, we manually set the failbit 
                                                // so the program will go into the next `if()`
      }
    }
  }

  if (is.fail()) {
    std::ostringstream os;
    os << "Invalid Parameter format for " << key_
       << " expect " << type_ << " but value=\'" << value<< '\'';
    throw dmlc::ParamError(os.str());
  }
}
```

The declaration of explicit constructor of `istringstream` is 

```c++
explicit istringstream (const string& str, ios_base::openmode which = ios_base::in);
// str: A string object, whose content is copied.
```

#### 3.2.4 `public:` set_default

In default, class inherited from `FieldEntryBase` do not have a default value.
`set_default()` function changes it by setting `has_default_` to be `true` and 
sets `default_value_` defined by itself to the value of input variable. At last,
it returns itself to allow chaining.

```c++
inline TEntry &set_default(const DType &default_value) {
  default_value_ = default_value;
  has_default_ = true;
  return this->self();
}
```

#### 3.2.5 `public:` self

`self()` first change the type of pointer `this` to the pointer type of its sub-class.
Then, it returns the pointed instance of the pointer.

```c++
inline TEntry &self() {
  return *(static_cast<TEntry*>(this));
}
```

Note: a `static_cast` is useful to perform a conversion that the compiler will not
generate automatically.

#### 3.2.6 `public:` SetDefault

Since the function `set_default()` changes the flag `has_default_` to be `true`, we 
are able to modify the target instance with the pre-set `default_value_`. If it is 
called before `set_default()`, it will throw out a error.

```c++
virtual void SetDefault(void *head) const {
  if (!has_default_) {
    std::ostringstream os;
    os << "Required parameter " << key_
       << " of " << type_ << " is not presented";
    throw dmlc::ParamError(os.str());
  } else {
    this->Get(head) = default_value_;
  }
}
```

#### 3.2.7 `public:` describe

It simply sets the self-defined member variables `description_` to the input.

```c++
inline TEntry &describe(const std::string &description) {
  description_ = description;
  return this->self();
}
```

#### 3.2.8 `protected:` PrintValue 

`PrintValue()` simply prints the value with `DType`.

```c++
virtual void PrintValue(std::ostream &os, DType value) const {
  os << value;
}
```

#### 3.2.9 `protected:` PrintDefaultValueString

`PrintDefaultValueString()` simply prints the default value.

```c++
virtual void PrintDefaultValueString(std::ostream &os) const { 
  PrintValue(os, default_value_);
}
```

#### 3.2.10 `public:` GetStringValue

`GetStringValue()` simply prints the value it stores.

```c++
virtual std::string GetStringValue(void *head) const {
  std::ostringstream os;
  PrintValue(os, this->Get(head));
  return os.str();
}
```

#### 3.2.11 `public:` GetFieldInfo

It simply sets information of the class into the four member variables in `ParamFieldInfo`,
and returns it.

```c++
virtual ParamFieldInfo GetFieldInfo() const {
  ParamFieldInfo info;
  std::ostringstream os;
  info.name = key_;
  info.type = type_;
  os << type_;
  if (has_default_) {
    os << ',' << " optional, default=";
    PrintDefaultValueString(os);
  } else {
    os << ", required";
  }
  info.type_info_str = os.str();
  info.description = description_;
  return info;
}
```

## 4. FieldEntryNumeric

`FieldEntryNumeric` is a base class for numeric types that have ranges. It is the 
child class of `FieldEntryBase` and grandchild class of `FieldAccessEntry`.

```c++
template<typename TEntry, typename DType>
class FieldEntryNumeric : public FieldEntryBase<TEntry, DType> {};
```

### 4.1 Member Variables

```c++
bool has_begin_, has_end_;
DType begin_, end_;

// inherited from `FieldEntryBase`
ptrdiff_t offset_;
DType default_value_;

// inherited from `FieldAccessEntry`
bool has_default_;
size_t index_;
std::string key_;
std::string type_;
std::string description_;
```

### 4.2 Constructor 

In default, a `FieldEntryNumeric` type do not have range constraints.

```c++
FieldEntryNumeric()
    : has_begin_(false), has_end_(false) {}
```

### 4.3 Member Functions

#### 4.3.1 set_range

It simply sets the range of `begin` and `end`, and turns on the flag
`has_begin_` and `has_end_`.

```c++
virtual TEntry &set_range(DType begin, DType end) {
  begin_ = begin; end_ = end;
  has_begin_ = true; has_end_ = true;
  return this->self();
}
```

#### 4.3.2 set_lower_bound

It simply sets the `begin_` and turns on the flag `has_begin_`.

```c++
virtual TEntry &set_lower_bound(DType begin) {
  begin_ = begin; has_begin_ = true;
  return this->self();
}
```

#### 4.3.3 Check

It does a simple checking of parameter constraints.

```c++
virtual void Check(void *head) const {
  FieldEntryBase<TEntry, DType>::Check(head); // not implement
  DType v = this->Get(head);
  if (has_begin_ && has_end_) {
    if (v < begin_ || v > end_) {
      std::ostringstream os;
      os << "value " << v << "for Parameter " << this->key_
         << " exceed bound [" << begin_ << ',' << end_ <<']';
      throw dmlc::ParamError(os.str());
    }
  } else if (has_begin_ && v < begin_) {
      std::ostringstream os;
      os << "value " << v << "for Parameter " << this->key_
         << " should be greater equal to " << begin_;
      throw dmlc::ParamError(os.str());
  } else if (has_end_ && v > end_) {
      std::ostringstream os;
      os << "value " << v << "for Parameter " << this->key_
         << " should be smaller equal to " << end_;
      throw dmlc::ParamError(os.str());
  }
}
```

## 5. FieldEntry

`FieldEntry` defines parsing and checking behavior of `DType`.
This class can be specialized to implement specific behavior of more settings.

### 5.1 FieldEntry`<typename DType>`

It is a general definition of FieldEntry, which only determines it inheriting from
a common `FieldEntryBase` class or a specific `FieldEntryNumeric` class for numeric.

```c++
template<typename DType>
class FieldEntry :
      public IfThenElseType<dmlc::is_arithmetic<DType>::value,
                            FieldEntryNumeric<FieldEntry<DType>, DType>,
                            FieldEntryBase<FieldEntry<DType>, DType> >::Type {
};
```

Basically, it uses a trick to let `FieldEntry` inherit from different classes.
Following codes will be helpful for understanding.

```c++
// all defined in `./type_traits.h`
template<typename T>
struct is_arithmetic {
#if DMLC_USE_CXX11
  /*! \brief the value of the traits */
  static const bool value = std::is_arithmetic<T>::value;
#else
  /*! \brief the value of the traits */
  static const bool value = (dmlc::is_integral<T>::value ||
                             dmlc::is_floating_point<T>::value);
#endif
};

template<typename Then, typename Else>
struct IfThenElseType<true, Then, Else> {
  typedef Then Type;
};

template<typename Then, typename Else>
struct IfThenElseType<false, Then, Else> {
  typedef Else Type;
};
```

### 5.2 FieldEntry`<int>`

```c++
template<>
class FieldEntry<int> : public FieldEntryNumeric<FieldEntry<int>, int> {};
```

It specializes the definition of FieldEntry for `int` type to implement some
specific behaviors.

#### 5.2.1 Member Variables

```c++
bool is_enum_;
std::map<std::string, int> enum_map_;
std::map<int, std::string> enum_back_map_;

// inherited from `FieldEntryNumeric`
bool has_begin_, has_end_;
DType begin_, end_;

// inherited from `FieldEntryBase`
ptrdiff_t offset_;
DType default_value_;

// inherited from `FieldAccessEntry`
bool has_default_;
size_t index_;
std::string key_;
std::string type_;
std::string description_;
```

#### 5.2.2 Constructor

In default, we cannot do enumeration.

```c++
FieldEntry<int>() : is_enum_(false) {}
```

#### 5.2.3 Member Functions

##### 5.2.3.1 Set

It adds the consideration of enumeration case. If `is_enum_` is `true`,
it will first find the corresponding instance of `value` and return its
iterator. Then, it will determine whether to set the value according to 
the return. All in all, the actual set is still performed by its father
class. 

```c++
virtual void Set(void *head, const std::string &value) const {
  if (is_enum_) {
    std::map<std::string, int>::const_iterator it = enum_map_.find(value);
    std::ostringstream os;
    if (it == enum_map_.end()) {
      os << "Invalid Input: \'" << value;
      os << "\', valid values are: ";
      PrintEnums(os);
      throw dmlc::ParamError(os.str());
    } else {
      os << it->second;
      Parent::Set(head, os.str());
    }
  } else {
    Parent::Set(head, value);
  }
}
```

##### 5.2.3.2 add_enum

It only works when the `enum_map_.size()` is `0` and the `key` in `enum_map_` has not 
been set, and `value` in `enum_back_map_` also has not been set.

Its workflow is set the corresponding relation between `key` and `value` directly and 
reversely. Then, it will turn on the flag `is_enum_` and return itself for chaining.

```c++
inline FieldEntry<int> &add_enum(const std::string &key, int value) {
  if ((enum_map_.size() != 0 && enum_map_.count(key) != 0) || \
      enum_back_map_.count(value) != 0) {
    std::ostringstream os;
    os << "Enum " << "(" << key << ": " << value << " exisit!" << ")\n";
    os << "Enums: ";
    for (std::map<std::string, int>::const_iterator it = enum_map_.begin();
         it != enum_map_.end(); ++it) {
      os << "(" << it->first << ": " << it->second << "), ";
    }
    throw dmlc::ParamError(os.str());
  }
  enum_map_[key] = value;
  enum_back_map_[value] = key;
  is_enum_ = true;
  return this->self();
}
```

```warning
why not use auto here?
may not get the const_iterator?
why not use cbegin()
```

##### 5.2.3.3 PrintValue

`PrintValue()` directly print the input value of type `int` if the flag `is_enum_` is not on.
If the flag is on, it first check whether the value is in the `enum_back_map_`. If it is, it 
returns the value corresponded to `value` in `enum_back_map_`.

```c++
virtual void PrintValue(std::ostream &os, int value) const { 
  if (is_enum_) {
    CHECK_NE(enum_back_map_.count(value), 0U)
        << "Value not found in enum declared";
    os << enum_back_map_.at(value);
  } else {
    os << value;
  }
}
```

##### 5.2.3.4 PrintDefaultValueString

It simply call the function `PrintValue()` to print its `default_value_`.

```c++
virtual void PrintDefaultValueString(std::ostream &os) const { 
  os << '\'';
  PrintValue(os, default_value_);
  os << '\'';
}
```

##### 5.2.3.5 PrintEnums

It simply output every `string` type, which is the key in the `enum_map_`.

```c++
inline void PrintEnums(std::ostream &os) const { 
  os << '{';
  for (std::map<std::string, int>::const_iterator
           it = enum_map_.begin(); it != enum_map_.end(); ++it) {
    if (it != enum_map_.begin()) {
      os << ", ";
    }
    os << "\'" << it->first << '\'';
  }
  os << '}';
}
```

##### 5.2.3.6 GetFieldInfo

If `is_enum_` is not on, it simply call `GetFieldInfo()` of its parent class.
Otherwise, it perform its own codes. The only difference is the `PrintEnums(os)`.

```c++
virtual ParamFieldInfo GetFieldInfo() const {
  if (is_enum_) {
    ParamFieldInfo info;
    std::ostringstream os;
    info.name = key_;
    info.type = type_;
    PrintEnums(os);
    if (has_default_) {
      os << ',' << "optional, default=";
      PrintDefaultValueString(os);
    } else {
      os << ", required";
    }
    info.type_info_str = os.str();
    info.description = description_;
    return info;
  } else {
    return Parent::GetFieldInfo();
  }
}
```

### 5.3 FieldEntry`<std::string>`

It specializes the definition of `FieldEntry` for `std::string` type to implement some specific behaviors.

```c++
template<>
class FieldEntry<std::string> 
    : public FieldEntryBase<FieldEntry<std::string>, std::string> {};
```

#### 5.3.1 Member Functions

##### 5.3.1.1 Set

Since the `Set()` function in its base class `FieldEntryBase` has a design for avoiding `space`
in the value, it is naturally insuitable for a specific `std::string` type. So it overloads the 
`Set()` by directly set the target instance as the input `value`.

```c++
virtual void Set(void *head, const std::string &value) const {
  this->Get(head) = value;
}
```

##### 5.3.1.2 PrintDefaultValueString

It overloads the `PrintDefaultValueString()` to add single quotes on its two sides.

```c++
virtual void PrintDefaultValueString(std::ostream &os) const {  // NOLINT(*)
  os << '\'' << default_value_ << '\'';
}
```

### 5.4 FieldEntry`<bool>`

It specializes the definition of `FieldEntry` for `bool` type to implement some specific behaviors.

```c++
template<>
class FieldEntry<bool>
    : public FieldEntryBase<FieldEntry<bool>, bool> {};
```

#### 5.4.1 Member Functions

##### 5.4.1.1 Set

It first define a `string` with same size of `value`. Then it transforms the characters 
in `value` to be lowercase and stores in the defined string. If the value is one of 
`true`, `false`, `1`, and `0`, it will set the target instance as `true` of `false`
accordingly. Otherwise, it will raise an error.

```c++
virtual void Set(void *head, const std::string &value) const {
  std::string lower_case; lower_case.resize(value.length());
  std::transform(value.begin(), value.end(), lower_case.begin(), ::tolower);
  bool &ref = this->Get(head);  // directly set the value by reference
  if (lower_case == "true") {
    ref = true;
  } else if (lower_case == "false") {
    ref = false;
  } else if (lower_case == "1") {
    ref = true;
  } else if (lower_case == "0") {
    ref = false;
  } else {
    std::ostringstream os;
    os << "Invalid Parameter format for " << key_
       << " expect " << type_ << " but value=\'" << value<< '\'';
    throw dmlc::ParamError(os.str());
  }
}
```

##### 5.4.1.2 PrintValue

Instead of directly output `true` and `false` as `1` and `0`, it outputs them as 
a `string`.

```c++
virtual void PrintValue(std::ostream &os, bool value) const {
  if (value) {
    os << "True";
  } else {
    os << "False";
  }
}
```

## 6. ParamManager

`class ParamManager {};`

`ParamManager` class is to handle parameter settings for each type in the structure.
An `ParamManager` will be created for each parameter structures.

### 6.1 Member Variables

- `name_`: parameter holding struct name
- `entry_`: positional list of entries
- `entry_map_`: map from key to entry

```c++
std::string name_;
std::vector<FieldAccessEntry*> entry_;
std::map<std::string, FieldAccessEntry*> entry_map_;
```

### 6.2 Destructor

`ParamManager` explicitly deletes every `entry_` to trigger their destructors.

```c++
~ParamManager() {
  for (size_t i = 0; i < entry_.size(); ++i) {
    delete entry_[i];
  }
}
```

### 6.3 Member Functions

#### 6.3.1 AddEntry

`AddEntry()` is an internal function to add entry to manager, and the manager will take over 
the ownership of the entry.

It first sets the member variable `index_` in `FieldAccessEntry` to current `entry_.size()`.
So, every `FieldEntry` can be sorted according to the order of add. Second, if the `key` is
already in the `entry_map_`, it will raise an error. Otherwise, it will push the object
of type `FieldAccessEntry` into its `entry_`, and set the map connection between `key` and
the object instance.

```c++
inline void AddEntry(const std::string &key, FieldAccessEntry *e) {
  e->index_ = entry_.size();
  if (entry_map_.count(key) != 0) {
    LOG(FATAL) << "key " << key << " has already been registered in " << name_;
  }
  entry_.push_back(e);
  entry_map_[key] = e;
}
```

#### 6.3.2 AddAlias

`AddAlias()` is to set an alias to existent entry. It first checks whether the `field` name
has already registered, and `alias` has not been registered. If both is true, it create a 
new map connection between `alias` and the object instance.

```c++
inline void AddAlias(const std::string& field, const std::string& alias) {
  if (entry_map_.count(field) == 0) {
    LOG(FATAL) << "key " << field << " has not been registered in " << name_;
  }
  if (entry_map_.count(alias) != 0) {
    LOG(FATAL) << "Alias " << alias << " has already been registered in " << name_;
  }
  entry_map_[alias] = entry_map_[field];
}
```

#### 6.3.3 set_name

`set_name()` simply set its member variable `name_` to be the name of `struct`.

```c++
inline void set_name(const std::string &name) {
  name_ = name;
}
```

#### 6.3.4 GetFieldInfo

`GetFieldInfo()` simply builds a `vector` to store the field information of every
entry respectively and returns it.

```c++
inline std::vector<ParamFieldInfo> GetFieldInfo() const {
  std::vector<ParamFieldInfo> ret(entry_.size());
  for (size_t i = 0; i < entry_.size(); ++i) {
    ret[i] = entry_[i]->GetFieldInfo();
  }
  return ret;
}
```

#### 6.3.5 PrintDocString

`PrintDocString()` is designed for readible docstring.

For every component in `entry_`, it first fetches its corresponding `ParamFieldInfo`, and
output its member variable `name` and `type_info_str`. If it has `description`, that will
also be output.

```c++
inline void PrintDocString(std::ostream &os) const {
  for (size_t i = 0; i < entry_.size(); ++i) {
    ParamFieldInfo info = entry_[i]->GetFieldInfo();
    os << info.name << " : " << info.type_info_str << '\n';
    if (info.description.length() != 0) {
      os << "    " << info.description << '\n';
    }
  }
}
```

#### 6.3.6 GetDict

It gets the internal parameters and push them into a vector of pairs. The `first` value in 
the pair is the name of the instance, and the `second` value is the content of the internal
parameters been converted to `string`.

```c++
inline std::vector<std::pair<std::string, std::string> > GetDict(void * head) const {
  std::vector<std::pair<std::string, std::string> > ret;
  for (std::map<std::string, FieldAccessEntry*>::const_iterator
          it = entry_map_.begin(); it != entry_map_.end(); ++it) {
    ret.push_back(std::make_pair(it->first, it->second->GetStringValue(head)));
  }
  return ret;
}
```

#### 6.3.7 Find 

It finds the entry in `entry_map_` by the `key` value.

```c++
inline FieldAccessEntry *Find(const std::string &key) const {
  std::map<std::string, FieldAccessEntry*>::const_iterator it =
      entry_map_.find(key);
  if (it == entry_map_.end()) return NULL;
  return it->second;
}
```

#### 6.3.8 RunInit

It does the initialization to every entry stored, by the contents inside of `RandomAccessIterator`.

It first checks whether the `key` inside of `it->first` has been registered in `entry_map_`.
If it is, it fetches the class `FieldEntry` corresponding to the `key`, and set its actual value
to be `it->second`. Then, it will call `Check()` to examine the satisfication of constraints, and
insert the processed `FieldEntry` into `selected_args`.

Otherwise, it will check several extra conditions to determine whether keep the program running or
raise an error. We will leave the discussion until we meet the case that uses them.

After the initialization, it checks every component inside `entry_map_`, if there are components
not being initialized, the function `SetDefault()` will be called.

```c++
template<typename RandomAccessIterator>
inline void RunInit(void *head,
                    RandomAccessIterator begin,
                    RandomAccessIterator end,
                    std::vector<std::pair<std::string, std::string> > *unknown_args,
                    parameter::ParamInitOption option) const {
  std::set<FieldAccessEntry*> selected_args;
  for (RandomAccessIterator it = begin; it != end; ++it) {
    FieldAccessEntry *e = Find(it->first);
    if (e != NULL) {
      e->Set(head, it->second);
      e->Check(head);
      selected_args.insert(e);
    } else {
      if (unknown_args != NULL) {
        unknown_args->push_back(*it);
      } else {
        if (option != parameter::kAllowUnknown) {
          if (option == parameter::kAllowHidden &&
              it->first.length() > 4 &&
              it->first.find("__") == 0 &&
              it->first.rfind("__") == it->first.length()-2) {
            continue;
          }
          std::ostringstream os;
          os << "Cannot find argument \'" << it->first << "\', Possible Arguments:\n";
          os << "----------------\n";
          PrintDocString(os);
          throw dmlc::ParamError(os.str());
        }
      }
    }
  }

  for (std::map<std::string, FieldAccessEntry*>::const_iterator it = entry_map_.begin();
       it != entry_map_.end(); ++it) {
    if (selected_args.count(it->second) == 0) {
      it->second->SetDefault(head);
    }
  }
}
```

## 7. ParamManagerSingleton

In the execution, the typename `PType` will be replaced by the type of use-defined `struct`.
In fact, the process of its initialization is actually also the initialization process for 
`ParamManager`.

It first defines a `param` with `PType`. Then, call the member function `__DECLARE__()` of `PType`
with pointer `this` passing into it. Inside the `__DECLARE__`, we do the initialization of `PType`
by manipulating the member variable `manager` with type `ParamManager` through `this` pointer. 
At last, it set the name of the `manager` according to the name of the user-defined `struct`.

```c++
template<typename PType>
struct ParamManagerSingleton {
  ParamManager manager;
  explicit ParamManagerSingleton(const std::string &param_name) {
    PType param;
    param.__DECLARE__(this);
    manager.set_name(param_name);
  }
};
```

The detailed discussion of this class should be combined with the following contents.

## 8. Macro

In `./parameter.h` we introduce several macros to declare parameter.

### 8.1 DMLC_DECLARE_PARAMETER

`DMLC_DECLARE_PARAMETER` is used inside the `struct` as a declaration or definition of
member functions.

The usage of `DMLC_DECLARE_PARAMETER` is firstly declare a function named `__MANAGER__()`,
then define a function named `__DECLARE__()` whose input variable is with type
`ParamManagerSingleton`.

```c++
#define DMLC_DECLARE_PARAMETER(PType)                                   \
  static ::dmlc::parameter::ParamManager *__MANAGER__();                \
  inline void __DECLARE__(::dmlc::parameter::ParamManagerSingleton<PType> *manager) \
```

### 8.2 DMLC_DECLARE_FIELD

`DMLC_DECLARE_FIELD` should follow right behind `DMLC_DECLARE_PARAMETER`, it call 
the `DECLARE()` function of class `Parameter` to register parameter.

```c++
#define DMLC_DECLARE_FIELD(FieldName)  this->DECLARE(manager, #FieldName, FieldName)
```

### 8.3 DMLC_DECLARE_ALIAS

`DMLC_DECLARE_ALIAS` is also used inside the `struct` to store parameters. It always
appears after `DMLC_DECLARE_PARAMETER` and `DMLC_DECLARE_FIELD` to set alias of 
defined parameters by calling member function `AddAlias()` of class `Parameter`.

```c++
#define DMLC_DECLARE_ALIAS(FieldName, AliasName)  manager->manager.AddAlias(#FieldName, #AliasName)
```

### 8.4 DMLC_REGISTER_PARAMETER

This macro need to be put in a source file so that the `__MANAGER__()` function declared inside
the `struct` can be defined.

The usage of `DMLC_REGISTER_PARAMETER` is to
 define the function `__MANAGER__()` declared in the `struct`, and
call the defined function `__MANAGER__()` (I wonder the reason of its existence).

```c++
#define DMLC_REGISTER_PARAMETER(PType)                                  \
  ::dmlc::parameter::ParamManager *PType::__MANAGER__() {               \
    static ::dmlc::parameter::ParamManagerSingleton<PType> inst(#PType); \
    return &inst.manager;                                               \
  }                                                                     \
  static DMLC_ATTRIBUTE_UNUSED ::dmlc::parameter::ParamManager&         \
  __make__ ## PType ## ParamManager__ =                                 \
      (*PType::__MANAGER__())                                           \
```

We indicates `__make__ ## PType ## ParamManager__` will never be used in the 
context. It only triggers the function `__MANAGER__()` and accpets its return.



## 9. Parameter

`Parameter` is the base type that every parameter struct should inheritate from.

The following code is a complete example to setup parameters. Note: It is 
important for every parameter `struct` inherited from class `Parameter<PType>`,
where `PType` is itself.

```c++
struct Param : public dmlc::Parameter<Param> {
  float learning_rate;
  int num_hidden;
  std::string name;
  // declare parameters in header file
  DMLC_DECLARE_PARAMETER(Param) {
    DMLC_DECLARE_FIELD(num_hidden).set_range(0, 1000);
    DMLC_DECLARE_FIELD(learning_rate).set_default(0.01f);
    DMLC_DECLARE_FIELD(name).set_default("hello");
  }
};

// register it in cc file
DMLC_REGISTER_PARAMETER(Param);
```

The only difference between a normal `struct` is that we will need to 
declare all the `fields`, as well as their default value or constraints.

### 9.1 Member Functions

#### 9.1.1 `protected:` DECLARE

`DECLARE()` function is the most important member function of `Parameter`.
Everytime we want to manipulate the stored parameter inside `struct`, it will
be called.

Inside of it, it constructs and inits a new instance with type `FieldEntry<DType>`. 
The `DType` is determined by the type of its input variable `ref`. At last, it 
add the entry into the `ParamManager`.

```c++
template<typename DType>
inline parameter::FieldEntry<DType>& DECLARE(parameter::ParamManagerSingleton<PType> *manager,
                                             const std::string &key, DType &ref) {
  parameter::FieldEntry<DType> *e = new parameter::FieldEntry<DType>();
  e->Init(key, this->head(), ref);
  manager->manager.AddEntry(key, e);
  return *e;
}
```

At here, we can conclude the execution process of the calling `PType::__MANAGER__()`.

- (1) It initializes a new `ParamManagerSingleton`. Inside of the constructor, a temporal
`struct` will be built. And the `__DECLARE__()` function of the temporal `struct` will
be called to initialize a new `FieldEntry<DType>` added as a new entry of the `ParamManager`.

- (2) The `ParamManager` will be returned for further execution of the `struct` like 
`Init()`, `__DICT__()` or `__DOC__()`.



#### 9.1.2 `public:` Init

It initializes the parameter by keyword arguments, stored in a container, which can 
be a `map` or a `vector` containing `pair`. Inside this function, parameter `struct`
will bo initialized, and every parameter will be checked and error will be thrown
if something is wrong.

```c++
template<typename Container>
inline void Init(const Container &kwargs,
                 parameter::ParamInitOption option = parameter::kAllowHidden) {
  PType::__MANAGER__()->RunInit(static_cast<PType*>(this),
                                kwargs.begin(), kwargs.end(),
                                NULL,
                                option);
}
```

Followe the example before, the usage of `Init()` function is:

```c++
int main() {
   MyParam param;
   std::vector<std::pair<std::string, std::string> > param_data = {
     {"num_hidden", "100"},
     {"learning_rate", "0.1f"},
     {"name", "MyNet"}
   };
   // set the parameters
   param.Init(param_data);
   return 0;
}
```

The execution process is as follows. First, it calls the `__MANAGER__()` function of
the `struct`.

```c++
ParamManager *PType::__MANAGER__() {
  static ::dmlc::parameter::ParamManagerSingleton<PType> inst(#PType);
  return &inst.manager;                                              
} 
```

Inside the `__MANAGER__()`, a `ParamManagerSingleton` will be created, whose constructor
will call the member function `__DECLARE__()` of the `struct`. Inside struct, every parameter 
related default value or constraints will all be set. Then, it returns the member variable
`manager` with type `ParamManager`.

Second, the member function `RunInit()` of returned `manager` will be called. We only consider
the case that all parameter are correctly set, so all parameters are set inside the `RunInit()`
referred to its description in this notebook.

```note
Let us summarize the usage of `./parameter.h`. It is major difference to normal structure is it
introduce more information into the struct like default value, constraints, and descriptions.

Indeed, the parameter is stored in the same way as usual. But, the difference is the way to 
manipulate them. Everytime we want to manipulate them, a temporal `ParamManager` will be created
to handle it.

Once we finish our task, we will run into the end of function `Init()`, which will automatically 
trigger the destructor of `ParamManager`.
```

#### 9.1.3 `private:` head

`head()` returns head pointer of child structure

```c++
inline PType *head() const {
  return static_cast<PType*>(const_cast<Parameter<PType>*>(this));
}
```

#### 9.1.4 `public:` \__DICT__

It gets the internal parameters and push them into a vector of pairs. 
The `first` value in the `pair` is the name of the instance, and the 
`second` value is the content of the internal parameters been converted to `string`.
Then, it use `range constructor` to initialize a `map`.

Note of range constructor, We can also initialize an associative container from a range of values, 
so long as those values can be converted to the type of the container.


```c++
inline std::map<std::string, std::string> __DICT__() const {
  std::vector<std::pair<std::string, std::string> > vec
      = PType::__MANAGER__()->GetDict(this->head());
  return std::map<std::string, std::string>(vec.begin(), vec.end());
}
```


#### 9.1.5 `public:` \__FIELDS__

It simply return the result of function `GetFieldInfo()` of a `ParamManager` type.

```c++
inline static std::vector<ParamFieldInfo> __FIELDS__() {
  return PType::__MANAGER__()->GetFieldInfo();
}
```

#### 9.1.6 `public:` \__DOC__

It simply output the result of function `PrintDocString()` of a `ParamManager` type.

```c++
inline static std::string __DOC__() {
  std::ostringstream os;
  PType::__MANAGER__()->PrintDocString(os);
  return os.str();
}
```





## Missing Explanation

### std::string::length()
`size_t length() const;`

`std::string::length()` returns the length of the string, in terms of bytes. e.g.

```c++
#include <iostream>
#include <string>

int main ()
{
  std::string str ("Test string");
  std::cout << "The size of str is " << str.length() << " bytes.\n";
  return 0;
}

// output: The size of str is 11 bytes
```

### map::find()

`map::find(k)` searches the container for an element with a key 
equivalent to k and returns an iterator to it if found, otherwise 
it returns an iterator to map::end.

### map::size()

`map::size()` returns the number of elements in the map container.

### map::count()

`map::count(k)` searches the container for elements with a key equivalent 
to `k` and returns the number of matches.

Because all elements in a map container are unique, the 
function can only return `1` (if the element is found) or 
`0` (otherwise).


### map::at()

`std::at(k)` returns a reference to the mapped value of the element 
identified with key `k`.

If `k` does not match the key of any element in the container, 
the function throws an `out_of_range` exception.

### std::resize()

```c++
void resize (size_t n);
void resize (size_t n, char c);
```

`std::resize()` resizes the string to a length of `n` characters.

If `n` is smaller than the current string length, the current value 
is shortened to its first `n` character, removing the characters beyond the `n`-th.

If `n` is greater than the current string length, the current content is extended by 
inserting at the end as many characters as needed to reach a size of `n`. If `c` is 
specified, the new elements are initialized as copies of `c`, otherwise, they are 
value-initialized characters (null characters).

### std::transform()

```c++
template< class InputIt, class OutputIt, class UnaryOperation >
OutputIt transform(InputIt first1, InputIt last1, OutputIt d_first, UnaryOperation unary_op);
```

`std::transform()` applies the given function to a range and stores the result in another range, beginning at `d_first`.
The unary operation `unary_op` is applied to the range defined by `[first1, last1)`.


### std::tolower()

`std::tolower()` converts the given character to lowercase according to the character conversion rules.

### Special Character in `#define`

- `##`: is to combine two operatees, e.g. `#define conn(x,y) x##y` will result `int n = conn(123,456)` as `n = 123456`
- `#@`: is to add single quotes, e.g. `#define ToChar(x) #@x` will result `char a = ToChar(1)` as `a = '1'`
- `#` : is to add double quotes, e.g. `#define ToString(x) #x` will result `char* str = ToString(123)` as `str = "123"`

### DMLC_ATTRIBUTE_UNUSED

Normally, the compiler warns if a variable is declared but is never referenced. 
`__attribute__((unused))` informs the compiler that you expect a variable to be unused and 
tells it not to issue a warning if it is not used. So, in `dmlc-core`, we define

```c++
#if defined(__GNUC__)
#define DMLC_ATTRIBUTE_UNUSED __attribute__((unused))
#else
#define DMLC_ATTRIBUTE_UNUSED
#endif
```

## Missing Component 

### Parameter::InitAllowUnknown

We will consider it later, when we meet it

```c++
template<typename Container>
inline std::vector<std::pair<std::string, std::string> >
InitAllowUnknown(const Container &kwargs) {
  std::vector<std::pair<std::string, std::string> > unknown;
  PType::__MANAGER__()->RunInit(static_cast<PType*>(this),
                                kwargs.begin(), kwargs.end(),
                                &unknown, parameter::kAllowUnknown);
  return unknown;
}
```

### Parameter::Save

We will leave it to the discussion of `./json.h`

```c++
inline void Save(dmlc::JSONWriter *writer) const {
  writer->Write(this->__DICT__());
}
```

### Parameter::Load

We will leave it to the discussion of `./json.h`

```c++
inline void Load(dmlc::JSONReader *reader) {
  std::map<std::string, std::string> kwargs;
  reader->Read(&kwargs);
  this->Init(kwargs);
}
```

### FieldEntry<optional<int> >

Currently, we have no idea of its usage.

```c++
template<>
class FieldEntry<optional<int> > : public FieldEntryBase<FieldEntry<optional<int> >, optional<int> > {
 public:
  // construct
  FieldEntry<optional<int> >() : is_enum_(false) {}
  // parent
  typedef FieldEntryBase<FieldEntry<optional<int> >, optional<int> > Parent;
  // override set
  virtual void Set(void *head, const std::string &value) const {
    if (is_enum_ && value != "None") {
      std::map<std::string, int>::const_iterator it = enum_map_.find(value);
      std::ostringstream os;
      if (it == enum_map_.end()) {
        os << "Invalid Input: \'" << value;
        os << "\', valid values are: ";
        PrintEnums(os);
        throw dmlc::ParamError(os.str());
      } else {
        os << it->second;
        Parent::Set(head, os.str());
      }
    } else {
      Parent::Set(head, value);
    }
  }
  virtual ParamFieldInfo GetFieldInfo() const {
    if (is_enum_) {
      ParamFieldInfo info;
      std::ostringstream os;
      info.name = key_;
      info.type = type_;
      PrintEnums(os);
      if (has_default_) {
        os << ',' << "optional, default=";
        PrintDefaultValueString(os);
      } else {
        os << ", required";
      }
      info.type_info_str = os.str();
      info.description = description_;
      return info;
    } else {
      return Parent::GetFieldInfo();
    }
  }
  // add enum
  inline FieldEntry<optional<int> > &add_enum(const std::string &key, int value) {
    CHECK_NE(key, "None") << "None is reserved for empty optional<int>";
    if ((enum_map_.size() != 0 && enum_map_.count(key) != 0) || \
        enum_back_map_.count(value) != 0) {
      std::ostringstream os;
      os << "Enum " << "(" << key << ": " << value << " exisit!" << ")\n";
      os << "Enums: ";
      for (std::map<std::string, int>::const_iterator it = enum_map_.begin();
           it != enum_map_.end(); ++it) {
        os << "(" << it->first << ": " << it->second << "), ";
      }
      throw dmlc::ParamError(os.str());
    }
    enum_map_[key] = value;
    enum_back_map_[value] = key;
    is_enum_ = true;
    return this->self();
  }

 protected:
  // enum flag
  bool is_enum_;
  // enum map
  std::map<std::string, int> enum_map_;
  // enum map
  std::map<int, std::string> enum_back_map_;
  // override print behavior
  virtual void PrintDefaultValueString(std::ostream &os) const { // NOLINT(*)
    os << '\'';
    PrintValue(os, default_value_);
    os << '\'';
  }
  // override print default
  virtual void PrintValue(std::ostream &os, optional<int> value) const {  // NOLINT(*)
    if (is_enum_) {
      if (!value) {
        os << "None";
      } else {
        CHECK_NE(enum_back_map_.count(value.value()), 0)
            << "Value not found in enum declared";
        os << enum_back_map_.at(value.value());
      }
    } else {
      os << value;
    }
  }


 private:
  inline void PrintEnums(std::ostream &os) const {  // NOLINT(*)
    os << "{None";
    for (std::map<std::string, int>::const_iterator
             it = enum_map_.begin(); it != enum_map_.end(); ++it) {
      os << ", ";
      os << "\'" << it->first << '\'';
    }
    os << '}';
  }
};

```
