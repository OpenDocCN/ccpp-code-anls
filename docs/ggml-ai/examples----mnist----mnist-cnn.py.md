# `ggml\examples\mnist\mnist-cnn.py`

```cpp
#!/usr/bin/env python3
# 指定使用 Python3 解释器

import sys
# 导入 sys 模块，用于访问与 Python 解释器交互的变量和函数
import gguf
# 导入 gguf 模块
import numpy as np
# 导入 numpy 模块，并使用别名 np
from tensorflow import keras
# 从 tensorflow 模块中导入 keras 子模块
from tensorflow.keras import layers
# 从 tensorflow.keras 模块中导入 layers 子模块

def train(model_name):
    # 定义训练函数，接受模型名称作为参数

    # Model / data parameters
    # 模型/数据参数
    num_classes = 10
    # 类别数量为 10
    input_shape = (28, 28, 1)
    # 输入形状为 (28, 28, 1)

    # Load the data and split it between train and test sets
    # 加载数据并将其分为训练集和测试集
    (x_train, y_train), (x_test, y_test) = keras.datasets.mnist.load_data()

    # Scale images to the [0, 1] range
    # 将图像缩放到 [0, 1] 范围内
    x_train = x_train.astype("float32") / 255
    x_test = x_test.astype("float32") / 255
    # Make sure images have shape (28, 28, 1)
    # 确保图像的形状为 (28, 28, 1)
    x_train = np.expand_dims(x_train, -1)
    x_test = np.expand_dims(x_test, -1)
    print("x_train shape:", x_train.shape)
    print(x_train.shape[0], "train samples")
    print(x_test.shape[0], "test samples")

    # convert class vectors to binary class matrices
    # 将类向量转换为二进制类矩阵
    y_train = keras.utils.to_categorical(y_train, num_classes)
    y_test = keras.utils.to_categorical(y_test, num_classes)

    model = keras.Sequential(
        [
            keras.Input(shape=input_shape),
            layers.Conv2D(32, kernel_size=(3, 3), activation="relu"),
            layers.MaxPooling2D(pool_size=(2, 2)),
            layers.Conv2D(64, kernel_size=(3, 3), activation="relu"),
            layers.MaxPooling2D(pool_size=(2, 2)),
            layers.Flatten(),
            layers.Dropout(0.5),
            layers.Dense(num_classes, activation="softmax"),
        ]
    )

    model.summary()
    batch_size = 128
    epochs = 15
    model.compile(loss="categorical_crossentropy", optimizer="adam", metrics=["accuracy"])
    model.fit(x_train, y_train, batch_size=batch_size, epochs=epochs, validation_split=0.1)

    score = model.evaluate(x_test, y_test, verbose=0)
    print("Test loss:", score[0])
    print("Test accuracy:", score[1])
    model.save(model_name)
    print("Keras model saved to '" + model_name + "'")

def convert(model_name):
    # 定义转换函数，接受模型名称作为参数
    model = keras.models.load_model(model_name)
    # 加载模型
    gguf_model_name = model_name + ".gguf"
    # 为模型名称添加后缀 ".gguf"
    # 创建一个GGUFWriter对象，用于将模型转换为GGUF格式并保存
    gguf_writer = gguf.GGUFWriter(gguf_model_name, "mnist-cnn")

    # 获取第一个卷积层的权重，并将其格式转换为GGUF所需的格式，然后添加到GGUFWriter对象中
    kernel1 = model.layers[0].weights[0].numpy()
    kernel1 = np.moveaxis(kernel1, [2,3], [0,1])
    kernel1 = kernel1.astype(np.float16)
    gguf_writer.add_tensor("kernel1", kernel1, raw_shape=(32, 1, 3, 3))

    # 获取第一个卷积层的偏置，并将其格式转换为GGUF所需的格式，然后添加到GGUFWriter对象中
    bias1 = model.layers[0].weights[1].numpy()
    bias1 = np.repeat(bias1, 26*26)
    gguf_writer.add_tensor("bias1", bias1, raw_shape=(1, 32, 26, 26))

    # 获取第二个卷积层的权重，并将其格式转换为GGUF所需的格式，然后添加到GGUFWriter对象中
    kernel2 = model.layers[2].weights[0].numpy()
    kernel2 = np.moveaxis(kernel2, [0,1,2,3], [2,3,1,0])
    kernel2 = kernel2.astype(np.float16)
    gguf_writer.add_tensor("kernel2", kernel2, raw_shape=(64, 32, 3, 3))

    # 获取第二个卷积层的偏置，并将其格式转换为GGUF所需的格式，然后添加到GGUFWriter对象中
    bias2 = model.layers[2].weights[1].numpy()
    bias2 = np.repeat(bias2, 11*11)
    gguf_writer.add_tensor("bias2", bias2, raw_shape=(1, 64, 11, 11))

    # 获取最后一个全连接层的权重，并将其格式转换为GGUF所需的格式，然后添加到GGUFWriter对象中
    dense_w = model.layers[-1].weights[0].numpy()
    dense_w = dense_w.transpose()
    gguf_writer.add_tensor("dense_w", dense_w, raw_shape=(10, 1600))

    # 获取最后一个全连接层的偏置，并将其格式转换为GGUF所需的格式，然后添加到GGUFWriter对象中
    dense_b = model.layers[-1].weights[1].numpy()
    gguf_writer.add_tensor("dense_b", dense_b)

    # 将GGUF文件头信息写入文件
    gguf_writer.write_header_to_file()
    # 将键值对数据写入文件
    gguf_writer.write_kv_data_to_file()
    # 将张量数据写入文件
    gguf_writer.write_tensors_to_file()
    # 关闭GGUFWriter对象
    gguf_writer.close()
    # 打印模型转换并保存的信息
    print("Model converted and saved to '{}'".format(gguf_model_name)
# 如果当前脚本被直接执行，则执行以下代码
if __name__ == '__main__':
    # 如果命令行参数少于3个，则打印提示信息并退出程序
    if len(sys.argv) < 3:
        print("Usage: %s <train|convert> <model_name>".format(sys.argv[0]))
        sys.exit(1)
    # 如果第一个命令行参数为'train'，则调用train函数并传入第二个参数作为模型名称
    if sys.argv[1] == 'train':
        train(sys.argv[2])
    # 如果第一个命令行参数为'convert'，则调用convert函数并传入第二个参数作为模型名称
    elif sys.argv[1] == 'convert':
        convert(sys.argv[2])
    # 如果命令行参数不是'train'也不是'convert'，则打印提示信息并退出程序
    else:
        print("Usage: %s <train|convert> <model_name>".format(sys.argv[0]))
        sys.exit(1)
```