To generate the library execute the following example commands (the paths will be updated):
```
fpga_crossgen -v -fPIC ../../lib_rtl/lib_rtl_spec_lzc32.xml --emulation_model ../../lib_rtl/lib_rtl_model_lzc32.cpp --target sycl -o ../../lib_rtl/lib_rtl_lzc32.o
fpga_libtool -v ../../lib_rtl/lib_rtl_lzc32.o --target sycl --create ../../lib_rtl/lib.a
```
