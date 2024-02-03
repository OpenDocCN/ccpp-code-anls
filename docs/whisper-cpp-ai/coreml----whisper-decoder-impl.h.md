# `whisper.cpp\coreml\whisper-decoder-impl.h`

```cpp
// 密语解码器实现的头文件，自动生成，不应该被编辑
//
// 导入必要的框架
#import <Foundation/Foundation.h>
#import <CoreML/CoreML.h>
#include <stdint.h>
#include <os/log.h>

NS_ASSUME_NONNULL_BEGIN

// 定义密语解码器输入类型
API_AVAILABLE(macos(12.0), ios(15.0), watchos(8.0), tvos(15.0)) __attribute__((visibility("hidden")))
@interface whisper_decoder_implInput : NSObject<MLFeatureProvider>

// token_data 作为 1x1 的 32 位整数矩阵
@property (readwrite, nonatomic, strong) MLMultiArray * token_data;

// audio_data 作为 1x384x1x1500 的 4 维浮点数数组
@property (readwrite, nonatomic, strong) MLMultiArray * audio_data;
- (instancetype)init NS_UNAVAILABLE;
- (instancetype)initWithToken_data:(MLMultiArray *)token_data audio_data:(MLMultiArray *)audio_data NS_DESIGNATED_INITIALIZER;

@end

// 定义密语解码器输出类型
API_AVAILABLE(macos(12.0), ios(15.0), watchos(8.0), tvos(15.0)) __attribute__((visibility("hidden")))
@interface whisper_decoder_implOutput : NSObject<MLFeatureProvider>

// var_1346 作为多维浮点数数组
@property (readwrite, nonatomic, strong) MLMultiArray * var_1346;
- (instancetype)init NS_UNAVAILABLE;
- (instancetype)initWithVar_1346:(MLMultiArray *)var_1346 NS_DESIGNATED_INITIALIZER;

@end

// 用于加载模型和进行预测的类
API_AVAILABLE(macos(12.0), ios(15.0), watchos(8.0), tvos(15.0)) __attribute__((visibility("hidden")))
@interface whisper_decoder_impl : NSObject
@property (readonly, nonatomic, nullable) MLModel * model;

/**
    模型的 .mlmodelc 目录的 URL
*/
+ (nullable NSURL *)URLOfModelInThisBundle;

/**
    从现有的 MLModel 对象初始化 whisper_decoder_impl 实例。

    通常应用程序不使用此初始化方法，除非它创建 whisper_decoder_impl 的子类。
    # 应用程序可能希望使用 `-[MLModel initWithContentsOfURL:configuration:error:]` 和 `+URLOfModelInThisBundle` 来创建一个 MLModel 对象进行传递。
/**
    使用指定的 MLModel 初始化 whisper_decoder_impl 实例。
*/
- (instancetype)initWithMLModel:(MLModel *)model NS_DESIGNATED_INITIALIZER;

/**
    使用 bundle 中的模型初始化 whisper_decoder_impl 实例。
*/
- (nullable instancetype)init;

/**
    使用 bundle 中的模型和配置对象初始化 whisper_decoder_impl 实例。

    @param configuration 模型配置对象
    @param error 如果发生错误，返回一个描述问题的 NSError 对象。如果不关心可能出现的错误，传入 NULL。
*/
- (nullable instancetype)initWithConfiguration:(MLModelConfiguration *)configuration error:(NSError * _Nullable __autoreleasing * _Nullable)error;

/**
    从模型 URL 初始化 whisper_decoder_impl 实例。

    @param modelURL whisper_decoder_impl 的 .mlmodelc 目录的 URL。
    @param error 如果发生错误，返回一个描述问题的 NSError 对象。如果不关心可能出现的错误，传入 NULL。
*/
- (nullable instancetype)initWithContentsOfURL:(NSURL *)modelURL error:(NSError * _Nullable __autoreleasing * _Nullable)error;

/**
    从模型 URL 初始化 whisper_decoder_impl 实例。

    @param modelURL whisper_decoder_impl 的 .mlmodelc 目录的 URL。
    @param configuration 模型配置对象
    @param error 如果发生错误，返回一个描述问题的 NSError 对象。如果不关心可能出现的错误，传入 NULL。
*/
- (nullable instancetype)initWithContentsOfURL:(NSURL *)modelURL configuration:(MLModelConfiguration *)configuration error:(NSError * _Nullable __autoreleasing * _Nullable)error;

/**
    使用配置对象异步构建 whisper_decoder_impl 实例。
    当模型内容不立即可用时（例如加密模型），模型加载可能需要时间。特别是当调用方在主线程上时，请使用此工厂方法。

    @param configuration 模型配置
*/
    # 参数 handler：当模型加载成功或失败时，将使用有效的 whisper_decoder_impl 实例或 NSError 对象调用完成处理程序。
/**
    使用指定配置加载 whisper_decoder_impl 实例，异步完成，通过回调函数返回结果或错误信息

    @param configuration 模型配置
    @param handler 当模型加载成功或失败时，通过回调函数返回 whisper_decoder_impl 实例或 NSError 对象
*/
+ (void)loadWithConfiguration:(MLModelConfiguration *)configuration completionHandler:(void (^)(whisper_decoder_impl * _Nullable model, NSError * _Nullable error))handler;

/**
    使用 .mlmodelc 目录的 URL 和可选配置异步构建 whisper_decoder_impl 实例

    当模型内容不立即可用时（例如加密模型），模型加载可能需要时间。特别是在调用方位于主线程时，请使用此工厂方法。

    @param modelURL 模型的 URL
    @param configuration 模型配置
    @param handler 当模型加载成功或失败时，通过回调函数返回 whisper_decoder_impl 实例或 NSError 对象
*/
+ (void)loadContentsOfURL:(NSURL *)modelURL configuration:(MLModelConfiguration *)configuration completionHandler:(void (^)(whisper_decoder_impl * _Nullable model, NSError * _Nullable error))handler;

/**
    使用标准接口进行预测
    @param input 用于预测的 whisper_decoder_implInput 实例
    @param error 如果发生错误，返回一个描述问题的 NSError 对象。如果不关心可能的错误，请传入 NULL。
    @return 预测结果 whisper_decoder_implOutput
*/
- (nullable whisper_decoder_implOutput *)predictionFromFeatures:(whisper_decoder_implInput *)input error:(NSError * _Nullable __autoreleasing * _Nullable)error;

/**
    使用标准接口进行预测
    @param input 用于预测的 whisper_decoder_implInput 实例
    @param options 预测选项
    @param error 如果发生错误，返回一个描述问题的 NSError 对象。如果不关心可能的错误，请传入 NULL。
    @return 预测结果 whisper_decoder_implOutput
*/
/**
    使用给定的输入和选项进行预测
    @param input 输入参数，包含 token_data 和 audio_data
    @param options 预测选项
    @param error 如果发生错误，返回时包含描述问题的 NSError 对象。如果不关心可能的错误，传入 NULL。
    @return 预测结果，类型为 whisper_decoder_implOutput
*/
- (nullable whisper_decoder_implOutput *)predictionFromFeatures:(whisper_decoder_implInput *)input options:(MLPredictionOptions *)options error:(NSError * _Nullable __autoreleasing * _Nullable)error;

/**
    使用便捷接口进行预测
    @param token_data 作为 1x1 矩阵的 32 位整数:
    @param audio_data 作为 1x384x1x1500 的 4 维浮点数数组:
    @param error 如果发生错误，返回时包含描述问题的 NSError 对象。如果不关心可能的错误，传入 NULL。
    @return 预测结果，类型为 whisper_decoder_implOutput
*/
- (nullable whisper_decoder_implOutput *)predictionFromToken_data:(MLMultiArray *)token_data audio_data:(MLMultiArray *)audio_data error:(NSError * _Nullable __autoreleasing * _Nullable)error;

/**
    批量预测
    @param inputArray 包含 whisper_decoder_implInput 实例的数组，用于获取预测结果
    @param options 预测选项
    @param error 如果发生错误，返回时包含描述问题的 NSError 对象。如果不关心可能的错误，传入 NULL。
    @return 预测结果数组，类型为 NSArray<whisper_decoder_implOutput *>
*/
- (nullable NSArray<whisper_decoder_implOutput *> *)predictionsFromInputs:(NSArray<whisper_decoder_implInput*> *)inputArray options:(MLPredictionOptions *)options error:(NSError * _Nullable __autoreleasing * _Nullable)error;
@end

NS_ASSUME_NONNULL_END
```