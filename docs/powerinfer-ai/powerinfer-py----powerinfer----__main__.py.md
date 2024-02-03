# `PowerInfer\powerinfer-py\powerinfer\__main__.py`

```cpp
# 导入命令行参数解析模块
import argparse

# 导入求解器模块中的 solve_gpu_split 函数
from .solver import solve_gpu_split
# 导入导出分割模块中的 export_split 函数
from .export_split import export_split

# 如果当前脚本为主程序
if __name__ == "__main__":
    
    # 设置命令行参数
    parser = argparse.ArgumentParser(description='Optimize neuron activation based on VRAM capacity and other parameters.')
    # 添加命令行参数：激活数据的路径
    parser.add_argument('--activation', type=str, required=True, help='Path to the directory containing activation data.')
    # 添加命令行参数：网络中神经元的总数
    parser.add_argument('--neuron', type=int, default=8192*4, help='Total number of neurons in the network.')
    # 添加命令行参数：模型的总 VRAM 容量
    parser.add_argument('--capacity', type=int, default=int(8192*4*32*0.1), help='Total VRAM capacity for the model.')
    # 添加命令行参数：神经网络中的总层数
    parser.add_argument('--layer', type=int, default=59, help='Total number of layers in the neural network.')
    # 添加命令行参数：可用于分割的总 VRAM 容量（字节）
    parser.add_argument('--vram-capacity', type=int, help='Total VRAM capacity (Bytes) available for splitting')
    # 添加命令行参数：处理的批次大小
    parser.add_argument('--batch', type=int, default=256, help='Batch size for processing.')
    # 添加命令行参数：分割层跨多个 GPU 的阈值
    parser.add_argument('--threshold', type=int, default=0, help='Threshold for splitting a layer across multiple GPUs.')
    # 添加命令行参数：输出 pickle 文件的路径
    parser.add_argument('--output', type=str, required=True, help='File path for the output pickle file.')

    # 解析命令行参数
    args = parser.parse_args()

    # 打印解析后的参数
    print("solver args:", args)

    # 调用 solve_gpu_split 函数求解
    solved = solve_gpu_split(
        activation_path=args.activation,
        neuron=args.neuron,
        capacity=args.capacity,
        layer=args.layer,
        batch=args.batch,
        threshold=args.threshold,
    )

    # 打印求解结果和总神经元数
    print(f"solved: {solved}, total neurons: {sum(solved)}")

    # 调用 export_split 函数导出分割结果
    export_split(
        activations_path=args.activation,
        output_path=args.output,
        solved_list=solved,
        vram_capacity=args.vram_capacity
    )

    # 打印导出结果的路径
    print(f"Exported to {args.output}")
```