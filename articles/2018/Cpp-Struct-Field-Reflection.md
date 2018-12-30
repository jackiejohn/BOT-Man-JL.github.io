﻿# 简单的 C++ 结构体字段反射

> 2019/1/1
> 
> 基于 C++ 14 原生语法，不到 100 行代码：让编译器帮你写 **序列化/反序列化** 代码，告别体力劳动 🙃

## 背景

很多人喜欢把程序员称为 **码农**，程序员也经常嘲讽自己每天都在 **搬砖**。这时候，大家会想：能否构造出一些 **更好的工具**，代替我们做那些无意义的 **体力劳动** 中呢？

在实际 C++ 项目中，我们经常需要实现一些与外部系统交互的 **接口** —— 外部系统传入 JSON 参数，我们的程序处理后，再以 JSON 的格式传回外部系统。这个过程就涉及到了两次数据结构的转换：

- 输入的 JSON 转换为 C++ 数据结构（**反序列化**, _deserialization_）
- C++ 数据结构 转换为 输出的 JSON（**序列化**, _serialization_）

如果传输的 JSON 数据格式比较复杂，那么序列化/反序列化的代码也会变得非常复杂 —— 需要处理 **结构嵌套**、**可选字段**、**输入合法性检查** 等问题。如果为每个 JSON 数据结构都 **人工手写** 一套序列化/反序列化代码，那么 **工作量** 会特别大。

例如，[chromium/headless 的 devtools 接口](https://github.com/chromium/chromium/blob/master/headless/public/internal/headless_devtools_client_impl.h) 里就定义了 33 个 **领域模型** _(domain model)_，而每个模型中又定义了许多字段。如果针对每个模型编写序列化/反序列化代码，那么 Google 的工程师是不会乐意的。。。😑 所以，他们构建了一套 [代码生成工具](https://github.com/chromium/chromium/tree/master/mojo)，[帮助程序员](https://github.com/chromium/chromium/tree/master/components/autofill_assistant/browser/devtools) 完成这些体力劳动。

如果觉得引入一套新的代码生成工具的成本比较高，那么我们不妨考虑让 **编译器** 帮我们完成 **代码生成** 的工作。

## 目标

- 基于 C++ **原生语法**，不需要引入第三方库
- 提供 **声明式** _(declarative)_ 的方法，只需要声明 **格式** _(schema)_，不需要实现具体逻辑
- 不会带来 **额外的运行时开销**，能达到和手写代码一样的运行时效率

基于 [nlohmann 的 C++ JSON 库](https://github.com/nlohmann/json)，给定一个 C++ 结构体 `SimpleStruct`：

``` cpp
struct SimpleStruct {
  bool bool_;
  int int_;
  double double_;
  std::string string_;
  std::vector<double> vector_;
  std::unique_ptr<bool> optional_;
};
```

- 由于 [`std::optional`](https://en.cppreference.com/w/cpp/utility/optional) 需要 C++ 17 支持，所以我们使用 [`std::unique_ptr`](https://en.cppreference.com/w/cpp/memory/unique_ptr) 表示 **可选字段**
- 针对 **可选字段** 的 JSON 序列化/反序列化 **扩展代码**，见 [`optional_json.h`](Cpp-Struct-Field-Reflection/third_party/optional_json.h)（参考：[How do I convert third-party types? | nlohmann/json](https://github.com/nlohmann/json#how-do-i-convert-third-party-types)）

一般的业务处理，往往包括三部分：

- 解析输入（字符串到 JSON 对象的转换 + JSON 对象到领域模型的 **反序列化**）
- 处理业务逻辑（实际需要我们写的代码）
- 转储输出（领域模型到 JSON 对象的 **序列化** + JSON 对象到字符串的转换）

``` cpp
// input
json json_input = json::parse(
    "{"
    "  \"_bool\": true,"
    "  \"_int\": 1,"
    "  \"_double\": 1.0,"
    "  \"_string\": \"hello json\","
    "  \"_vector\": [1, 1.0]"
    "}");
SimpleStruct object = json_input.get<SimpleStruct>();

// use
object.string += " in simple struct";

// output
json json_output = json(object);
std::string string_output = json_output.dump();
```

- 对于 JSON 对象和字符串之间的转换，主流的 **JSON 库都实现** 了：
  - 调用 `json::parse` 从字符串得到输入 JSON 对象
  - 调用 `json::dump` 将 JSON 对象转为用于输出的字符串
- 而 JSON 对象和 C++ 结构体之间的转换，**需要我们实现**：
  - 通过反序列化，调用 `json::get<SimpleStruct>()` 得到 `SimpleStruct object`
  - 通过序列化，使用 `object` 构造输出 JSON 对象

## 实现

实现从 C++ 结构体到 JSON 的序列化/反序列化操作，需要用到以下信息：

- 结构体有 **哪些字段**
  - `bool_`/`int_`/`double_`/`string_`/`vector_`/`optional_`
- 每个 **字段** 在 **结构体中** 的什么 **位置**
  - `&SimpleStruct::bool_`/`&SimpleStruct::int_`/`&SimpleStruct::double_`/`&SimpleStruct::string_`/`&SimpleStruct::vector_`/`&SimpleStruct::optional_`
- 每个 **字段** 在 **JSON 中** 对应的 **名称** 是什么
  - `"_bool"`/`"_int"`/`"_double"`/`"_string"`/`"_vector"`/`"_optional"`
- 每个 **字段** 如何从 C++ 到 JSON 进行 **类型映射**
  - `bool` 对应 `Boolean`，`int` 对应 `Number(Integer)`，`double` 对应 `Number`，`string` 对应 `String`，`vector` 对应 `Array`
  - 必选字段 **缺失** 或字段类型与 JSON 数据 **类型不匹配**，则抛出异常
  - 如果 **可选字段**（例如 `optional_`）缺失，则跳过检查

对于很多支持 [**反射** _(reflection)_](https://en.wikipedia.org/wiki/Reflection_%28computer_programming%29) 的语言，**JSON 的解析者** 可以通过反射接口，查询到 `SimpleStruct` 所有的 **字段信息**。

尽管 C++ 支持 [**运行时类型信息** _(RTTI, run-time type information)_](https://en.wikipedia.org/wiki/Run-time_type_information)，但无法得到上述所有信息。所以，对于暂时还 **不支持反射** 的 C++ 语言，需要 **`SimpleStruct` 的定义者** 把这些信息告诉 **JSON 的解析者**。

### 人工手写 序列化/反序列化 代码

> [代码链接](Cpp-Struct-Field-Reflection/raw_json.cc)

实现序列化/反序列化最简单的方法，就是通过 **人工编写** 代码：

``` cpp
void to_json(nlohmann::json& j, const SimpleStruct& value) {
  j["bool"] = value.bool_;
  j["int"] = value.int_;
  j["double"] = value.double_;
  j["string"] = value.string;
  j["vector"] = value.vector;
  j["optional"] = value.optional;
}

void from_json(const nlohmann::json& j, SimpleStruct& value) {
  j.at("bool").get_to(value.bool_);
  j.at("int").get_to(value.int_);
  j.at("double").get_to(value.double_);
  j.at("string").get_to(value.string);
  j.at("vector").get_to(value.vector);
  if (j.find("optional") != j.cend()) {
    j.at("optional").get_to(value.optional);
  }
}
```

- 在 `to_json`/`from_json` 包含了 **所有字段** 的 **位置、名称、映射方法**：
  - 使用 `j[name] = field` 序列化
  - 使用 `j.at(name).get_to(field)` 反序列化
  - 针对可选字段检查字段是否存在，不存在则跳过
- nlohmann 的 C++ JSON 库基于 C++ 原生的 **异常处理**（`throw-try-catch`）：
  - 如果字段不存在，函数 `json::at` 抛出异常
  - 如果字段实际类型和 JSON 输入类型不匹配，函数 `json::get_to` 抛出异常

手写 `to_json`/`from_json` 需要写 2 份类似的代码，导致 **代码冗余**；另外，这两份代码还需要做不相似的处理（例如，判断字段是否为 **可选字段**），不易于用统一编写。

### 动态反射

得益于 nlohmann 的 C++ JSON 库易用性比较好（已经支持了 **结构嵌套** 的处理，并通过异常处理机制实现 **输入合法性检查**），即使手写代码，写起来也比较简单。

如果我们使用 [chromium/`base::Value`](https://github.com/chromium/chromium/blob/master/base/values.h) 处理 JSON，还需要 **手写更多代码**。所以 Google 的工程师构建了一种基于 **动态反射** _(dynamic reflection)_ 的反序列化机制，不需要代码生成器，实现统一的 JSON 数据和 C++ 结构体转换。（参考：[chromium/`base::JSONValueConverter`](https://github.com/chromium/chromium/blob/master/base/json/json_value_converter.h)）

**核心原理** 是：利用 [**适配器模式** _(adapter pattern)_](../2017/Design-Patterns-Notes-2.md#Adapter)，通过 **运行时多态** _(runtime polymorphism)_ 机制，使用 **接口** _(interface)_ 抹除具体 **字段信息**（位置、名称、映射方法）的类型，并遍历调用接口进行实际的转换操作。

> Talk is cheap, show me the code ——
> [代码链接](Cpp-Struct-Field-Reflection/dynamic_reflection.h)

首先，为不同 **字段类型** 定义一个通用的转换接口 `ValueConverter<FieldType>`，用于存储实际的 C++ 类型与 JSON 类型的转换操作（**仅关联操作的字段类型，抹除具体的转换操作**）：

``` cpp
template <typename FieldType>
using ValueConverter =
    std::function<void(FieldType* field, const std::string& name)>;
```

- 原代码将 `ValueConverter` 定义为纯接口；这里为了化简，直接使用 `std::function`
- 参数 `field` 是字段的值，`name` 是字段的名称

然后，为不同类型的 **结构体** 定义一个通用的转换接口 `FieldConverterBase<StructType>`，用于存储结构体内所有字段的转换操作（**仅关联结构体的类型，抹除操作的字段类型**）：

``` cpp
template <typename StructType>
class FieldConverterBase {
 public:
  virtual ~FieldConverterBase() = default;
  virtual void operator()(StructType* obj) const = 0;
};
```

接着，通过 `FieldConverter<StructType, FieldType>` 将上边两个接口 **承接** 起来，存储特定 **结构体** 的特定 **字段类型** 的实际转换操作，同时关联上具体某个字段的 **名称** `field_name_` 和 **位置** `field_pointer_`（**实现  `FieldConverterBase` 接口，调用 `ValueConverter` 接口**）：

``` cpp
template <typename StructType, typename FieldType>
class FieldConverter : public FieldConverterBase<StructType> {
 public:
  FieldConverter(const std::string& name,
                 FieldType StructType::*pointer,
                 ValueConverter<FieldType> converter)
      : field_name_(name),
        field_pointer_(pointer),
        value_converter_(converter) {}

  void operator()(StructType* obj) const override {
    return value_converter_(&(obj->*field_pointer_), field_name_);
  }

 private:
  std::string field_name_;
  FieldType StructType::*field_pointer_;
  ValueConverter<FieldType> value_converter_;
};
```

- 构造时传递 字段名称 `field_name_`，字段的 **成员指针** _(member pointer)_（即字段位置）`field_pointer_`，字段的映射方法 `value_converter_`
- 在 `operator()` 转换时，调用 `value_converter_.operator()`，传入字段的值和名称；其中字段的值通过 `obj->*field_pointer_` 得到

最后，针对 **结构体** 定义一个存储 **所有字段** 信息（名称、位置、映射方法）的容器 `StructValueConverter<StructType>`，并提供 **注册** 字段信息的接口（有哪些字段）`RegisterField` 和执行所有转换操作的接口 `operator()`（**仅关联结构体的类型，利用 `FieldConverterBase` 抹除操作的字段信息**）：

``` cpp
template <class StructType>
class StructValueConverter {
 public:
  template <typename FieldType>
  void RegisterField(FieldType StructType::*field_pointer,
                     const std::string& field_name,
                     ValueConverter<FieldType> value_converter) {
    fields_.push_back(std::make_unique<FieldConverter<StructType, FieldType>>(
        field_name, field_pointer, std::move(value_converter)));
  }

  void operator()(StructType* obj) const {
    for (const auto& field_converter : fields_) {
      (*field_converter)(obj);
    }
  }

 private:
  std::vector<std::unique_ptr<FieldConverterBase<StructType>>> fields_;
};
```

> [使用样例代码链接](Cpp-Struct-Field-Reflection/dynamic_iostream.cc)

具体使用时，只需要两步：

1. 构造 `converter` 对象，调用 `RegisterField` **动态绑定字段信息**（名称、位置、映射方法）
2. 调用 `converter(&simple)` 对所有注册了的字段 **进行转换**

``` cpp
// setup converter (partial)
auto int_converter = [](int* field, const std::string& name) {
  std::cout << name << ": " << *field << std::endl;
};
auto string_converter = [](std::string* field, const std::string& name) {
  std::cout << name << ": " << *field << std::endl;
};

StructValueConverter<SimpleStruct> converter;
converter.RegisterField(&SimpleStruct::int_, "int",
                        ValueConverter<int>(int_converter));
converter.RegisterField(&SimpleStruct::string_, "string",
                        ValueConverter<std::string>(string_converter));

// use converter
SimpleStruct simple{2, "hello dynamic reflection"};
converter(&simple);

// output:
//   int: 2
//   string: hello dynamic reflection
```

### 静态反射

实际上，实现序列化/反序列化所需要的信息（有哪些字段，每个字段的位置、名称、映射方法），在 **编译时** _(compile-time)_ 就已经确定了。所以，我们可以利用 **静态反射** _(static reflection)_ 的方法，把这些信息告诉 **编译器**，让它帮我们 **生成代码**，并且不会带来额外的运行时开销。

**核心原理** 是：利用 [**元组** `std::tuple`](https://en.cppreference.com/w/cpp/utility/tuple) 存储 **字段信息**（位置、名称），然后遍历这个元组（有哪些字段），通过 **编译时多态** _(compile-time polymorphism)_ 针对某个特定的 **字段类型** 进行转换操作（映射方法）。

> Talk is cheap, show me the code ——
> [代码链接](Cpp-Struct-Field-Reflection/static_reflection.h)

首先，定义一个 `StructSchema<StructType>` **函数模板** _(function template)_，返回所有字段信息（默认返回空元组）：

``` cpp
template <typename T>
inline constexpr auto StructSchema() {
  return std::make_tuple();
}
```

然后，提供 `DEFINE_STRUCT_SCHEMA` 和 `DEFINE_STRUCT_FIELD` 两个 **宏** _(macro)_ ，定义结构体 **字段信息**（有哪些、位置、名称），隐藏 `StructSchema` 和 `std::tuple` 的实现细节：

``` cpp
#define DEFINE_STRUCT_SCHEMA(Struct, ...)        \
  template <>                                    \
  inline constexpr auto StructSchema<Struct>() { \
    using _Struct = Struct;                      \
    return std::make_tuple(__VA_ARGS__);         \
  }

#define DEFINE_STRUCT_FIELD(StructField, StructName) \
  std::make_tuple(&_Struct::StructField, StructName)
```

- `StructSchema` 返回元组的结构是：`((&field1, name1), (&field2, name2), ...)`
- `DEFINE_STRUCT_SCHEMA` 定义了 **结构体** `Struct` **有哪些字段**
- `DEFINE_STRUCT_FIELD` 定义了每个 **字段** 的 **位置、名称**
- `using _Struct = Struct` 提供了一种宏内数据接力的方法，让下一个宏能获取上一个宏的数据

最后，提供 `ForEachField<StructType>` 函数，从对应的 `StructSchema` 取出存储了结构体 **所有字段信息** 的元组，然后遍历这个元组，从中取出 **每个字段的位置、名称**，作为参数调用 `fn`：

``` cpp
template <typename T, typename Fn>
inline constexpr void ForEachField(T&& value, Fn&& fn) {
  constexpr auto struct_schema = StructSchema<std::decay_t<T>>();
  detail::ForEachTuple(struct_schema, [&value, &fn](auto&& field_schema) {
    fn(value.*(std::get<0>(std::forward<decltype(field_schema)>(field_schema))),
       std::get<1>(std::forward<decltype(field_schema)>(field_schema)));
  });
}
```

- `fn` 接受的参数分别为：字段的值和名称 `(field_value, field_name)`
  - 字段的值通过 `value.*field_pointer` 得到
- `ForEachTuple` 的实现中还用到了 **静态断言** _(static assert)_ 检查，具体见 [代码](Cpp-Struct-Field-Reflection/static_reflection.h)
  - 检查遍历的结构体 **是否有字段**
  - 检查每个字段的信息 **是否都包含了位置和名称**

> [使用样例代码链接](Cpp-Struct-Field-Reflection/static_iostream.cc)

具体使用时，也是需要两步：

1. 使用 `DEFINE_STRUCT_SCHEMA` 和 `DEFINE_STRUCT_FIELD` **静态定义字段信息**（名称、位置）
2. 调用 `ForEachField` 并传入 **映射方法**（函数模板或泛型 lambda 表达式），对所有字段调用这个函数；映射方法针对不同 **字段类型** 的实际转换操作

``` cpp
// define schema (partial)
DEFINE_STRUCT_SCHEMA(
    SimpleStruct,
    DEFINE_STRUCT_FIELD(int_, "int"),
    DEFINE_STRUCT_FIELD(string_, "string"));

// use ForEachTuple
ForEachField(SimpleStruct{1, "hello static reflection"},
             [](auto&& field, auto&& name) {
               std::cout << name << ": "
                         << field << std::endl;
             });

// output:
//   int: 1
//   string: hello static reflection
```

上边代码编译后，几乎等效于下面的代码 —— 除了编译慢一点点，没有额外的运行时开销：

``` cpp
SimpleStruct simple{1, "hello static reflection"};
std::cout << "int" << ": " << simple.int_ << std::endl;
std::cout << "string" << ": " << simple.string_ << std::endl;
```

使用编译时静态反射，相对于运行时动态反射，有许多优点：

| | 动态反射 | 静态反射 |
|-|---------|----------|
| 使用难度 |（难）需要 **编写注册代码**，调用 `RegisterField` 动态绑定字段信息 |（易）可以通过 **声明式** 的方法，静态定义字段信息 |
| 可复用性 |（差）每个 `converter` 对象绑定了各个 **字段类型** 的具体 **映射方法**；如果需要不同的映射方法，则需要另外创建 `converter` 对象 |（好）在调用 `ForEachField` 时，**映射方法** 作为参数传递进去；利用编译时多态的机制，自动为不同的 **字段类型** 选择合适的操作 |
| 运行时开销 |（有）需要动态构造 `converter` 对象，需要通过 **虚函数表** 实现面向对象的多态 |（无）**编译时** 静态展开代码，和直接手写一样 |

### 编译器生成 序列化/反序列化 代码

> [代码链接](Cpp-Struct-Field-Reflection/reflection_json.cc)

利用基于静态反射的 `ForEachField`，我们就可以实现 **通用** 的结构体序列化/反序列化函数了：

``` cpp
template <typename T>
struct adl_serializer<T, std::enable_if_t<::has_schema<T>>> {
  template <typename BasicJsonType>
  static void to_json(BasicJsonType& j, const T& value) {
    ForEachField(value, [&j](auto&& field, auto&& name) {
      j[name] = field;
    });
  }

  template <typename BasicJsonType>
  static void from_json(const BasicJsonType& j, T& value) {
    ForEachField(value, [&j](auto&& field, auto&& name) {
      // ignore missing field of optional
      if (::is_optional_v<decltype(field)> &&
          j.find(name) == j.end())
        return;

      j.at(name).get_to(field);
    });
  }
};
```

- 和 [sec|人工手写 序列化/反序列化 代码] 的代码类似：
  - 使用 `j[name] = field` 序列化
  - 使用 `j.at(name).get_to(field)` 反序列化
  - 针对可选字段检查字段是否存在，不存在则跳过
- 关于如何使用 `nlohmann::adl_serializer` 扩展自定义类型的序列化/反序列化操作，参考 [How do I convert third-party types? | nlohmann/json](https://github.com/nlohmann/json#how-do-i-convert-third-party-types)
- 使用了两个简单的 **变量模板** _(variable template)_，具体见 [代码](Cpp-Struct-Field-Reflection/reflection_json.cc)
  - `has_schema<T>` 检查是否定义了 `StructSchema<T>`
  - `is_optional_v<decltype(field)>` 检查字段类型是不是可选参数

对于需要进行序列化/反序列化的自定义结构体，我们只需要使用 `DEFINE_STRUCT_SCHEMA` 和 `DEFINE_STRUCT_FIELD` 声明其字段信息即可：

``` cpp
DEFINE_STRUCT_SCHEMA(
    SimpleStruct,
    DEFINE_STRUCT_FIELD(bool_, "_bool"),
    DEFINE_STRUCT_FIELD(int_, "_int"),
    DEFINE_STRUCT_FIELD(double_, "_double"),
    DEFINE_STRUCT_FIELD(string_, "_string"),
    DEFINE_STRUCT_FIELD(vector_, "_vector"),
    DEFINE_STRUCT_FIELD(optional_, "_optional"));
```

基于 **声明式** 的语法，让没有编程基础的人都可以写代码：

[align-center]

[img=max-width:70%]

![Declarative](Cpp-Reference-in-Functional/declarative.png)

[align-center]

图片来源：[Declarative Programming And The Web](https://www.smashingmagazine.com/2014/07/declarative-programming/)

于是，编译器生成的代码，基本上和 [sec|人工手写 序列化/反序列化 代码] 的代码一致 —— 没有依赖于第三方库，只需要声明结构体的格式，生成的代码没有额外的运行时开销 —— 这就是 **现代 C++ 元编程**。

## 写在最后

如果还有人认为 C++ 元编程就是 **屠龙之技**，那可能是因为他们还在 **手写重复的代码**。掌握 C++ 元编程，自己打造工具，解放生产力，告别搬砖的生活！

> 延伸阅读：
> 
> - [浅谈 C++ 元编程 by BOT Man](../2017/Cpp-Metaprogramming.md)
> - [Modern C++ 元编程应用 by 祁宇](http://purecpp.org/purecpp/static/1699861ac67c43b1809284fbe77aed87.pdf)
> - [C++ 反射的应用与实践 by 卜恪](http://purecpp.org/purecpp/static/47b2b7e63efb458da091913a1f526811.pdf)

如果有什么问题，**欢迎交流**。😄

Delivered under MIT License &copy; 2019, BOT Man