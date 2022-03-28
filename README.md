# Template Vector Library - Primitive Data

---

This repository contains all relevant files for the [TVL Generator](https://github.com/db-tu-dresden/TVLGen/tree/main).
All information are provided using *.yaml files. A schema listing all available keys and their description can be found [here](https://github.com/db-tu-dresden/TVLGen/blob/main/config/data/tvl_generator_schema.yaml). 

There are three fundamental entries:
- [Extensions](#extension)
- [Primitives](#primitive)
- [Primitive Classes](#primitive-class)

## Extension
An extension encapsulates all relevant information for a specific vector extension.

We decided to limit the extensions to their corresponding supersets, e.g., (SSE, SSE2, SSSE3, SSE4.1, SSE4.2) are all represented by the extension `sse`.
This reduces the overall number of specified extensions and improves the comprehensibility. 
To include all relevant subsets of an extension, the corresponding lscpu entries has to be included in the extension description.

There is **one** yaml file per extension. All extension files are placed within [./extensions](https://github.com/db-tu-dresden/TVLPrimitiveData/tree/main/extensions). 
Note that there are several subdirectories for *simd* (and *simt*) and one subdirectory for every vendor below those (e.g., intel).  
Every extension yaml file consists of a single yaml object. 

An extension has the following keys (note that this list _may not be_ completed):
<details open>
<summary><b>Required Fields</b></summary>

| Key Name                    |    Type     | Description                                                               |         Example          | Note                                                                                                               |
|-----------------------------|:-----------:|---------------------------------------------------------------------------|:------------------------:|--------------------------------------------------------------------------------------------------------------------|
| `vendor`                      |    `str`    | The vendor name.                                                          |         `"intel"`          |                                                                                                                    |
| `extension_name`              |    `str`    | Name of the specific extension (used as filename).                        |          `"sse"`           |                                                                                                                    |
| `lscpu_flags`                 | `List[str]` | List of extension specific flags (which may be exposed by lscpu).         | `["sse", "sse2", "ssse3"]` |                                                                                                                    |
| `simdT_name`                  |    `str`    | Name of the extension which will be used inside the **_TVL_**.                  |          `"sse"`           |                                                                                                                    |
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


### Example
Here you see a example yaml file for the **_TVL_**-Extension *SSE*.
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
For every optional parameter the default value provided by the schema will be added on the fly while generating the **_TVL_**-code. 
As the schema is used for rudimentary type-checking and casting, producing an error if a provided value does not match the specification and can not be casted into the required type, no error is generated for _additional_ values within your yaml files. 
The specified data will be passed into the _JINJA2_ templates and can be accessed from there to obtain maximum flexibility and extensibility.

## Primitive


A primitive is the **_TVL_**-equivalent to an SIMD-intrinsic. Primitives are organized in [Primitive Classes](#primitive-class). 

For every primitive class a single yaml file exists, containing all related primitives.
Every primitive in **_TVL_** exists as a fully templated function (_Primitive Declaration_), which calls the static member function of the associated _Primitive Definition_ struct.
The introduced indirection allow partial specialization of primitivies (see [Example of a generated _TVL_-primitive add](#example-of-a-generated-tvl-primitive-add)).

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
| `simdT_name`                  |    `str`    | Name of the extension which will be used inside the **_TVL_**.                  |          `"sse"`           |                                                                                                                    |
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
Here you see a example yaml file for the **_TVL_**-Extension *SSE*.
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
For every optional parameter the default value provided by the schema will be added on the fly while generating the **_TVL_**-code. 
As the schema is used for rudimentary type-checking and casting, producing an error if a provided value does not match the specification and can not be casted into the required type, no error is generated for _additional_ values within your yaml files. 
The specified data will be passed into the _JINJA2_ templates and can be accessed from there to obtain maximum flexibility and extensibility.

