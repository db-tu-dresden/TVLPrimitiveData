# Template Vector Library - Primitive Data

This repository contains all relevant files for the [TVL Generator](https://github.com/db-tu-dresden/TVLGen/tree/main).
The TVL Generator uses pythons [Jinja2](https://pypi.org/project/Jinja2/) to generate the library code. 
To reduce the generator's overall complexity, we heavily utilize the template and control flow capabilities of Jinja2.
Therefore, every structural relevant entry of the _TVL primitive data_ is represented by a template [[1](#extension-template), [2](#primitive-declaration-template), [3](#primitive-definition-template)] and its associated data [[1](#extension-file)].
The associated data files use the YAML format since it is human-readable, easy to parse, and narrow but expressive.

We provide a [schema](https://github.com/db-tu-dresden/TVLGen/blob/main/config/data/tvl_generator_schema.yaml) containing relevant keywords for data files.
There are two classes of keywords, (i) required and (ii) optional. The required keywords must be included in the YAML object, while the optional ones may be omitted. 
For the most optional parameters, default values exist. If so, the default value provided by the schema will be added on the fly while generating the _TVL_-code. 
As the schema is used for basic type-checking and casting, producing an error if a provided value does not match the specification and can not be cast into the required type, no error is generated for _additional_ values within the YAML document. 
The specified data will be passed into the Jinja2 templates and can be accessed from there to obtain maximum flexibility and extensibility.

Table of Contents:

1. [Extensions](#1-extension)
   1. [Template File](#11-extension-template-file)
   2. [Data File (required and optional fields)](#12-extension-data-file)
   3. [Example](#13-example)
2. [Primitives](#2-primitive)
   1. [Primitive Declaration](#21-primitive-declaration)
      1. [Template File](#211-primitive-declaration-template-file)
   2. [Primitive Definition](#22-primitive-definition)
      1. [Template File](#221-primitive-definition-template-file)
   3. [Data File (required and optional fields)](#23-primitive-data-yaml-object)
   4. [Example](#24-example)
3. [Primitive Classes](#3-primitive-class)

## 1. Extension
An extension encapsulates all relevant information for a specific vector extension in a C-struct.
An extension struct is used by the fundamental template type resolution struct of the _TVL_: the **simd** struct, which exposes all types defined by the extension as well as the base type (representing all values stored in a vector register), and provides some compile-time evaluated helper functions.
###### Possible Implementation of the TVL simd struct
```cpp
template<
    Arithmetic BaseType, TargetExtension< BaseType > TargetExtensionType, std::size_t VectorSizeInBits = TargetExtensionType::default_size_in_bits::value
>
struct simd {
    using base_type = BaseType;
    using target_extension = TargetExtensionType;
    using register_type = typename TargetExtensionType::template types<BaseType, VectorSizeInBits>::register_t;
    using mask_type = typename TargetExtensionType::template types<BaseType, VectorSizeInBits>::mask_t;
    static constexpr std::size_t vector_size_b() {
        return VectorSizeInBits;
    }
    static constexpr std::size_t vector_size_B() {
        /*... */
    }
    static constexpr std::size_t vector_element_count() {
        /*... */
    }
    static constexpr std::size_t vector_alignment() {
        /*... */        
    }
    /*... */
};
```
An extension is limited to the vendor-specific superset of all SIMD extensions which use the same vector register type (/width), e.g., SSE, SSE2, SSSE3, SSE4.1, SSE4.2 are all represented by the extension `sse`.  
This reduces the overall number of specified extensions and improves maintainability.

To include *all* relevant subsets of an extension, the corresponding `lscpu_flags` entry of the extension data file.

### 1.1 Extension Template File
The [extension template](https://github.com/db-tu-dresden/TVLGen/tree/main/config/data/templates/tvl_extension.template) contains a template for an extension which is used together with an extension YAML file to generate an extension C++-header file (using Jinja2).  
An extension struct exposes (only) three types: (i) the default vector size in bits as an integral constant, (ii) a vector register type (e.g., `__m128i`), and (iii) a vector mask type (e.g.g, `__mmask8`).
###### Possible Implementation of an extension struct Jinj2 template 
```cpp
struct {{ simdT_name }} {
    using default_size_in_bits = std::integral_constant< std::size_t, {{ simdT_default_size_in_bits }} >;
    template< Arithmetic BaseType, std::size_t VectorSizeInBits = default_size_in_bits::value >
    struct types {
        using register_t {{ simdT_register_type_attributes }} = {{ simdT_register_type }};
        using mask_t = {{ simdT_mask_type }};
    };
};
```

### 1.2 Extension Data File
There is **a single** YAML file per extension. All extension files must be located within the [extensions](https://github.com/db-tu-dresden/TVLPrimitiveData/tree/main/extensions) directory. 
Note that there are several subdirectories for *SIMD* and one subdirectory for every vendor below this (e.g., intel).  
Every extension YAML file consists of a single YAML object. 

The TVL-schema specifies the following keys (note that this list _may not be_ complete):
<details open>
<summary><b>Required Fields</b></summary>

| Key Name                    |    Type     | Description                                                               |         Example          | Note                                                                                                               |
|-----------------------------|:-----------:|---------------------------------------------------------------------------|:------------------------:|--------------------------------------------------------------------------------------------------------------------|
| `vendor`                      |    `str`    | The vendor name.                                                          |         `"intel"`          |                                                                                                                    |
| `extension_name`              |    `str`    | Name of the specific extension (used as filename).                        |          `"sse"`           |                                                                                                                    |
| `lscpu_flags`                 | `List[str]` | List of extension specific flags (which may be exposed by lscpu).         | `["sse", "sse2", "ssse3"]` |                                                                                                                    |
| `simdT_name`                  |    `str`    | Name of the extension which will be used inside the _TVL_.                  |          `"sse"`           |                                                                                                                    |
| `simdT_default_size_in_bits`  |    `str`    | Default size of a vector register of the extension.                       |          `"128"`           | This can also be a more sophisticated value like "sizeof(BaseType)*8".                                             |
| `simdT_register_type`         |    `str`    | Vector register type, depending on "BaseType" (the underlying data type). |       `"__m128i"`        |                                                                                                                    |
| `simdT_mask_type`             |    `str`    | Mask type.                                                                |       `"__m128i"`        | Thus SSE and AVX(2) does not know arithmetic mask types (e.g., \_\_mmask8), we use a vector register as mask type. |
</details>

<details>
<summary><b>Optional Fields</b></summary>

| Key Name                       |          Type          | Description                                                                                                                                                                                                                                                                                                                                                                                      |                        Example                         | Default Value |
|--------------------------------|:----------------------:|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|:------------------------------------------------------:|---------------|
| `simdT_register_type_attributes` |          `str`           | Attributes of vector register type.                                                                                                                                                                                                                                                                                                                                                              |        `"__attribute__((__may_alias__))"`        | `""`           |
| `description`                    |          `str`           | A description string for the extension used for doxygen generation.                                                                                                                                                                                                                                                                                                                              |               `"Extension struct for SSE"`               | `"todo."`      |
| `includes`                       |       `List[str]`        | A list of all includes needed by the extension.                                                                                                                                                                                                                                                                                                                                                  |                   `["<immintrin.h>"]`                    | `[]`           |
| `intrin_tp`                      |  `Dict[str, List[str]]`  | Intrinsics of a specific extension often follow a specific pattern (return type of the intrinsic or parameter types are encoded into the name). To avoid redundancy in the primitive files, you can specify a dictionary here, containing such information. A key could be a specific data type, the value could be the corresponding mapping. This can than be used for a primitive definition. |  `{"uint8_t": ["epu", "8"], "uint16_t": ["epu", "16"]}`  | `{}`            |
| `intrin_tp_full`                 |     `Dict[str, str]`     | Same as intrin_tp but the List is concatenated to reduce redundancy even more.                                                                                                                                                                                                                                                                                                                   |        `{"uint8_t": "epu8", "uint16_t": "epu16"}`        | `{}`            |
</details>

### 1.3 Example
Intel's Streaming SIMD Extension (SSE) provides 128-bit wide vector registers. 
There are three different vector-datatypes, depending on which values are stored in the vectors: `__m128i` for integral types, `__m128` for 4-byte floating-point data, and `__m128d` for double-precision floating points.
These types could be used in a `std::conditional_t` to determine the corresponding _TVL_ register type. 
However, GCC produces an `ignored attributes` warning when using `std::conditional_t` together with such types. 
While this behavior can be suppressed using the appropriate compiler flags, such a warning may be helpful in a project which uses _TVL_. 
Thus, we recommend defining the register type how _gcc_ does it.  
###### Example yaml file for _TVL_-Extension _SSE_
```yaml
---
description: "Definition of the SIMD TargetExtension sse."
vendor: "intel"
extension_name: "sse"
lscpu_flags: ["sse", "sse2", "ssse3", "sse4_1", "sse4_2" ]
includes: ["<immintrin.h>"]
simdT_name: "sse"
simdT_default_size_in_bits: "128"
simdT_register_type_attributes: |-
   __attribute__ ((
      __vector_size__ (
         VectorSizeInBits/sizeof(
            std::conditional_t(
               (std::is_integral_v< BaseType >),
               long long,
               BaseType
            )
         )
      ), 
      __may_alias__, 
      __aligned__(VectorSizeInBits/sizeof(char))
   ))
simdT_register_type: |-
   std::conditional_t(
      (std::is_integral_v< BaseType >),
      long long,
      BaseType
   )
simdT_mask_type: "register_t"
...
```
The YAML object for Intels [_SSE_](#example-yaml-file-for-_tvl_-extension-_sse_) is used together with the corresponding [template](#possible-implementation-of-an-extension-struct-jinj2-template) to generate the following code. 
###### Possible Implementation of sse-extension struct
```cpp
struct sse {
    using default_size_in_bits = std::integral_constant< std::size_t, 128 >;
    template< Arithmetic BaseType, std::size_t VectorSizeInBits = default_size_in_bits::value >
    struct types {
        using type = std::conditional_t<std::is_integral_v<BaseType>, long long, BaseType>;
        using register_t 
            __attribute__((
                __vector_size__(128/sizeof(type)),
                __may_alias__,
                __aligned__(VectorSizeInBits/sizeof(char))
            )) = type;
        using mask_t = register_t;
    };
};
```




## 2. Primitive

A primitive is the _TVL_-equivalent to a SIMD-intrinsic. Primitives are organized in [Primitive Classes](#3-primitive-class). 

For every primitive class, a single YAML file exists, containing all related primitives.
Every primitive in the _TVL_ exists as a fully templated function (_Primitive Declaration_), 
which calls the static member function of the associated _Primitive Definition_ struct.
The introduced indirection allows partial specialization of primitives.

### 2.1 Primitive Declaration
A primitive declaration defines the general setting for a primitive. Consequently, the name must be specified, and other mandatory information must be provided (most of them have default values and are therefore optional).

#### 2.1.1 Primitive Declaration Template File
The primitive declaration consists of two parts: (i) the template function and (ii) a forward declaration of the implementation struct (within namespace `details`). 
The template function forwards the given arguments into the static member function of the struct and returns its result. 

###### Possible Implementation of a primitive declaration Jinj2 template 
```cpp
namespace {{ tvl_implementation_namespace }} {
  template< VectorProcessingStyle {{ vector_name }}, ImplementationDegreeOfFreedom {{ idof_name }} >
  struct {{ primitive_name }}_impl{};
}
{{ tvl_function_doxygen }}
template< VectorProcessingStyle {{ vector_name }}, ImplementationDegreeOfFreedom {{ idof_name }} = workaround{{ ns.parameter_pack_typenames_str }} >
  {{ '[[nodiscard]] ' if return_type != 'void' else '' }}
  {{ 'TVL_FORCE_INLINE ' if force_inline else '' -}}
  {{ returns['ctype'] }} {{ primitive_name }}(
    {{ ns.full_qualified_parameters_str }}
  ) {
    {{ 'return ' if returns['ctype'] != 'void' else '' -}}
    {{ tvl_implementation_namespace }}::{{ primitive_name }}_impl< {{ vector_name }}, {{ idof_name }} >::apply(
      {{ ns.parameters_str }}
    );
  }
```
### 2.2 Primitive Definition
A primitive definition is a (partial) template specialization of the primitive implementation struct. 
A definition can be fully specialized for a specific _TVL_ extension, with a specific vector size and a degree of freedom.
It is also possible that it is only partial specialized for a specific _TVL_ extension (e.g., `simd<uint32_t, avx>`) or only for a simd extension (e.g. `simd< T, avx>`). 
In addition, the definition can be vector length agnostic (which is useful for ARM-SVE or GPU implementations).

Every definition exposes whether the implementation is natively supported by the underlying hardware. The corresponding information is retrieved from a specific flag from the primitive definition data object.

Only the definitions which are available at the given hardware and were specified from the generator cli are generated at all, if the lscpu flags specified in the [primitive data yaml object](#23-primitive-data-yaml-object) is a subset of the targets requested from the generator.

#### 2.2.1 Primitive Definition Template File
```cpp
namespace {{ tvl_implementation_namespace }} {
  template< 
    {{ 'typename T, ' if ns.partial_specialized else '' }}
    {{ 'std::size_t VectorSize,' if vector_length_agnostic else '' }} 
    ImplementationDegreeOfFreedom {{ idof_name }} 
  >
    struct {{ primitive_name }}_impl<simd<{{ ctype }}, {{ target_extension }}{{ ns.simd_register_length }}>, {{ idof_name }}> {
      using {{ vector_name }} = simd<{{ ctype }}, {{ target_extension }} {{ ', {{ vector_length_bits }}' if vector_length_bits != 0 else '' }}>;
      static constexpr bool native_supported() {
        return {{ 'true;' if is_native else 'false;' }}
      }
      {{ tvl_function_doxygen }}
      {% if ns.contains_parameter_pack %}
      template< {{ ns.parameter_pack_typenames_str }} >
      {%- endif -%}
      {{ '[[nodiscard]] ' if returns['ctype'] != 'void' else ''}}
      {{ '' if is_native else "TVL_NO_NATIVE_SUPPORT_WARNING" }}
      {{ 'TVL_FORCE_INLINE ' if force_inline else '' -}}
      static {{ returns['ctype'] }} apply(
        {{ ns.full_qualified_parameters_str }}
      ) {
        {%- if not is_native %}
        static_assert( !std::is_same_v< {{ idof_name }}, native >, "The primitive {{ primitive_name }} is not supported by your hardware natively while it is forced by using native" );
        {% endif %}
        {{ implementation }}
      }
    };
}
```

### 2.3 Primitive Data YAML Object

### 2.4 Example

The following keys are defined by the schema (note that this list _may not be_ complete):

<details open>
<summary><b>Required Fields</b></summary>

| Key Name                     |    Type     | Description                                                                  |          Example           |
|------------------------------|:-----------:|------------------------------------------------------------------------------|:--------------------------:|
| `primitive_name`             |    `str`    | The name of the primitive which is used to call it from outside the library. |          `"add"`           |
</details>

<details>
<summary><b>Optional Fields</b></summary>

| Key Name       |        Type        | Description                                                                                        |                                                                             Example                                                                             | Default Value                          |
|----------------|:------------------:|----------------------------------------------------------------------------------------------------|:---------------------------------------------------------------------------------------------------------------------------------------------------------------:|----------------------------------------|
| `parameters`   | `List[Parameter]`  | List of [paramater dictionary](#primitive-parameter).                                              | `[{ctype: 'Vec::vector_type const &', name: 'a', description: 'first summand'}, {ctype: 'Vec::vector_type const &', name: 'b', description: 'second summand'}]` | `[]`                                   |
| `returns`      |    `ReturnVal`     | A [return](#primitive-return) value.                                                               |                                              `{"ctype": "Vec::vector_type", "description": "Sum of two vectors."}`                                              | `{"ctype": "void", "description": ""}` |
| `force_inline` |       `bool`       | A flag indicating, whether the function call should be inlined or not.                             |                                                                             `True`                                                                              | `True`                                 |
| `includes`     |    `List[str]`     | A list of all includes needed by the primitive (note that this should be implementation-agnostic). |                                                                          `[<cstddef>]`                                                                          | `[]`                                   |
| `definitions`  | `List[Definition]` | List of [definitions](#primitive-definition).                                                      |                                                               see section _Primitive Definition_                                                                | `[]`                                   |
</details>

<details>
<summary><b>Additional Optional Fields</b></summary>

| Key Name                       | Type  | Description                                                                                                  |                                 Example                                 | Default Value |
|--------------------------------|:-----:|--------------------------------------------------------------------------------------------------------------|:-----------------------------------------------------------------------:|---------------|
| `tvl_implementation_namespace` | `str` | Namespace for template specializations.                                                                      |                               `"details"`                               | `"details"`   |
| `brief_description`            | `str` | Brief description of the primitive (used for documentation generation).                                      |                           `"Add primitive."`                            | `"todo."`     |
| `detailed_description`         | `str` | Detailed description of the primitive (used for documentation generation).                                   | `"Adds two vector registers provided as input and returns the result."` | `"todo."`     |
| `vector_name`                  | `str` | The template class name of the [vector extension](#extension)                                                |                                 `"Vec"`                                 | `"Vec"`       |
| `idof_name`                    | `str` | Template class which is used to define the degree of freedom when mapping a primitive to its specialization. |                                `"Idof"`                                 | `"Idof"`      |
</details>

#### Primitive Parameter
Here comes the description of primitive parameters

#### Primitive Return
Here comes the primitive returns schema


### Primitive Definition

### Example of a generated TVL-primitive add()
<table>
<tr>
<td align="center">
Primitive Declaration
</td>
<td align="center">
Primitive Definition
</td>
</tr>
<tr>
<td>

```cpp
namespace tvl {
   namespace details {
   
      template<
         VectorProcessingStyle Vec, 
         ImplementationDegreeOfFreedom Idof
      >
      struct add_impl{};
      
   }
   
   template<
      VectorProcessingStyle Vec, 
      ImplementationDegreeOfFreedom Idof = workaround
   >
   [[nodiscard]] inline __attribute__((always_inline))
   typename Vec::register_type add(
      const typename Vec::register_type & a, 
      const typename Vec::register_type & b
   ) {
      return details::add_impl<Vec, Idof>::apply(vec_a, vec_b);
   }
   
}
```
</td>
<td>

```cpp
namespace tvl {
   namespace details {
   
      template<ImplementationDegreeOfFreedom Idof>
      struct add_impl<simd<int64_t, sse>, Idof>{
         using Vec = simd<int64_t, sse>;
         static constexpr bool native_supported() { 
            return true; 
         }
         
         [[nodiscard]] inline __attribute__((always_inline))
         static typename Vec::register_type apply(
            const typename Vec::register_type & a, 
            const typename Vec::register_type & b
         ) {
            return _mm_add_epi64(a,b);
         }
      };
      
   }
}
```
</td>
</tr>
</table>


encapsulates all relevant information for a specific vector extension.

We decided to limit the extensions to their corresponding supersets, e.g., (SSE, SSE2, SSSE3, SSE4.1, SSE4.2) are all represented by the extension **sse**.
This reduces the overall number of specified extensions and improves the comprehensibility. 
To include all relevant subsets of an extension, the corresponding lscpu entries has to be included in the extension description.

There is **one** yaml file per extension. All extension files are placed within [./extensions](https://github.com/db-tu-dresden/TVLPrimitiveData/tree/main/extensions). 
Note that there are several subdirectories for *simd* (and *simt*) and one subdirectory for every vendor below those (e.g., intel).  
Every extension yaml file consists of a single yaml object. 

An extension has the following keys (note that this list _may not be_ completed):
<details>
<summary>Required Fields</summary>

| Key Name                    |    Type     | Description                                                               |         Example          | Note                                                                                                               |
|-----------------------------|:-----------:|---------------------------------------------------------------------------|:------------------------:|--------------------------------------------------------------------------------------------------------------------|
| `vendor`                      |    `str`    | The vendor name.                                                          |         `"intel"`          |                                                                                                                    |
| `extension_name`              |    `str`    | Name of the specific extension (used as filename).                        |          `"sse"`           |                                                                                                                    |
| `lscpu_flags`                 | `List[str]` | List of extension specific flags (which may be exposed by lscpu).         | `["sse", "sse2", "ssse3"]` |                                                                                                                    |
| `simdT_name`                  |    `str`    | Name of the extension which will be used inside the _TVL_.                  |          `"sse"`           |                                                                                                                    |
| `simdT_default_size_in_bits`  |    `str`    | Default size of a vector register of the extension.                       |          `"128"`           | This can also be a more sophisticated value like "sizeof(BaseType)*8".                                             |
| `simdT_register_type`         |    `str`    | Vector register type, depending on "BaseType" (the underlying data type). |       `"__m128i"`        |                                                                                                                    |
| `simdT_mask_type`             |    `str`    | Mask type.                                                                |       `"__m128i"`        | Thus SSE and AVX(2) does not know arithmetic mask types (e.g., \_\_mmask8), we use a vector register as mask type. |
</details>

<details>
<summary>Optional Fields</summary>

| Key Name                       |          Type          | Description                                                                                                                                                                                                                                                                                                                                                                                      |                        Example                         | Default Value |
|--------------------------------|:----------------------:|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|:------------------------------------------------------:|---------------|
| `simdT_register_type_attributes` |          `str`           | Attributes of vector register type.                                                                                                                                                                                                                                                                                                                                                              |        `"__attribute__((__may_alias__))"`        | `""`           |
| `description`                    |          `str`           | A description string for the extension used for doxygen generation.                                                                                                                                                                                                                                                                                                                              |               `"Extension struct for SSE"`               | `"todo."`      |
| `includes`                       |       `List[str]`        | A list of all includes needed by the extension.                                                                                                                                                                                                                                                                                                                                                  |                   `["<immintrin.h>"]`                    | `[]`           |
| `intrin_tp`                      |  `Dict[str, List[str]]`  | Intrinsics of a specific extension often follow a specific pattern (return type of the intrinsic or parameter types are encoded into the name). To avoid redundancy in the primitive files, you can specify a dictionary here, containing such information. A key could be a specific data type, the value could be the corresponding mapping. This can than be used for a primitive definition. |  `{"uint8_t": ["epu", "8"], "uint16_t": ["epu", "16"]}`  | `{}`            |
| `intrin_tp_full`                 |     `Dict[str, str]`     | Same as intrin_tp but the List is concatenated to reduce redundancy even more.                                                                                                                                                                                                                                                                                                                   |        `{"uint8_t": "epu8", "uint16_t": "epu16"}`        | `{}`            |
</details>


### Example
Here you see a example yaml file for the _TVL_-Extension *SSE*.
```yaml
---
description: "Definition of the SIMD TargetExtension sse."
vendor: "intel"
extension_name: "sse"
lscpu_flags: ["sse", "sse2", "ssse3", "sse4_1", "sse4_2" ]
includes: ["<immintrin.h>"]
simdT_name: "sse"
simdT_default_size_in_bits: "128"
simdT_register_type_attributes: |-
   __attribute__ ((
      __vector_size__ (
         VectorSizeInBits/sizeof(
            TVL_DEP_TYPE(
               (std::is_integral_v< BaseType >),
               long long,
               TVL_DEP_TYPE(
                  (sizeof( BaseType ) == 4),
                  float,
                  double
               )
            )
         )
      ), 
      __may_alias__, 
      __aligned__(VectorSizeInBits/sizeof(char))
   ))
simdT_register_type: |-
   TVL_DEP_TYPE(
      (std::is_integral_v< BaseType >),
      long long,
      TVL_DEP_TYPE(
         (sizeof( BaseType ) == 4),
         float,
         double
      )
   )
simdT_mask_type: "register_t"
...
```
You may notice `TVL_DEP_TYPE`. This is just macro capturing `std::conditional_t`. 
For every optional parameter the default value provided by the schema will be added on the fly while generating the _TVL_-code. 
As the schema is used for rudimentary type-checking and casting, producing an error if a provided value does not match the specification and can not be casted into the required type, no error is generated for _additional_ values within your yaml files. 
The specified data will be passed into the _JINJA2_ templates and can be accessed from there to obtain maximum flexibility and extensibility.

