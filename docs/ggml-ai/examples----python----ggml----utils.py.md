# `ggml\examples\python\ggml\utils.py`

```cpp
"""
  Common helpers for working with ggml + numpy
"""
# 导入所需的库
from ggml import ffi, lib
from typing import Union, Optional
import numpy as np

# 初始化 ggml 上下文，当指针被垃圾回收时会自动释放
def init(mem_size: int, mem_buffer: ffi.CData = ffi.NULL, no_alloc: bool = False) -> ffi.CData:
    """
      Initialize a ggml context, which will be freed automatically when the pointer is garbage collected.
    """
    # 创建初始化参数结构体
    params = ffi.new('struct ggml_init_params*')
    params.mem_size = mem_size
    params.mem_buffer = mem_buffer
    params.no_alloc = no_alloc
    # 调用 ggml_init 函数初始化上下文，并使用 ffi.gc 进行垃圾回收
    return ffi.gc(lib.ggml_init(params[0]), lib.ggml_free)
 
# 定义 TensorLike 类型
TensorLike = Union[ffi.CData, np.ndarray]

# 复制一个张量的内容到另一个张量，透明地进行任何必要的（解）量化
def copy(from_tensor: TensorLike, to_tensor: TensorLike, allow_requantize: bool = True):
    """
      Copy the contents of one tensor to another, doing any necessary (de/re)quantization transparently.
      Works across numpy & ggml tensors, but they must have the same shape (and be contiguous).

      Parameters
      ----------
      from_tensor : TensorLike
          The tensor to copy from (a numpy array or possibly-quantized ggml tensor)
      to_tensor : TensorLike
          The tensor to copy to (a numpy array or possibly-quantized ggml tensor)
      allow_requantize : bool
          If False, will throw an error if requantization is required (i.e. both from_tensor
          and to_tensor are quantized with different quantization types)
    """
    # 如果源张量和目标张量是同一个对象，则直接返回
    if id(from_tensor) == id(to_tensor):
        return
 
    # 检查源张量和目标张量的布局是否一致
    __expect_same_layout("source", from_tensor, "destination", to_tensor)
    # 检查源张量的形状是否与类型一致
    __check_shape_consistent_with_type(from_tensor)
    # 检查目标张量的形状是否与类型一致
    __check_shape_consistent_with_type(to_tensor)

    # 获取源张量和目标张量的类型
    from_type = __get_type(from_tensor)
    to_type = __get_type(to_tensor)

    # 如果源张量和目标张量的类型相同，则使用 ffi.memmove 进行内存复制
    if from_type == to_type:
        ffi.memmove(__get_data(to_tensor), __get_data(from_tensor), __get_nbytes(from_tensor))
    # 如果不满足以下条件，则触发断言错误
    assert allow_requantize or not lib.ggml_is_quantized(from_type) or not lib.ggml_is_quantized(to_type), \
        f"Requantizing from {__type_name(from_type)} to {__type_name(to_type)} is disabled. Force with allow_requantize=True"
    # 将 from_tensor 中的浮点数值设置到 to_tensor 中
    __set_floats(to_tensor, __get_floats(from_tensor))
def numpy(tensor: ffi.CData, allow_copy: Union[bool, np.ndarray] = False, allow_requantize=False) -> np.ndarray:
    """
      Convert a ggml tensor to a numpy array.
      If the tensor isn't quantized, the returned numpy array will be a view over its data.
 
      If it is quantized (and allow_copy is True), the copy will involve dequantization and the returned array will
      be a copy of the original tensor (any changes to the numpy array won't then be reflected back to the tensor).

      Parameters
      ----------
      tensor : ffi.CData
          The tensor to convert to a numpy array
      allow_copy : bool or np.ndarray
          If False, will throw an error if the tensor is quantized (since dequantization requires extra memory).
          If True, will dequantize the tensor and return a copy of the data in a new float32 numpy array.
          If an np.ndarray, will copy the data into the given array (which must be the same shape as the tensor) when dequantization is needed
      allow_requantize : bool
          If allow_copy is a tensor with a different quantization type than the source tensor, will throw an error unless allow_requantize is True.
    """
    # 获取张量的形状
    shape = __get_shape(tensor)

    # 检查张量是否是量化的
    if lib.ggml_is_quantized(tensor.type):
        # 如果不允许复制，则抛出错误
        if allow_copy == False:
            raise ValueError(f"{__describe(tensor)} is quantized, conversion to numpy requires a copy (pass allow_copy=True; changes to the numpy array won't affect the original).")
        # 如果允许复制，并且 allow_copy 是一个 np.ndarray，则检查形状是否一致，然后将数据复制到给定的数组中
        elif isinstance(allow_copy, np.ndarray):
            __expect_same_layout("source tensor", tensor, "dequantization output tensor", allow_copy)
            destination = allow_copy
        # 如果允许复制，并且 allow_copy 不是一个 np.ndarray，则创建一个形状相同的空数组
        else:
            destination = np.empty(shape, dtype=np.float32)

        # 复制数据到新数组中，并返回新数组
        copy(tensor, destination, allow_requantize=allow_requantize)
        return destination
    # 如果不是特殊类型的张量，则将其类型转换为对应的 numpy 数据类型
    dtype = __type_to_dtype(tensor.type)
    # 如果没有对应的 numpy 数据类型，则抛出 NotImplementedError
    if not dtype:
        raise NotImplementedError(f'Cannot convert {__describe(tensor)} to numpy')

    # 断言张量是连续的，如果不是，则抛出异常
    assert __is_contiguous(tensor), f"Cannot convert {__describe(tensor)} to numpy (support contiguous tensors only)"
    # 计算张量占用的字节数
    nbytes = lib.ggml_nelements(tensor) * lib.ggml_type_size(tensor.type)
    # 从张量的数据指针创建 numpy 数组
    array = np.frombuffer(ffi.buffer(lib.ggml_get_data(tensor), nbytes), dtype=dtype)
    # 设置数组的形状
    array.shape = shape
    # 返回转换后的 numpy 数组
    return array
# 根据类型返回类型名称的函数
def __type_name(type: int) -> str:
    # 调用外部库的函数获取类型名称
    name = lib.ggml_type_name(type)
    # 将返回的字节流解码成 utf-8 字符串，如果为空则返回 None
    return ffi.string(name).decode('utf-8') if name else None

# 定义一组量化类型的集合
__k_quant_types = set([
  lib.GGML_TYPE_Q2_K,
  lib.GGML_TYPE_Q3_K,
  lib.GGML_TYPE_Q4_K,
  lib.GGML_TYPE_Q5_K,
  lib.GGML_TYPE_Q6_K,
  lib.GGML_TYPE_Q8_K,
])

# 定义类型到数据类型的映射字典
__type_to_dtype_dict = {
  lib.GGML_TYPE_I8: np.int8,
  lib.GGML_TYPE_I16: np.int16,
  lib.GGML_TYPE_I32: np.int32,
  lib.GGML_TYPE_F16: np.float16,
  lib.GGML_TYPE_F32: np.float32,
}

# 根据类型返回对应的数据类型，如果不存在则返回 None
def __type_to_dtype(type: int) -> Optional[np.dtype]: return __type_to_dtype_dict.get(type)

# 根据数据类型返回对应的类型
def __dtype_to_type(dtype: np.dtype):
    if dtype == np.float32: return lib.GGML_TYPE_F32
    elif dtype == np.float16: return lib.GGML_TYPE_F16
    elif dtype == np.int32: return lib.GGML_TYPE_I32
    elif dtype == np.int16: return lib.GGML_TYPE_I16
    elif dtype == np.int8: return lib.GGML_TYPE_I8
    else: raise ValueError(f"Unsupported dtype: {dtype}")

# 返回描述张量的字符串
def __describe(tensor: ffi.CType): return f'Tensor[{__type_name(__get_type(tensor))}, {__get_shape(tensor)}]'

# 获取张量的类型
def __get_type(tensor: TensorLike): return __dtype_to_type(tensor.dtype) if isinstance(tensor, np.ndarray) else tensor.type

# 获取张量的形状
def __get_shape(x: TensorLike): return x.shape if isinstance(x, np.ndarray) else tuple([x.ne[i] for i in range(x.n_dims)])

# 获取张量的步长
def __get_strides(x: TensorLike): return x.strides if isinstance(x, np.ndarray) else tuple([x.nb[i] for i in range(x.n_dims)])

# 获取张量的数据
def __get_data(x: TensorLike) -> ffi.CData: return ffi.from_buffer(x) if isinstance(x, np.ndarray) else lib.ggml_get_data(x)

# 获取张量的字节数
def __get_nbytes(tensor: TensorLike): return tensor.nbytes if isinstance(tensor, np.ndarray) else lib.ggml_nbytes(tensor)

# 获取张量的元素个数
def __get_nelements(tensor: TensorLike): return tensor.size if isinstance(tensor, np.ndarray) else lib.ggml_nelements(tensor)

# 检查张量是否是连续的
def __is_contiguous(tensor: TensorLike): return tensor.flags['C_CONTIGUOUS'] if isinstance(tensor, np.ndarray) else lib.ggml_is_contiguous(tensor)

# 获取张量的浮点数数据
def __get_floats(tensor: TensorLike) -> ffi.CData:
    # 从张量中获取数据和数据类型
    data, type = __get_data(tensor), __get_type(tensor)
    # 如果数据类型为 32 位浮点数
    if type == lib.GGML_TYPE_F32:
        # 将数据转换为 32 位浮点数并返回
        return ffi.cast('float*', data)
    else:
        # 获取张量中元素的数量
        nelements = __get_nelements(tensor)
        # 创建一个新的 32 位浮点数数组
        floats = ffi.new('float[]', nelements)
        # 如果数据类型为 16 位浮点数
        if type == lib.GGML_TYPE_F16:
            # 调用库函数将 16 位浮点数转换为 32 位浮点数
            lib.ggml_fp16_to_fp32_row(ffi.cast('uint16_t*', data), floats, nelements)
        # 如果数据类型为量化类型
        elif lib.ggml_is_quantized(type):
            # 获取量化类型的特性
            qtype = lib.ggml_internal_get_type_traits(type)
            # 检查是否支持将该类型转换为浮点数
            assert qtype.to_float, f"Type {__type_name(type)} is not supported by ggml"
            # 调用相应的函数将量化数据转换为浮点数
            qtype.to_float(data, floats, nelements)
        else:
            # 抛出未实现的错误，无法从张量中读取浮点数
            raise NotImplementedError(f'Cannot read floats from {__describe(tensor)}')
        # 返回转换后的浮点数数组
        return floats
# 将浮点数数据写入张量中
def __set_floats(tensor: TensorLike, f32_data: ffi.CData) -> None:
    # 获取张量的数据、类型和字节数
    data, type, nbytes = __get_data(tensor), __get_type(tensor), __get_nbytes(tensor)
    # 如果类型为 lib.GGML_TYPE_F32
    if type == lib.GGML_TYPE_F32:
        # 将 f32_data 中的数据复制到 data 中
        ffi.memmove(data, f32_data, nbytes)
    else:
        # 获取张量的元素数量
        nelements = __get_nelements(tensor)
        # 如果类型为 lib.GGML_TYPE_F16
        if type == lib.GGML_TYPE_F16:
            # 将 f32_data 中的数据转换为半精度浮点数，并存储到 data 中
            lib.ggml_fp32_to_fp16_row(f32_data, ffi.cast('uint16_t*', data), nelements)
        # 如果类型为量化类型
        elif lib.ggml_is_quantized(type):
            # 获取量化类型的特性
            qtype = lib.ggml_internal_get_type_traits(type)
            # 检查是否支持从浮点数转换
            assert qtype.from_float, f"Type {__type_name(type)} is not supported by ggml"
            # 将 f32_data 中的数据转换为量化类型，并存储到 data 中
            qtype.from_float(f32_data, data, nelements)
        else:
            # 抛出未实现的错误
            raise NotImplementedError(f'Cannot write floats to {__describe(tensor)}')

# 检查两个张量的形状是否一致
def __expect_same_layout(name1: str, tensor1: TensorLike, name2: str, tensor2: TensorLike):
    # 获取两个张量的形状
    shape1, shape2 = __get_shape(tensor1), __get_shape(tensor2)
    # 断言形状是否一致
    assert shape1 == shape2, f"Shape mismatch: {name1} has {shape1} but {name2} has {shape2}"
    # 断言张量是否是连续的
    assert __is_contiguous(tensor1) and __is_contiguous(tensor2), f"Only contiguous tensors are supported (got {name1} with strides {__get_strides(tensor1)} and {name2} with strides {__get_strides(tensor2)})"

# 检查张量的形状是否与类型一致
def __check_shape_consistent_with_type(tensor: TensorLike):
    # 获取张量的类型
    type = __get_type(tensor)
    # 如果不是量化类型，则直接返回
    if not lib.ggml_is_quantized(type):
        return
    # 获取张量的形状
    shape = __get_shape(tensor)
    # 获取量化类型的块大小
    block_size = lib.ggml_blck_size(type)
    # 断言是否支持量化
    assert not (block_size == 0 and type in __k_quant_types), f"Can't quantize, native library was not compiled with USE_K_QUANTS!"
    # 断言块大小大于 0
    assert block_size > 0, f"Invalid block size {block_size} for type {__type_name(type)}"
    # 遍历张量的每个维度
    for i, d in enumerate(shape):
        # 断言每个维度能够被块大小整除，以支持量化
        assert d % block_size == 0, f"Dimension {i} of {__describe(tensor)} is not divisible by {block_size}, required for quantization."
```