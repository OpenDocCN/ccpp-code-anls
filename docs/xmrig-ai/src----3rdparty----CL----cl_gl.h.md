# `xmrig\src\3rdparty\CL\cl_gl.h`

```
// 版权声明和许可声明
// 定义了__OPENCL_CL_GL_H宏，避免重复包含
// 包含了OpenCL的头文件
// 如果是C++环境，则采用C语言的方式进行编译
// 定义了cl_gl_object_type、cl_gl_texture_info、cl_gl_platform_info和cl_GLsync类型
// 枚举值，表示OpenGL对象类型
#define CL_GL_OBJECT_BUFFER                     0x2000
// 定义 OpenGL 对象类型常量
#define CL_GL_OBJECT_TEXTURE2D                  0x2001
#define CL_GL_OBJECT_TEXTURE3D                  0x2002
#define CL_GL_OBJECT_RENDERBUFFER               0x2003
#ifdef CL_VERSION_1_2
#define CL_GL_OBJECT_TEXTURE2D_ARRAY            0x200E
#define CL_GL_OBJECT_TEXTURE1D                  0x200F
#define CL_GL_OBJECT_TEXTURE1D_ARRAY            0x2010
#define CL_GL_OBJECT_TEXTURE_BUFFER             0x2011
#endif

/* cl_gl_texture_info           */
// 定义 OpenGL 纹理信息常量
#define CL_GL_TEXTURE_TARGET                    0x2004
#define CL_GL_MIPMAP_LEVEL                      0x2005
#ifdef CL_VERSION_1_2
#define CL_GL_NUM_SAMPLES                       0x2012
#endif

// 创建一个从 OpenGL 缓冲区对象中创建 OpenCL 内存对象的函数声明
extern CL_API_ENTRY cl_mem CL_API_CALL
clCreateFromGLBuffer(cl_context     context,
                     cl_mem_flags   flags,
                     cl_GLuint      bufobj,
                     cl_int *       errcode_ret) CL_API_SUFFIX__VERSION_1_0;

#ifdef CL_VERSION_1_2
// 创建一个从 OpenGL 纹理对象中创建 OpenCL 内存对象的函数声明
extern CL_API_ENTRY cl_mem CL_API_CALL
clCreateFromGLTexture(cl_context      context,
                      cl_mem_flags    flags,
                      cl_GLenum       target,
                      cl_GLint        miplevel,
                      cl_GLuint       texture,
                      cl_int *        errcode_ret) CL_API_SUFFIX__VERSION_1_2;
#endif

// 创建一个从 OpenGL 渲染缓冲区对象中创建 OpenCL 内存对象的函数声明
extern CL_API_ENTRY cl_mem CL_API_CALL
clCreateFromGLRenderbuffer(cl_context   context,
                           cl_mem_flags flags,
                           cl_GLuint    renderbuffer,
                           cl_int *     errcode_ret) CL_API_SUFFIX__VERSION_1_0;

// 获取 OpenGL 对象信息的函数声明
extern CL_API_ENTRY cl_int CL_API_CALL
clGetGLObjectInfo(cl_mem                memobj,
                  cl_gl_object_type *   gl_object_type,
                  cl_GLuint *           gl_object_name) CL_API_SUFFIX__VERSION_1_0;

extern CL_API_ENTRY cl_int CL_API_CALL
# 获取 OpenGL 纹理对象的信息
clGetGLTextureInfo(cl_mem               memobj,  # OpenGL 纹理对象
                   cl_gl_texture_info   param_name,  # 参数名
                   size_t               param_value_size,  # 参数值大小
                   void *               param_value,  # 参数值
                   size_t *             param_value_size_ret) CL_API_SUFFIX__VERSION_1_0;  # 返回参数值大小

# 将 OpenGL 对象关联到 OpenCL 命令队列
extern CL_API_ENTRY cl_int CL_API_CALL
clEnqueueAcquireGLObjects(cl_command_queue      command_queue,  # 命令队列
                          cl_uint               num_objects,  # 对象数量
                          const cl_mem *        mem_objects,  # 内存对象数组
                          cl_uint               num_events_in_wait_list,  # 等待事件列表中的事件数量
                          const cl_event *      event_wait_list,  # 等待事件列表
                          cl_event *            event) CL_API_SUFFIX__VERSION_1_0;  # 事件

# 释放与 OpenCL 命令队列关联的 OpenGL 对象
extern CL_API_ENTRY cl_int CL_API_CALL
clEnqueueReleaseGLObjects(cl_command_queue      command_queue,  # 命令队列
                          cl_uint               num_objects,  # 对象数量
                          const cl_mem *        mem_objects,  # 内存对象数组
                          cl_uint               num_events_in_wait_list,  # 等待事件列表中的事件数量
                          const cl_event *      event_wait_list,  # 等待事件列表
                          cl_event *            event) CL_API_SUFFIX__VERSION_1_0;  # 事件

# 创建 OpenCL 内存对象，关联到 OpenGL 2D 纹理对象
extern CL_API_ENTRY CL_EXT_PREFIX__VERSION_1_1_DEPRECATED cl_mem CL_API_CALL
clCreateFromGLTexture2D(cl_context      context,  # 上下文
                        cl_mem_flags    flags,  # 内存标志
                        cl_GLenum       target,  # 目标
                        cl_GLint        miplevel,  # Mip 层级
                        cl_GLuint       texture,  # 纹理
                        cl_int *        errcode_ret) CL_EXT_SUFFIX__VERSION_1_1_DEPRECATED;  # 错误码
# 定义一个函数，用于创建一个与 OpenGL 纹理相关的 3D 内存对象
clCreateFromGLTexture3D(cl_context      context,
                        cl_mem_flags    flags,
                        cl_GLenum       target,
                        cl_GLint        miplevel,
                        cl_GLuint       texture,
                        cl_int *        errcode_ret) CL_EXT_SUFFIX__VERSION_1_1_DEPRECATED;

/* cl_khr_gl_sharing extension  */

# 定义一个名为 cl_khr_gl_sharing 的扩展，数值为 1
#define cl_khr_gl_sharing 1

# 定义一个名为 cl_gl_context_info 的类型为 cl_uint
typedef cl_uint     cl_gl_context_info;

# 定义额外的错误代码，表示无效的 OpenGL 共享组引用
#define CL_INVALID_GL_SHAREGROUP_REFERENCE_KHR  -1000

# 定义 cl_gl_context_info 的值
#define CL_CURRENT_DEVICE_FOR_GL_CONTEXT_KHR    0x2006
#define CL_DEVICES_FOR_GL_CONTEXT_KHR           0x2007

# 定义额外的 cl_context_properties，表示 OpenGL 上下文
#define CL_GL_CONTEXT_KHR                       0x2008
#define CL_EGL_DISPLAY_KHR                      0x2009
#define CL_GLX_DISPLAY_KHR                      0x200A
#define CL_WGL_HDC_KHR                          0x200B
#define CL_CGL_SHAREGROUP_KHR                   0x200C

# 声明一个名为 clGetGLContextInfoKHR 的函数，用于获取与 OpenGL 上下文相关的信息
extern CL_API_ENTRY cl_int CL_API_CALL
clGetGLContextInfoKHR(const cl_context_properties * properties,
                      cl_gl_context_info            param_name,
                      size_t                        param_value_size,
                      void *                        param_value,
                      size_t *                      param_value_size_ret) CL_API_SUFFIX__VERSION_1_0;

# 定义一个名为 clGetGLContextInfoKHR_fn 的函数指针类型，用于获取与 OpenGL 上下文相关的信息
typedef CL_API_ENTRY cl_int (CL_API_CALL *clGetGLContextInfoKHR_fn)(
    const cl_context_properties * properties,
    cl_gl_context_info            param_name,
    size_t                        param_value_size,
    void *                        param_value,
    size_t *                      param_value_size_ret);

# 结束 C++ 的声明
#ifdef __cplusplus
}
#endif

# 结束条件编译
#endif  /* __OPENCL_CL_GL_H */
```