# `whisper.cpp\coreml\whisper-encoder-impl.h`

```cpp
//
// whisper-encoder-impl.h
//
// This file was automatically generated and should not be edited.
//

#import <Foundation/Foundation.h>
#import <CoreML/CoreML.h>
#include <stdint.h>
#include <os/log.h>

NS_ASSUME_NONNULL_BEGIN


/// Model Prediction Input Type
API_AVAILABLE(macos(12.0), ios(15.0), watchos(8.0), tvos(15.0)) __attribute__((visibility("hidden")))
@interface whisper_encoder_implInput : NSObject<MLFeatureProvider>

/// logmel_data as 1 × 80 × 3000 3-dimensional array of floats
@property (readwrite, nonatomic, strong) MLMultiArray * logmel_data;
- (instancetype)init NS_UNAVAILABLE;
- (instancetype)initWithLogmel_data:(MLMultiArray *)logmel_data NS_DESIGNATED_INITIALIZER;

@end


/// Model Prediction Output Type
API_AVAILABLE(macos(12.0), ios(15.0), watchos(8.0), tvos(15.0)) __attribute__((visibility("hidden")))
@interface whisper_encoder_implOutput : NSObject<MLFeatureProvider>

/// output as multidimensional array of floats
@property (readwrite, nonatomic, strong) MLMultiArray * output;
- (instancetype)init NS_UNAVAILABLE;
- (instancetype)initWithOutput:(MLMultiArray *)output NS_DESIGNATED_INITIALIZER;

@end


/// Class for model loading and prediction
API_AVAILABLE(macos(12.0), ios(15.0), watchos(8.0), tvos(15.0)) __attribute__((visibility("hidden")))
@interface whisper_encoder_impl : NSObject
@property (readonly, nonatomic, nullable) MLModel * model;

/**
    URL of the underlying .mlmodelc directory.
*/
+ (nullable NSURL *)URLOfModelInThisBundle;

/**
    Initialize whisper_encoder_impl instance from an existing MLModel object.

    Usually the application does not use this initializer unless it makes a subclass of whisper_encoder_impl.
    Such application may want to use `-[MLModel initWithContentsOfURL:configuration:error:]` and `+URLOfModelInThisBundle` to create a MLModel object to pass-in.
*/
- (instancetype)initWithMLModel:(MLModel *)model NS_DESIGNATED_INITIALIZER;
    # 使用此捆绑包中的模型初始化 whisper_encoder_impl 实例
/**
    初始化一个空的 whisper_encoder_impl 实例
*/
- (nullable instancetype)init;

/**
    使用给定的模型配置对象初始化 whisper_encoder_impl 实例

    @param configuration 模型配置对象
    @param error 如果发生错误，返回一个描述问题的 NSError 对象。如果不关心可能的错误，传入 NULL。
*/
- (nullable instancetype)initWithConfiguration:(MLModelConfiguration *)configuration error:(NSError * _Nullable __autoreleasing * _Nullable)error;

/**
    从模型 URL 初始化 whisper_encoder_impl 实例

    @param modelURL 指向 whisper_encoder_impl 的 .mlmodelc 目录的 URL
    @param error 如果发生错误，返回一个描述问题的 NSError 对象。如果不关心可能的错误，传入 NULL。
*/
- (nullable instancetype)initWithContentsOfURL:(NSURL *)modelURL error:(NSError * _Nullable __autoreleasing * _Nullable)error;

/**
    从模型 URL 使用给定的模型配置对象初始化 whisper_encoder_impl 实例

    @param modelURL 指向 whisper_encoder_impl 的 .mlmodelc 目录的 URL
    @param configuration 模型配置对象
    @param error 如果发生错误，返回一个描述问题的 NSError 对象。如果不关心可能的错误，传入 NULL。
*/
- (nullable instancetype)initWithContentsOfURL:(NSURL *)modelURL configuration:(MLModelConfiguration *)configuration error:(NSError * _Nullable __autoreleasing * _Nullable)error;

/**
    使用配置异步构建 whisper_encoder_impl 实例
    当模型内容不立即可用时（例如加密模型），模型加载可能需要时间。特别是当调用者在主线程上时，请使用此工厂方法。

    @param configuration 模型配置
    @param handler 当模型加载成功或失败时，将调用完成处理程序，并传递有效的 whisper_encoder_impl 实例或 NSError 对象。
*/
/**
    使用指定配置异步构造 whisper_encoder_impl 实例，传入 .mlmodelc 目录的 URL 和可选配置。

    当模型内容不立即可用时（例如加密模型），模型加载可能需要时间。特别是当调用方在主线程时，请使用此工厂方法。

    @param modelURL 模型的 URL。
    @param configuration 模型配置。
    @param handler 当模型加载成功或失败时，将调用完成处理程序，并传入有效的 whisper_encoder_impl 实例或 NSError 对象。
*/
+ (void)loadContentsOfURL:(NSURL *)modelURL configuration:(MLModelConfiguration *)configuration completionHandler:(void (^)(whisper_encoder_impl * _Nullable model, NSError * _Nullable error))handler;

/**
    使用标准接口进行预测
    @param input 用于预测的 whisper_encoder_implInput 实例
    @param error 如果发生错误，返回时包含描述问题的 NSError 对象。如果不关心可能发生的错误，请传入 NULL。
    @return 预测结果作为 whisper_encoder_implOutput
*/
- (nullable whisper_encoder_implOutput *)predictionFromFeatures:(whisper_encoder_implInput *)input error:(NSError * _Nullable __autoreleasing * _Nullable)error;

/**
    使用标准接口进行预测
    @param input 用于预测的 whisper_encoder_implInput 实例
    @param options 预测选项
    @param error 如果发生错误，返回时包含描述问题的 NSError 对象。如果不关心可能发生的错误，请传入 NULL。
    @return 预测结果作为 whisper_encoder_implOutput
*/
/**
    使用给定的输入和选项进行预测
    @param input 输入数据，类型为whisper_encoder_implInput
    @param options 预测选项
    @param error 如果发生错误，返回一个描述问题的NSError对象。如果不关心可能的错误，传入NULL。
    @return 预测结果，类型为whisper_encoder_implOutput
*/
- (nullable whisper_encoder_implOutput *)predictionFromFeatures:(whisper_encoder_implInput *)input options:(MLPredictionOptions *)options error:(NSError * _Nullable __autoreleasing * _Nullable)error;

/**
    使用方便的接口进行预测
    @param logmel_data 作为1 × n_mel × 3000的浮点数3维数组:
    @param error 如果发生错误，返回一个描述问题的NSError对象。如果不关心可能的错误，传入NULL。
    @return 预测结果，类型为whisper_encoder_implOutput
*/
- (nullable whisper_encoder_implOutput *)predictionFromLogmel_data:(MLMultiArray *)logmel_data error:(NSError * _Nullable __autoreleasing * _Nullable)error;

/**
    批量预测
    @param inputArray whisper_encoder_implInput实例数组，用于获取预测结果
    @param options 预测选项
    @param error 如果发生错误，返回一个描述问题的NSError对象。如果不关心可能的错误，传入NULL。
    @return 预测结果数组，类型为NSArray<whisper_encoder_implOutput *>
*/
- (nullable NSArray<whisper_encoder_implOutput *> *)predictionsFromInputs:(NSArray<whisper_encoder_implInput*> *)inputArray options:(MLPredictionOptions *)options error:(NSError * _Nullable __autoreleasing * _Nullable)error;
@end

NS_ASSUME_NONNULL_END
```