# `ggml\examples\python\test_tensor.py`

```cpp
# 导入 pytest 模块
import pytest
# 从 pytest 模块中导入 raises 函数
from pytest import raises

# 从 ggml 模块中导入 lib 和 ffi 对象
from ggml import lib, ffi
# 从 ggml.utils 模块中导入 init, copy, numpy 函数
from ggml.utils import init, copy, numpy
# 从 numpy 模块中导入 np 对象
import numpy as np
# 从 numpy.testing 模块中导入 npt 对象
import numpy.testing as npt

# 定义一个 pytest 的 fixture，用于初始化和销毁上下文
@pytest.fixture()
def ctx():
    # 打印 "setup"，表示初始化
    print("setup")
    # 初始化上下文，设置内存大小为 10MB
    yield init(mem_size=10*1024*1024)
    # 打印 "teardown"，表示销毁
    print("teardown")

# 定义一个测试类 TestNumPy
class TestNumPy:
    
    # Single element

    # 测试设置和获取单个 i32 类型数据
    def test_set_get_single_i32(self, ctx):
        # 创建一个 i32 类型的对象 i，值为 42
        i = lib.ggml_new_i32(ctx, 42)
        # 断言获取 i 的第一个元素为 42
        assert lib.ggml_get_i32_1d(i, 0) == 42
        # 断言将 i 转换为 numpy 数组后的值为 [42]，数据类型为 np.int32
        assert numpy(i) == np.array([42], dtype=np.int32)

    # 测试设置和获取单个 f32 类型数据
    def test_set_get_single_f32(self, ctx):
        # 创建一个 f32 类型的对象 i，值为 4.2
        i = lib.ggml_new_f32(ctx, 4.2)
        
        # 设置一个误差范围
        epsilon = 0.000001 # Not sure why so large a difference??
        # 使用 pytest.approx 断言获取 i 的第一个元素为 4.2，误差范围为 epsilon
        pytest.approx(lib.ggml_get_f32_1d(i, 0), 4.2, epsilon)
        # 使用 pytest.approx 断言将 i 转换为 numpy 数组后的值为 [4.2]，数据类型为 np.float32，误差范围为 epsilon
        pytest.approx(numpy(i), np.array([4.2], dtype=np.float32), epsilon)

    # 定义一个私有方法 _test_copy_np_to_ggml，用于测试将 numpy 数组复制到 ggml 对象
    def _test_copy_np_to_ggml(self, a: np.ndarray, t: ffi.CData):
        # 复制原始数组 a，得到 a2
        a2 = a.copy() # Clone original
        # 将数组 a 复制到 ggml 对象 t
        copy(a, t)
        # 使用 npt.assert_array_equal 断言将 ggml 对象 t 转换为 numpy 数组后的值与 a2 相等
        npt.assert_array_equal(numpy(t), a2)

    # I32

    # 测试将 numpy 数组复制到 ggml 对象，1维 i32 类型
    def test_copy_np_to_ggml_1d_i32(self, ctx):
        # 创建一个 1 维 i32 类型的 ggml 对象 t，长度为 10
        t = lib.ggml_new_tensor_1d(ctx, lib.GGML_TYPE_I32, 10)
        # 创建一个 1 维长度为 10 的 np.int32 类型数组 a
        a = np.arange(10, dtype=np.int32)
        # 调用私有方法 _test_copy_np_to_ggml，将数组 a 复制到 ggml 对象 t
        self._test_copy_np_to_ggml(a, t)

    # 测试将 numpy 数组复制到 ggml 对象，2维 i32 类型
    def test_copy_np_to_ggml_2d_i32(self, ctx):
        # 创建一个 2 维 i32 类型的 ggml 对象 t，形状为 (2, 3)
        t = lib.ggml_new_tensor_2d(ctx, lib.GGML_TYPE_I32, 2, 3)
        # 创建一个 2 维形状为 (2, 3) 的 np.int32 类型数组 a
        a = np.arange(2 * 3, dtype=np.int32).reshape((2, 3))
        # 调用私有方法 _test_copy_np_to_ggml，将数组 a 复制到 ggml 对象 t
        self._test_copy_np_to_ggml(a, t)

    # 测试将 numpy 数组复制到 ggml 对象，3维 i32 类型
    def test_copy_np_to_ggml_3d_i32(self, ctx):
        # 创建一个 3 维 i32 类型的 ggml 对象 t，形状为 (2, 3, 4)
        t = lib.ggml_new_tensor_3d(ctx, lib.GGML_TYPE_I32, 2, 3, 4)
        # 创建一个 3 维形状为 (2, 3, 4) 的 np.int32 类型数组 a
        a = np.arange(2 * 3 * 4, dtype=np.int32).reshape((2, 3, 4))
        # 调用私有方法 _test_copy_np_to_ggml，将数组 a 复制到 ggml 对象 t
        self._test_copy_np_to_ggml(a, t)

    # 测试将 numpy 数组复制到 ggml 对象，4维 i32 类型
    def test_copy_np_to_ggml_4d_i32(self, ctx):
        # 创建一个 4 维 i32 类型的 ggml 对象 t，形状为 (2, 3, 4, 5)
        t = lib.ggml_new_tensor_4d(ctx, lib.GGML_TYPE_I32, 2, 3, 4, 5)
        # 创建一个 4 维形状为 (2, 3, 4, 5) 的 np.int32 类型数组 a
        a = np.arange(2 * 3 * 4 * 5, dtype=np.int32).reshape((2, 3, 4, 5))
        # 调用私有方法 _test_copy_np_to_ggml，将数组 a 复制到 ggml 对象 t
        self._test_copy_np_to_ggml(a, t)
    # 定义一个测试函数，用于将 numpy 数组复制到 GGML 的 4 维 int32 类型的张量中
    def test_copy_np_to_ggml_4d_n_i32(self, ctx):
        # 定义一个包含 4 个维度的数组，GGML_MAX_DIMS 为 4，超出会导致崩溃
        dims = [2, 3, 4, 5]
        # 使用 CFFI 创建一个 int64_t 类型的数组，用于存储维度信息
        pdims = ffi.new('int64_t[]', len(dims))
        # 遍历维度数组，将维度信息存储到 pdims 中
        for i, d in enumerate(dims): pdims[i] = d
        # 调用 GGML 库创建一个新的 int32 类型的张量
        t = lib.ggml_new_tensor(ctx, lib.GGML_TYPE_I32, len(dims), pdims)
        # 创建一个与 pdims 维度相同的 int32 类型的 numpy 数组
        a = np.arange(np.prod(dims), dtype=np.int32).reshape(tuple(pdims))
        # 调用测试函数，将 numpy 数组复制到 GGML 张量中
        self._test_copy_np_to_ggml(a, t)

    # F32

    # 定义一个测试函数，用于将 numpy 数组复制到 GGML 的 1 维 float32 类型的张量中
    def test_copy_np_to_ggml_1d_f32(self, ctx):
        # 调用 GGML 库创建一个新的 1 维 float32 类型的张量
        t = lib.ggml_new_tensor_1d(ctx, lib.GGML_TYPE_F32, 10)
        # 创建一个包含 10 个元素的 float32 类型的 numpy 数组
        a = np.arange(10, dtype=np.float32)
        # 调用测试函数，将 numpy 数组复制到 GGML 张量中
        self._test_copy_np_to_ggml(a, t)

    # ...（以下类似）
    # 定义一个测试函数，用于将 numpy 数组中的数据复制到 GGML 2D 浮点数张量中
    def test_copy_np_to_ggml_2d_f16(self, ctx):
        # 创建一个新的 GGML 2D 浮点数张量
        t = lib.ggml_new_tensor_2d(ctx, lib.GGML_TYPE_F16, 2, 3)
        # 创建一个 2x3 的浮点数 numpy 数组
        a = np.arange(2 * 3, dtype=np.float16).reshape((2, 3))
        # 调用内部函数，将 numpy 数组中的数据复制到 GGML 张量中
        self._test_copy_np_to_ggml(a, t)
    
    # 定义一个测试函数，用于将 numpy 数组中的数据复制到 GGML 3D 浮点数张量中
    def test_copy_np_to_ggml_3d_f16(self, ctx):
        # 创建一个新的 GGML 3D 浮点数张量
        t = lib.ggml_new_tensor_3d(ctx, lib.GGML_TYPE_F16, 2, 3, 4)
        # 创建一个 2x3x4 的浮点数 numpy 数组
        a = np.arange(2 * 3 * 4, dtype=np.float16).reshape((2, 3, 4))
        # 调用内部函数，将 numpy 数组中的数据复制到 GGML 张量中
        self._test_copy_np_to_ggml(a, t)
    
    # 定义一个测试函数，用于将 numpy 数组中的数据复制到 GGML 4D 浮点数张量中
    def test_copy_np_to_ggml_4d_f16(self, ctx):
        # 创建一个新的 GGML 4D 浮点数张量
        t = lib.ggml_new_tensor_4d(ctx, lib.GGML_TYPE_F16, 2, 3, 4, 5)
        # 创建一个 2x3x4x5 的浮点数 numpy 数组
        a = np.arange(2 * 3 * 4 * 5, dtype=np.float16).reshape((2, 3, 4, 5))
        # 调用内部函数，将 numpy 数组中的数据复制到 GGML 张量中
        self._test_copy_np_to_ggml(a, t)
    
    # 定义一个测试函数，用于将 numpy 数组中的数据复制到 GGML 4D 浮点数张量中
    def test_copy_np_to_ggml_4d_n_f16(self, ctx):
        # 定义维度数组
        dims = [2, 3, 4, 5] # GGML_MAX_DIMS is 4, going beyond would crash
        # 创建一个新的 GGML 4D 浮点数张量
        pdims = ffi.new('int64_t[]', len(dims))
        for i, d in enumerate(dims): pdims[i] = d
        t = lib.ggml_new_tensor(ctx, lib.GGML_TYPE_F16, len(dims), pdims)
        # 创建一个指定维度的浮点数 numpy 数组
        a = np.arange(np.prod(dims), dtype=np.float16).reshape(tuple(pdims))
        # 调用内部函数，将 numpy 数组中的数据复制到 GGML 张量中
        self._test_copy_np_to_ggml(a, t)
    
    # 测试不匹配的形状
    
    # 定义一个测试函数，用于测试在 1D 张量中复制不匹配形状的数据
    def test_copy_mismatching_shapes_1d(self, ctx):
        # 创建一个新的 GGML 1D 浮点数张量
        t = lib.ggml_new_tensor_1d(ctx, lib.GGML_TYPE_F32, 10)
        # 创建一个 10 个元素的浮点数 numpy 数组
        a = np.arange(10, dtype=np.float32)
        # 将 numpy 数组中的数据复制到 GGML 张量中，形状匹配
        copy(a, t) # OK
        # 将 numpy 数组重新调整为 5x2 的形状
        a = a.reshape((5, 2))
        # 测试复制不匹配形状的数据到 GGML 张量中，预期会引发 AssertionError
        with raises(AssertionError): copy(a, t)
        with raises(AssertionError): copy(t, a)
            
    # 定义一个测试函数，用于测试在 2D 张量中复制不匹配形状的数据
    def test_copy_mismatching_shapes_2d(self, ctx):
        # 创建一个新的 GGML 2D 浮点数张量
        t = lib.ggml_new_tensor_2d(ctx, lib.GGML_TYPE_F32, 2, 3)
        # 创建一个 6 个元素的浮点数 numpy 数组
        a = np.arange(6, dtype=np.float32)
        # 将 numpy 数组中的数据复制到 GGML 张量中，形状匹配
        copy(a.reshape((2, 3)), t) # OK
        # 将 numpy 数组重新调整为 3x2 的形状
        a = a.reshape((3, 2))
        # 测试复制不匹配形状的数据到 GGML 张量中，预期会引发 AssertionError
        with raises(AssertionError): copy(a, t)
        with raises(AssertionError): copy(t, a)
    # 测试在3D张量上复制不匹配的形状
    def test_copy_mismatching_shapes_3d(self, ctx):
        # 创建一个新的3D张量
        t = lib.ggml_new_tensor_3d(ctx, lib.GGML_TYPE_F32, 2, 3, 4)
        # 创建一个24个浮点数的一维数组
        a = np.arange(24, dtype=np.float32)
        # 将一维数组重塑为3D数组，并复制到张量t中
        copy(a.reshape((2, 3, 4)), t) # OK
        
        # 将数组a重塑为不同的3D形状
        a = a.reshape((2, 4, 3))
        # 使用断言检查复制操作是否会引发AssertionError
        with raises(AssertionError): copy(a, t)
        with raises(AssertionError): copy(t, a)

    # 测试在4D张量上复制不匹配的形状
    def test_copy_mismatching_shapes_4d(self, ctx):
        # 创建一个新的4D张量
        t = lib.ggml_new_tensor_4d(ctx, lib.GGML_TYPE_F32, 2, 3, 4, 5)
        # 创建一个包含24*5个浮点数的一维数组
        a = np.arange(24*5, dtype=np.float32)
        # 将一维数组重塑为4D数组，并复制到张量t中
        copy(a.reshape((2, 3, 4, 5)), t) # OK
        
        # 将数组a重塑为不同的4D形状
        a = a.reshape((2, 3, 5, 4))
        # 使用断言检查复制操作是否会引发AssertionError
        with raises(AssertionError): copy(a, t)
        with raises(AssertionError): copy(t, a)

    # 测试在F16到F32的复制
    def test_copy_f16_to_f32(self, ctx):
        # 创建一个新的F32类型的1D张量
        t = lib.ggml_new_tensor_1d(ctx, lib.GGML_TYPE_F32, 1)
        # 创建一个包含单个浮点数的数组，数据类型为F16
        a = np.array([123.45], dtype=np.float16)
        # 将数组a复制到张量t中
        copy(a, t)
        # 使用断言检查复制后的结果是否与期望值接近
        np.testing.assert_allclose(lib.ggml_get_f32_1d(t, 0), 123.45, rtol=1e-3)

    # 测试在F32到F16的复制
    def test_copy_f32_to_f16(self, ctx):
        # 创建一个新的F16类型的1D张量
        t = lib.ggml_new_tensor_1d(ctx, lib.GGML_TYPE_F16, 1)
        # 创建一个包含单个浮点数的数组，数据类型为F32
        a = np.array([123.45], dtype=np.float32)
        # 将数组a复制到张量t中
        copy(a, t)
        # 使用断言检查复制后的结果是否与期望值接近
        np.testing.assert_allclose(lib.ggml_get_f32_1d(t, 0), 123.45, rtol=1e-3)

    # 测试在F16到Q5_K的复制
    def test_copy_f16_to_Q5_K(self, ctx):
        # 创建一个包含256个浮点数的一维数组，数据类型为F16
        n = 256
        t = lib.ggml_new_tensor_1d(ctx, lib.GGML_TYPE_Q5_K, n)
        a = np.arange(n, dtype=np.float16)
        # 将数组a复制到张量t中
        copy(a, t)
        # 使用断言检查复制后的结果是否与期望值接近
        np.testing.assert_allclose(a, numpy(t, allow_copy=True), rtol=0.05)

    # 测试在Q5_K到F16的复制
    def test_copy_Q5_K_to_f16(self, ctx):
        # 创建一个包含256个浮点数的一维数组，数据类型为F32
        n = 256
        t = lib.ggml_new_tensor_1d(ctx, lib.GGML_TYPE_Q5_K, n)
        # 将一维数组复制到张量t中
        copy(np.arange(n, dtype=np.float32), t)
        # 创建一个包含256个浮点数的一维数组，数据类型为F16
        a = np.arange(n, dtype=np.float16)
        # 将张量t复制到数组a中
        copy(t, a)
        # 使用断言检查复制后的结果是否与期望值接近
        np.testing.assert_allclose(a, numpy(t, allow_copy=True), rtol=0.05)
    # 定义一个测试函数，用于测试复制不匹配类型的数据
    def test_copy_i16_f32_mismatching_types(self, ctx):
        # 创建一个一维的浮点型数据类型为 F32 的张量
        t = lib.ggml_new_tensor_1d(ctx, lib.GGML_TYPE_F32, 1)
        # 创建一个一维的整型数据类型为 int16 的数组
        a = np.arange(1, dtype=np.int16)
        # 使用 with 语句检查是否会抛出 NotImplementedError 异常，表示复制操作不支持不匹配类型的数据
        with raises(NotImplementedError): copy(a, t)
        # 使用 with 语句检查是否会抛出 NotImplementedError 异常，表示复制操作不支持不匹配类型的数据
        with raises(NotImplementedError): copy(t, a)
# 定义一个测试类 TestTensorCopy
class TestTensorCopy:

    # 定义测试方法 test_copy_self，参数为 ctx
    def test_copy_self(self, ctx):
        # 创建一个整型张量 t，值为 42
        t = lib.ggml_new_i32(ctx, 42)
        # 复制张量 t 到自身
        copy(t, t)
        # 断言张量 t 的第一个元素的值为 42
        assert lib.ggml_get_i32_1d(t, 0) == 42

    # 定义测试方法 test_copy_1d，参数为 ctx
    def test_copy_1d(self, ctx):
        # 创建两个长度为 10 的单维浮点型张量 t1 和 t2
        t1 = lib.ggml_new_tensor_1d(ctx, lib.GGML_TYPE_F32, 10)
        t2 = lib.ggml_new_tensor_1d(ctx, lib.GGML_TYPE_F32, 10)
        # 创建一个长度为 10 的浮点型数组 a
        a = np.arange(10, dtype=np.float32)
        # 将数组 a 复制到张量 t1
        copy(a, t1)
        # 将张量 t1 复制到张量 t2
        copy(t1, t2)
        # 断言数组 a 与张量 t2 的值在容许误差范围内相等
        assert np.allclose(a, numpy(t2))
        # 断言张量 t1 与张量 t2 的值在容许误差范围内相等
        assert np.allclose(numpy(t1), numpy(t2))

# 定义一个测试类 TestGraph
class TestGraph:

    # 定义测试方法 test_add，参数为 ctx
    def test_add(self, ctx):
        # 定义变量 n 为 256
        n = 256
        # 创建两个长度为 256 的单维浮点型张量 ta 和 tb
        ta = lib.ggml_new_tensor_1d(ctx, lib.GGML_TYPE_F32, n)
        tb = lib.ggml_new_tensor_1d(ctx, lib.GGML_TYPE_F32, n)
        # 创建一个存放 ta 和 tb 相加结果的张量 tsum
        tsum = lib.ggml_add(ctx, ta, tb)
        # 断言 tsum 的类型为浮点型
        assert tsum.type == lib.GGML_TYPE_F32

        # 创建一个指向 ggml_cgraph 结构体的指针 gf
        gf = ffi.new('struct ggml_cgraph*')
        # 构建 tsum 的前向传播计算图
        lib.ggml_build_forward_expand(gf, tsum)

        # 创建一个长度为 n 的浮点型数组 a 和 b
        a = np.arange(0, n, dtype=np.float32)
        b = np.arange(n, 0, -1, dtype=np.float32)
        # 将数组 a 复制到张量 ta
        copy(a, ta)
        # 将数组 b 复制到张量 tb
        copy(b, tb)

        # 使用 ctx 计算 gf 的计算图，迭代次数为 1
        lib.ggml_graph_compute_with_ctx(ctx, gf, 1)

        # 断言 tsum 的值与数组 a 和 b 相加的结果在容许误差范围内相等
        assert np.allclose(numpy(tsum, allow_copy=True), a + b)

# 定义一个测试类 TestQuantization
class TestQuantization:

    # 定义测试方法 test_quantized_add，参数为 ctx
    def test_quantized_add(self, ctx):
        # 定义变量 n 为 256
        n = 256
        # 创建一个长度为 256 的单维 Q5_K 类型张量 ta 和长度为 256 的单维浮点型张量 tb
        ta = lib.ggml_new_tensor_1d(ctx, lib.GGML_TYPE_Q5_K, n)
        tb = lib.ggml_new_tensor_1d(ctx, lib.GGML_TYPE_F32, n)
        # 创建一个存放 ta 和 tb 相加结果的张量 tsum
        tsum = lib.ggml_add(ctx, ta, tb)
        # 断言 tsum 的类型为 Q5_K 类型
        assert tsum.type == lib.GGML_TYPE_Q5_K

        # 创建一个指向 ggml_cgraph 结构体的指针 gf
        gf = ffi.new('struct ggml_cgraph*')
        # 构建 tsum 的前向传播计算图
        lib.ggml_build_forward_expand(gf, tsum)

        # 创建一个长度为 n 的浮点型数组 a 和 b
        a = np.arange(0, n, dtype=np.float32)
        b = np.arange(n, 0, -1, dtype=np.float32)
        # 将数组 a 复制到张量 ta
        copy(a, ta)
        # 将数组 b 复制到张量 tb
        copy(b, tb)

        # 使用 ctx 计算 gf 的计算图，迭代次数为 1
        lib.ggml_graph_compute_with_ctx(ctx, gf, 1)

        # 计算未量化的 a 和 b 相加的结果
        unquantized_sum = a + b
        # 获取张量 tsum 的值
        sum = numpy(tsum, allow_copy=True)

        # 计算未量化结果与张量 tsum 的差的无穷范数
        diff = np.linalg.norm(unquantized_sum - sum, np.inf)
        # 断言差大于 4
        assert diff > 4
        # 断言差小于 5
        assert diff < 5
```