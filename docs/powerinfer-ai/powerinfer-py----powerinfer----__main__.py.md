# `PowerInfer\powerinfer-py\powerinfer\__main__.py`

```
# 导入必要的模块
import argparse  # 用于解析命令行参数
from .solver import solve_gpu_split  # 从solver模块中导入solve_gpu_split函数
from .export_split import export_split  # 从export_split模块中导入export_split函数

# 如果该脚本被直接执行
if __name__ == "__main__":
    
    # 设置命令行参数
    parser = argparse.ArgumentParser(description='Optimize neuron activation based on VRAM capacity and other parameters.')  # 创建一个ArgumentParser对象，设置脚本的描述信息
    parser.add_argument('--activation', type=str, required=True, help='Path to the directory containing activation data.')  # 添加一个命令行参数，指定激活数据的路径
    parser.add_argument('--neuron', type=int, default=8192*4, help='Total number of neurons in the network.')  # 添加一个命令行参数，指定网络中的神经元总数
    parser.add_argument('--capacity', type=int, default=int(8192*4*32*0.1), help='Total VRAM capacity for the model.')  # 添加一个命令行参数，指定模型的总VRAM容量
    parser.add_argument('--layer', type=int, default=59, help='Total number of layers in the neural network.')  # 添加一个命令行参数，指定神经网络中的总层数
    parser.add_argument('--vram-capacity', type=int, help='Total VRAM capacity (Bytes) available for splitting')  # 添加一个命令行参数，指定可用于分割的总VRAM容量（字节）
    parser.add_argument('--batch', type=int, default=256, help='Batch size for processing.')  # 添加一个命令行参数，指定处理的批次大小
    parser.add_argument('--threshold', type=int, default=0, help='Threshold for splitting a layer across multiple GPUs.')  # 添加一个命令行参数，指定跨多个GPU分割层的阈值
    parser.add_argument('--output', type=str, required=True, help='File path for the output pickle file.')  # 添加一个命令行参数，指定输出pickle文件的路径
# 解析命令行参数
args = parser.parse_args()

# 打印解析后的参数
print("solver args:", args)

# 使用 GPU 解决分割问题，传入各种参数
solved = solve_gpu_split(
    activation_path=args.activation,
    neuron=args.neuron,
    capacity=args.capacity,
    layer=args.layer,
    batch=args.batch,
    threshold=args.threshold,
)

# 打印解决结果和总神经元数
print(f"solved: {solved}, total neurons: {sum(solved)}")

# 导出分割结果
export_split(
    activations_path=args.activation,
    output_path=args.output,
    solved_list=solved,
    vram_capacity=args.vram_capacity
)
    # 打印输出信息，显示导出的文件路径
    print(f"Exported to {args.output}")
```