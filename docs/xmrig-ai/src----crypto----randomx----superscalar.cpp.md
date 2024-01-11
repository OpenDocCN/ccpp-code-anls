# `xmrig\src\crypto\randomx\superscalar.cpp`

```
# 版权声明
# 该代码是根据BSD许可证发布的，允许以源代码和二进制形式进行再分发和使用
# 在满足以下条件的情况下：
#   * 源代码的再分发必须保留上述版权声明、此条件列表和以下免责声明
#   * 二进制形式的再分发必须在文档和/或其他提供的材料中复制上述版权声明、此条件列表和以下免责声明
#   * 不得使用版权持有人的名称或贡献者的名称，以支持或推广从本软件派生的产品，除非事先获得特定的书面许可
# 
# 本软件由版权持有人和贡献者“按原样”提供，任何明示或暗示的保证，包括但不限于对适销性和特定用途的适用性的暗示保证，都是不被承认的
# 在任何情况下，版权持有人或贡献者都不对任何直接、间接、偶然、特殊、惩罚性或后果性损害（包括但不限于替代商品或服务的采购、使用、数据或利润损失，或业务中断）承担任何责任，无论是合同、严格责任还是侵权行为的理论，即使事先已被告知此类损害的可能性
# 
# randomx 命名空间
namespace randomx {
    # 判断给定的指令类型是否为乘法指令
    static bool isMultiplication(SuperscalarInstructionType type) {
        # 如果指令类型为IMUL_R、IMULH_R、ISMULH_R或IMUL_RCP，则返回True，否则返回False
        return type == SuperscalarInstructionType::IMUL_R || type == SuperscalarInstructionType::IMULH_R || type == SuperscalarInstructionType::ISMULH_R || type == SuperscalarInstructionType::IMUL_RCP;
    }

    # 执行端口的命名空间，表示uOPs（微操作）只能执行在特定的执行端口上
    namespace ExecutionPort {
        using type = int;
        # 空执行端口
        constexpr type Null = 0;
        # 执行端口P0
        constexpr type P0 = 1;
        # 执行端口P1
        constexpr type P1 = 2;
        # 执行端口P5
        constexpr type P5 = 4;
        # 执行端口P0和P1
        constexpr type P01 = P0 | P1;
        # 执行端口P0和P5
        constexpr type P05 = P0 | P5;
        # 执行端口P0、P1和P5
        constexpr type P015 = P0 | P1 | P5;
    }

    # 宏操作作为x86解码器的输出
    # 通常一个宏操作等于一个x86指令，但有时2个指令会合并成一个宏操作
    # 宏操作可以由1个或2个uOPs组成
    class MacroOp {
    private:
        # 宏操作的名称
        const char* name_;
        # 宏操作的大小
        int size_;
        # 宏操作的延迟
        int latency_;
        # 第一个uOP可以执行的执行端口
        ExecutionPort::type uop1_;
        # 第二个uOP可以执行的执行端口
        ExecutionPort::type uop2_;
        # 是否依赖于其它操作
        bool dependent_ = false;
    };

    # 大小为3字节的宏操作
    const MacroOp MacroOp::Add_rr = MacroOp("add r,r", 3, 1, ExecutionPort::P015);
    const MacroOp MacroOp::Sub_rr = MacroOp("sub r,r", 3, 1, ExecutionPort::P015);
    const MacroOp MacroOp::Xor_rr = MacroOp("xor r,r", 3, 1, ExecutionPort::P015);
    const MacroOp MacroOp::Imul_r = MacroOp("imul r", 3, 4, ExecutionPort::P1, ExecutionPort::P5);
    const MacroOp MacroOp::Mul_r = MacroOp("mul r", 3, 4, ExecutionPort::P1, ExecutionPort::P5);
    const MacroOp MacroOp::Mov_rr = MacroOp("mov r,r", 3);

    # 大小为4字节的宏操作
    const MacroOp MacroOp::Lea_sib = MacroOp("lea r,r+r*s", 4, 1, ExecutionPort::P01);
    const MacroOp MacroOp::Imul_rr = MacroOp("imul r,r", 4, 3, ExecutionPort::P1);
    const MacroOp MacroOp::Ror_ri = MacroOp("ror r,i", 4, 1, ExecutionPort::P05);

    # 大小为7字节的宏操作（可以选择性地用nop填充为8或9字节）
    // 创建一个名为 Add_ri 的 MacroOp 对象，包含指令名称、长度、操作数个数和执行端口信息
    const MacroOp MacroOp::Add_ri = MacroOp("add r,i", 7, 1, ExecutionPort::P015);
    // 创建一个名为 Xor_ri 的 MacroOp 对象，包含指令名称、长度、操作数个数和执行端口信息
    const MacroOp MacroOp::Xor_ri = MacroOp("xor r,i", 7, 1, ExecutionPort::P015);

    // 创建一个名为 Mov_ri64 的 MacroOp 对象，包含指令名称、长度、操作数个数和执行端口信息
    // 大小为 10 字节
    const MacroOp MacroOp::Mov_ri64 = MacroOp("mov rax,i64", 10, 1, ExecutionPort::P015);

    // 创建一个名为 Ror_rcl 的 MacroOp 对象，包含指令名称、长度、操作数个数和执行端口信息
    // 但是标记为未使用
    const MacroOp MacroOp::Ror_rcl = MacroOp("ror r,cl", 3, 1, ExecutionPort::P0, ExecutionPort::P5);
    // 创建一个名为 Xor_self 的 MacroOp 对象，包含指令名称和长度
    const MacroOp MacroOp::Xor_self = MacroOp("xor rcx,rcx", 3);
    // 创建一个名为 Cmp_ri 的 MacroOp 对象，包含指令名称、长度、操作数个数和执行端口信息
    const MacroOp MacroOp::Cmp_ri = MacroOp("cmp r,i", 7, 1, ExecutionPort::P015);
    // 创建一个名为 Setcc_r 的 MacroOp 对象，包含指令名称、长度、操作数个数和执行端口信息
    const MacroOp MacroOp::Setcc_r = MacroOp("setcc cl", 3, 1, ExecutionPort::P05);
    // 创建一个名为 TestJz_fused 的 MacroOp 对象，包含指令名称、长度、操作数个数和执行端口信息
    const MacroOp MacroOp::TestJz_fused = MacroOp("testjz r,i", 13, 0, ExecutionPort::P5);

    // 创建一个名为 IMULH_R_ops_array 的 MacroOp 数组，包含多个 MacroOp 对象
    const MacroOp IMULH_R_ops_array[] = { MacroOp::Mov_rr, MacroOp::Mul_r, MacroOp::Mov_rr };
    // 创建一个名为 ISMULH_R_ops_array 的 MacroOp 数组，包含多个 MacroOp 对象
    const MacroOp ISMULH_R_ops_array[] = { MacroOp::Mov_rr, MacroOp::Imul_r, MacroOp::Mov_rr };
    // 创建一个名为 IMUL_RCP_ops_array 的 MacroOp 数组，包含多个 MacroOp 对象
    const MacroOp IMUL_RCP_ops_array[] = { MacroOp::Mov_ri64, MacroOp(MacroOp::Imul_rr, true) };

    // 定义一个名为 SuperscalarInstructionInfo 的类
    class SuperscalarInstructionInfo {
    public:
        // 返回指令的名称
        const char* getName() const {
            return name_;
        }
        // 返回指令的操作数数量
        int getSize() const {
            return ops_.size();
        }
        // 判断指令是否为简单指令（操作数数量为1）
        bool isSimple() const {
            return getSize() == 1;
        }
        // 返回指令的延迟时间
        int getLatency() const {
            return latency_;
        }
        // 返回指定索引位置的操作数
        const MacroOp& getOp(int index) const {
            return ops_[index];
        }
        // 返回指令的类型
        SuperscalarInstructionType getType() const {
            return type_;
        }
        // 返回结果操作数的索引
        int getResultOp() const {
            return resultOp_;
        }
        // 返回目标操作数的索引
        int getDstOp() const {
            return dstOp_;
        }
        // 返回源操作数的索引
        int getSrcOp() const {
            return srcOp_;
        }
        // 定义一系列静态常量，表示不同的超标量指令
        static const SuperscalarInstructionInfo ISUB_R;
        static const SuperscalarInstructionInfo IXOR_R;
        static const SuperscalarInstructionInfo IADD_RS;
        static const SuperscalarInstructionInfo IMUL_R;
        static const SuperscalarInstructionInfo IROR_C;
        static const SuperscalarInstructionInfo IADD_C7;
        static const SuperscalarInstructionInfo IXOR_C7;
        static const SuperscalarInstructionInfo IADD_C8;
        static const SuperscalarInstructionInfo IXOR_C8;
        static const SuperscalarInstructionInfo IADD_C9;
        static const SuperscalarInstructionInfo IXOR_C9;
        static const SuperscalarInstructionInfo IMULH_R;
        static const SuperscalarInstructionInfo ISMULH_R;
        static const SuperscalarInstructionInfo IMUL_RCP;
        static const SuperscalarInstructionInfo NOP;
    // 私有成员变量，存储指令名称、类型、宏操作、延迟以及操作数
    private:
        const char* name_;  // 指令名称
        SuperscalarInstructionType type_;  // 指令类型
        std::vector<MacroOp> ops_;  // 宏操作数组
        int latency_;  // 延迟
        int resultOp_ = 0;  // 结果操作数
        int dstOp_ = 0;  // 目标操作数
        int srcOp_ = 0;  // 源操作数

        // 构造函数，初始化指令名称和类型
        SuperscalarInstructionInfo(const char* name)
            : name_(name), type_(SuperscalarInstructionType::INVALID), latency_(0) {}
        
        // 构造函数，初始化指令名称、类型、宏操作、源操作数
        SuperscalarInstructionInfo(const char* name, SuperscalarInstructionType type, const MacroOp& op, int srcOp)
            : name_(name), type_(type), latency_(op.getLatency()), srcOp_(srcOp) {
            ops_.push_back(MacroOp(op));
        }
        
        // 模板构造函数，初始化指令名称、类型、宏操作数组、结果操作数、目标操作数、源操作数
        template <size_t N>
        SuperscalarInstructionInfo(const char* name, SuperscalarInstructionType type, const MacroOp(&arr)[N], int resultOp, int dstOp, int srcOp)
            : name_(name), type_(type), latency_(0), resultOp_(resultOp), dstOp_(dstOp), srcOp_(srcOp) {
            for (unsigned i = 0; i < N; ++i) {
                ops_.push_back(MacroOp(arr[i]));
                latency_ += ops_.back().getLatency();
            }
            static_assert(N > 1, "Invalid array size");
        }
    };

    // 初始化静态成员变量，指令信息对象
    const SuperscalarInstructionInfo SuperscalarInstructionInfo::ISUB_R = SuperscalarInstructionInfo("ISUB_R", SuperscalarInstructionType::ISUB_R, MacroOp::Sub_rr, 0);
    const SuperscalarInstructionInfo SuperscalarInstructionInfo::IXOR_R = SuperscalarInstructionInfo("IXOR_R", SuperscalarInstructionType::IXOR_R, MacroOp::Xor_rr, 0);
    const SuperscalarInstructionInfo SuperscalarInstructionInfo::IADD_RS = SuperscalarInstructionInfo("IADD_RS", SuperscalarInstructionType::IADD_RS, MacroOp::Lea_sib, 0);
    const SuperscalarInstructionInfo SuperscalarInstructionInfo::IMUL_R = SuperscalarInstructionInfo("IMUL_R", SuperscalarInstructionType::IMUL_R, MacroOp::Imul_rr, 0);
    const SuperscalarInstructionInfo SuperscalarInstructionInfo::IROR_C = SuperscalarInstructionInfo("IROR_C", SuperscalarInstructionType::IROR_C, MacroOp::Ror_ri, -1);
    // 创建一个名为IADD_C7的SuperscalarInstructionInfo对象，类型为IADD_C7，宏操作为Add_ri，参数为-1
    const SuperscalarInstructionInfo SuperscalarInstructionInfo::IADD_C7 = SuperscalarInstructionInfo("IADD_C7", SuperscalarInstructionType::IADD_C7, MacroOp::Add_ri, -1);
    // 创建一个名为IXOR_C7的SuperscalarInstructionInfo对象，类型为IXOR_C7，宏操作为Xor_ri，参数为-1
    const SuperscalarInstructionInfo SuperscalarInstructionInfo::IXOR_C7 = SuperscalarInstructionInfo("IXOR_C7", SuperscalarInstructionType::IXOR_C7, MacroOp::Xor_ri, -1);
    // 创建一个名为IADD_C8的SuperscalarInstructionInfo对象，类型为IADD_C8，宏操作为Add_ri，参数为-1
    const SuperscalarInstructionInfo SuperscalarInstructionInfo::IADD_C8 = SuperscalarInstructionInfo("IADD_C8", SuperscalarInstructionType::IADD_C8, MacroOp::Add_ri, -1);
    // 创建一个名为IXOR_C8的SuperscalarInstructionInfo对象，类型为IXOR_C8，宏操作为Xor_ri，参数为-1
    const SuperscalarInstructionInfo SuperscalarInstructionInfo::IXOR_C8 = SuperscalarInstructionInfo("IXOR_C8", SuperscalarInstructionType::IXOR_C8, MacroOp::Xor_ri, -1);
    // 创建一个名为IADD_C9的SuperscalarInstructionInfo对象，类型为IADD_C9，宏操作为Add_ri，参数为-1
    const SuperscalarInstructionInfo SuperscalarInstructionInfo::IADD_C9 = SuperscalarInstructionInfo("IADD_C9", SuperscalarInstructionType::IADD_C9, MacroOp::Add_ri, -1);
    // 创建一个名为IXOR_C9的SuperscalarInstructionInfo对象，类型为IXOR_C9，宏操作为Xor_ri，参数为-1
    const SuperscalarInstructionInfo SuperscalarInstructionInfo::IXOR_C9 = SuperscalarInstructionInfo("IXOR_C9", SuperscalarInstructionType::IXOR_C9, MacroOp::Xor_ri, -1);
    
    // 创建一个名为IMULH_R的SuperscalarInstructionInfo对象，类型为IMULH_R，操作数组为IMULH_R_ops_array，参数为1，0，1
    const SuperscalarInstructionInfo SuperscalarInstructionInfo::IMULH_R = SuperscalarInstructionInfo("IMULH_R", SuperscalarInstructionType::IMULH_R, IMULH_R_ops_array, 1, 0, 1);
    // 创建一个名为ISMULH_R的SuperscalarInstructionInfo对象，类型为ISMULH_R，操作数组为ISMULH_R_ops_array，参数为1，0，1
    const SuperscalarInstructionInfo SuperscalarInstructionInfo::ISMULH_R = SuperscalarInstructionInfo("ISMULH_R", SuperscalarInstructionType::ISMULH_R, ISMULH_R_ops_array, 1, 0, 1);
    // 创建一个名为IMUL_RCP的SuperscalarInstructionInfo对象，类型为IMUL_RCP，操作数组为IMUL_RCP_ops_array，参数为1，1，-1
    const SuperscalarInstructionInfo SuperscalarInstructionInfo::IMUL_RCP = SuperscalarInstructionInfo("IMUL_RCP", SuperscalarInstructionType::IMUL_RCP, IMUL_RCP_ops_array, 1, 1, -1);
    
    // 创建一个名为NOP的SuperscalarInstructionInfo对象，类型为NOP
    const SuperscalarInstructionInfo SuperscalarInstructionInfo::NOP = SuperscalarInstructionInfo("NOP");
    
    // 这些是将16字节窗口分割为3或4个x86指令的一些选项。
    // RandomX使用本机大小为3（sub，xor，mul，mov），4（lea，mul），7（xor，add immediate）或10字节（mov 64位立即数）的指令。
    // 定义不同大小的缓冲区，用于存储不同长度的指令
    const int buffer0[] = { 4, 8, 4 };  // 缓冲区0，包含3个元素，分别为4, 8, 4
    const int buffer1[] = { 7, 3, 3, 3 };  // 缓冲区1，包含4个元素，分别为7, 3, 3, 3
    const int buffer2[] = { 3, 7, 3, 3 };  // 缓冲区2，包含4个元素，分别为3, 7, 3, 3
    const int buffer3[] = { 4, 9, 3 };  // 缓冲区3，包含3个元素，分别为4, 9, 3
    const int buffer4[] = { 4, 4, 4, 4 };  // 缓冲区4，包含4个元素，分别为4, 4, 4, 4
    const int buffer5[] = { 3, 3, 10 };  // 缓冲区5，包含3个元素，分别为3, 3, 10

    // 定义一个名为DecoderBuffer的类
    class DecoderBuffer {
    public:
        static const DecoderBuffer Default;
        // 定义一个模板类 DecoderBuffer，包含构造函数和一些成员函数
        template <size_t N>
        DecoderBuffer(const char* name, int index, const int(&arr)[N])
            : name_(name), index_(index), counts_(arr), opsCount_(N) {}
        // 返回 counts_ 数组的指针
        const int* getCounts() const {
            return counts_;
        }
        // 返回 opsCount_ 的值
        int getSize() const {
            return opsCount_;
        }
        // 返回 index_ 的值
        int getIndex() const {
            return index_;
        }
        // 返回 name_ 的值
        const char* getName() const {
            return name_;
        }
        // 根据当前指令类型、周期数、乘法计数和生成器对象，返回下一个 DecoderBuffer 对象的指针
        const DecoderBuffer* fetchNext(SuperscalarInstructionType instrType, int cycle, int mulCount, Blake2Generator& gen) const {
            // 如果当前指令类型是 "IMULH_R" 或 "ISMULH_R"，则返回 decodeBuffer3310 对象的指针
            // 因为完整的 128 位乘法指令在 Intel CPU 上占用 3 个字节，并解码为 2 个微操作
            // Intel CPU 每个周期最多可以解码 4 个微操作，所以需要 2-1-1 的配置，总共 3 个宏操作
            if (instrType == SuperscalarInstructionType::IMULH_R || instrType == SuperscalarInstructionType::ISMULH_R)
                return &decodeBuffer3310;

            // 如果乘法计数小于周期数加 1，则返回 decodeBuffer4444 对象的指针
            // 为了确保乘法端口饱和，如果乘法计数小于周期数加 1，则生成 4-4-4-4 的配置
            if (mulCount < cycle + 1)
                return &decodeBuffer4444;

            // 如果当前指令类型是 "IMUL_RCP"，则根据生成器对象的字节值决定返回 decodeBuffer484 或 decodeBuffer493 对象的指针
            // decodeBuffer484 对象的第一个字节用于乘法，decodeBuffer493 对象的第一个字节不用于乘法
            if(instrType == SuperscalarInstructionType::IMUL_RCP)
                return (gen.getByte() & 1) ? &decodeBuffer484 : &decodeBuffer493;

            // 默认情况下，返回一个随机的 fetch 配置
            return fetchNextDefault(gen);
        }
    private:
        // 声明一个指向常量字符的指针，用于存储名称
        const char* name_ = nullptr;
        // 声明一个整型变量，用于存储索引
        int index_ = -1;
        // 声明一个指向常量整型的指针，用于存储计数
        const int* counts_ = nullptr;
        // 声明一个整型变量，用于存储操作计数
        int opsCount_ = 0;
        // 默认构造函数
        DecoderBuffer() = default;
        // 声明并定义一些静态常量DecoderBuffer对象
        static const DecoderBuffer decodeBuffer484;
        static const DecoderBuffer decodeBuffer7333;
        static const DecoderBuffer decodeBuffer3733;
        static const DecoderBuffer decodeBuffer493;
        static const DecoderBuffer decodeBuffer4444;
        static const DecoderBuffer decodeBuffer3310;
        // 声明一个指向DecoderBuffer对象的指针数组
        static const DecoderBuffer* decodeBuffers[4];
        // 定义一个私有成员函数，用于从Blake2Generator对象中获取下一个默认的DecoderBuffer对象
        const DecoderBuffer* fetchNextDefault(Blake2Generator& gen) const {
            return decodeBuffers[gen.getByte() & 3];
        }
    };

    // 定义静态常量DecoderBuffer对象，并初始化
    const DecoderBuffer DecoderBuffer::decodeBuffer484 = DecoderBuffer("4,8,4", 0, buffer0);
    const DecoderBuffer DecoderBuffer::decodeBuffer7333 = DecoderBuffer("7,3,3,3", 1, buffer1);
    const DecoderBuffer DecoderBuffer::decodeBuffer3733 = DecoderBuffer("3,7,3,3", 2, buffer2);
    const DecoderBuffer DecoderBuffer::decodeBuffer493 = DecoderBuffer("4,9,3", 3, buffer3);
    const DecoderBuffer DecoderBuffer::decodeBuffer4444 = DecoderBuffer("4,4,4,4", 4, buffer4);
    const DecoderBuffer DecoderBuffer::decodeBuffer3310 = DecoderBuffer("3,3,10", 5, buffer5);

    // 定义一个指向DecoderBuffer对象的指针数组，并初始化
    const DecoderBuffer* DecoderBuffer::decodeBuffers[4] = {
            &DecoderBuffer::decodeBuffer484,
            &DecoderBuffer::decodeBuffer7333,
            &DecoderBuffer::decodeBuffer3733,
            &DecoderBuffer::decodeBuffer493,
    };

    // 定义一个静态常量DecoderBuffer对象
    const DecoderBuffer DecoderBuffer::Default = DecoderBuffer();

    // 定义一个指向SuperscalarInstructionInfo对象的指针数组，并初始化
    const SuperscalarInstructionInfo* slot_3[]  = { &SuperscalarInstructionInfo::ISUB_R, &SuperscalarInstructionInfo::IXOR_R };
    const SuperscalarInstructionInfo* slot_3L[] = { &SuperscalarInstructionInfo::ISUB_R, &SuperscalarInstructionInfo::IXOR_R, &SuperscalarInstructionInfo::IMULH_R, &SuperscalarInstructionInfo::ISMULH_R };
    // 定义一个指针数组，包含两个指向SuperscalarInstructionInfo对象的指针
    const SuperscalarInstructionInfo* slot_4[]  = { &SuperscalarInstructionInfo::IROR_C, &SuperscalarInstructionInfo::IADD_RS };
    // 定义一个指针数组，包含两个指向SuperscalarInstructionInfo对象的指针
    const SuperscalarInstructionInfo* slot_7[]  = { &SuperscalarInstructionInfo::IXOR_C7, &SuperscalarInstructionInfo::IADD_C7 };
    // 定义一个指针数组，包含两个指向SuperscalarInstructionInfo对象的指针
    const SuperscalarInstructionInfo* slot_8[] = { &SuperscalarInstructionInfo::IXOR_C8, &SuperscalarInstructionInfo::IADD_C8 };
    // 定义一个指针数组，包含两个指向SuperscalarInstructionInfo对象的指针
    const SuperscalarInstructionInfo* slot_9[] = { &SuperscalarInstructionInfo::IXOR_C9, &SuperscalarInstructionInfo::IADD_C9 };
    // 定义一个指向SuperscalarInstructionInfo对象的指针
    const SuperscalarInstructionInfo* slot_10   = &SuperscalarInstructionInfo::IMUL_RCP;

    // 静态方法，用于选择寄存器
    static bool selectRegister(std::vector<int>& availableRegisters, Blake2Generator& gen, int& reg) {
        int index;
        // 如果可用寄存器数量为0，则返回false
        if (availableRegisters.size() == 0)
            return false;

        // 如果可用寄存器数量大于1，则随机选择一个寄存器
        if (availableRegisters.size() > 1) {
            index = gen.getUInt32() % availableRegisters.size();
        }
        // 如果可用寄存器数量为1，则直接选择该寄存器
        else {
            index = 0;
        }
        // 将选择的寄存器赋值给reg
        reg = availableRegisters[index];
        return true;
    }

    // 寄存器信息类
    class RegisterInfo {
    public:
        // 构造函数，初始化成员变量
        RegisterInfo() : latency(0), lastOpGroup(SuperscalarInstructionType::INVALID), lastOpPar(-1), value(0) {}
        int latency; // 延迟
        SuperscalarInstructionType lastOpGroup; // 上一个操作的类型
        int lastOpPar; // 上一个操作的参数
        int value; // 值
    };

    //"SuperscalarInstruction"由一个或多个宏操作组成
    // 超标量指令类
    class SuperscalarInstruction {
    private:
        const SuperscalarInstructionInfo* info_; // 指向SuperscalarInstructionInfo对象的指针
        int src_ = -1; // 源寄存器
        int dst_ = -1; // 目的寄存器
        int mod_ = 0; // 修改标志
        uint32_t imm32_ = 0; // 32位立即数
        SuperscalarInstructionType opGroup_ = SuperscalarInstructionType::INVALID; // 操作类型
        int opGroupPar_ = 0; // 操作参数
        bool canReuse_ = false; // 是否可重用
        bool groupParIsSource_ = false; // 操作参数是否为源操作数

        // 重置方法
        void reset() {
            src_ = dst_ = -1;
            canReuse_ = groupParIsSource_ = false;
        }

        // 构造函数，初始化info_
        SuperscalarInstruction(const SuperscalarInstructionInfo* info) : info_(info) {
        }
    };
    // 定义空指令，使用SuperscalarInstructionInfo::NOP初始化
    const SuperscalarInstruction SuperscalarInstruction::Null = SuperscalarInstruction(&SuperscalarInstructionInfo::NOP);

    // 定义循环映射大小为RANDOMX_SUPERSCALAR_MAX_LATENCY + 4
    constexpr int CYCLE_MAP_SIZE = RANDOMX_SUPERSCALAR_MAX_LATENCY + 4;
    // 定义向前查看周期数为4
    constexpr int LOOK_FORWARD_CYCLES = 4;
    // 定义最大丢弃计数为256
    constexpr int MAX_THROWAWAY_COUNT = 256;

    // 根据是否提交，调度微操作到执行端口
    template<bool commit>
    static int scheduleUop(ExecutionPort::type uop, ExecutionPort::type(&portBusy)[CYCLE_MAP_SIZE][3], int cycle) {
        // 这里的调度是乐观的，通过检查端口的可用性进行调度，顺序为P5 -> P0 -> P1，以避免过载P1（乘法）的指令
        for (; cycle < static_cast<int>(RandomX_CurrentConfig.SuperscalarLatency) + 4; ++cycle) {
            // 如果uop需要P5端口且P5端口在当前周期未被占用
            if ((uop & ExecutionPort::P5) != 0 && !portBusy[cycle][2]) {
                // 如果提交为真，打印P5在当前周期的信息
                if (commit) {
                    if (trace) std::cout << "; P5 at cycle " << cycle << std::endl;
                    portBusy[cycle][2] = uop;
                }
                return cycle;
            }
            // 如果uop需要P0端口且P0端口在当前周期未被占用
            if ((uop & ExecutionPort::P0) != 0 && !portBusy[cycle][0]) {
                // 如果提交为真，打印P0在当前周期的信息
                if (commit) {
                    if (trace) std::cout << "; P0 at cycle " << cycle << std::endl;
                    portBusy[cycle][0] = uop;
                }
                return cycle;
            }
            // 如果uop需要P1端口且P1端口在当前周期未被占用
            if ((uop & ExecutionPort::P1) != 0 && !portBusy[cycle][1]) {
                // 如果提交为真，打印P1在当前周期的信息
                if (commit) {
                    if (trace) std::cout << "; P1 at cycle " << cycle << std::endl;
                    portBusy[cycle][1] = uop;
                }
                return cycle;
            }
        }
        return -1;
    }

    template<bool commit>
    // 定义一个静态函数，用于调度宏操作的执行
    static int scheduleMop(const MacroOp& mop, ExecutionPort::type(&portBusy)[CYCLE_MAP_SIZE][3], int cycle, int depCycle) {
        // 如果这个宏操作依赖于前一个宏操作，则根据需要增加起始周期
        // 这处理了 IMUL_RCP 中的显式依赖链
        if (mop.isDependent()) {
            cycle = (cycle > depCycle) ? cycle : depCycle;
        }
        // 移动指令被消除，不需要执行单元
        if (mop.isEliminated()) {
            if (commit)
                if (trace) std::cout << "; (eliminated)" << std::endl;
            return cycle;
        }
        else if (mop.isSimple()) {
            // 这个宏操作只有一个微操作
            return scheduleUop<commit>(mop.getUop1(), portBusy, cycle);
        }
        else {
            // 有2个微操作的宏操作通过要求两个微操作在同一个周期内执行来保守地进行调度
            for (; cycle < static_cast<int>(RandomX_CurrentConfig.SuperscalarLatency) + 4; ++cycle) {

                int cycle1 = scheduleUop<false>(mop.getUop1(), portBusy, cycle);
                int cycle2 = scheduleUop<false>(mop.getUop2(), portBusy, cycle);

                if (cycle1 >= 0 && cycle1 == cycle2) {
                    if (commit) {
                        scheduleUop<true>(mop.getUop1(), portBusy, cycle1);
                        scheduleUop<true>(mop.getUop2(), portBusy, cycle2);
                    }
                    return cycle1;
                }
            }
        }

        return -1;
    }
    # 执行超标量程序，使用给定的寄存器和程序
    void executeSuperscalar(int_reg_t(&r)[8], SuperscalarProgram& prog) {
        # 遍历程序中的每一条指令
        for (unsigned j = 0; j < prog.getSize(); ++j) {
            # 获取当前指令
            Instruction& instr = prog(j);
            # 根据指令的操作码执行相应的操作
            switch ((SuperscalarInstructionType)instr.opcode)
            {
            # 如果操作码是ISUB_R，执行寄存器的减法操作
            case SuperscalarInstructionType::ISUB_R:
                r[instr.dst] -= r[instr.src];
                break;
            # 如果操作码是IXOR_R，执行寄存器的异或操作
            case SuperscalarInstructionType::IXOR_R:
                r[instr.dst] ^= r[instr.src];
                break;
            # 如果操作码是IADD_RS，执行寄存器的加法操作，并将源寄存器左移指定位数后相加
            case SuperscalarInstructionType::IADD_RS:
                r[instr.dst] += r[instr.src] << instr.getModShift();
                break;
            # 如果操作码是IMUL_R，执行寄存器的乘法操作
            case SuperscalarInstructionType::IMUL_R:
                r[instr.dst] *= r[instr.src];
                break;
            # 如果操作码是IROR_C，执行寄存器的循环右移操作，移动位数由立即数指定
            case SuperscalarInstructionType::IROR_C:
                r[instr.dst] = rotr64(r[instr.dst], instr.getImm32());
                break;
            # 如果操作码是IADD_C7、IADD_C8或IADD_C9，执行寄存器的加法操作，并将立即数符号扩展后相加
            case SuperscalarInstructionType::IADD_C7:
            case SuperscalarInstructionType::IADD_C8:
            case SuperscalarInstructionType::IADD_C9:
                r[instr.dst] += signExtend2sCompl(instr.getImm32());
                break;
            # 如果操作码是IXOR_C7、IXOR_C8或IXOR_C9，执行寄存器的异或操作，并将立即数符号扩展后异或
            case SuperscalarInstructionType::IXOR_C7:
            case SuperscalarInstructionType::IXOR_C8:
            case SuperscalarInstructionType::IXOR_C9:
                r[instr.dst] ^= signExtend2sCompl(instr.getImm32());
                break;
            # 如果操作码是IMULH_R，执行寄存器的高位乘法操作
            case SuperscalarInstructionType::IMULH_R:
                r[instr.dst] = mulh(r[instr.dst], r[instr.src]);
                break;
            # 如果操作码是ISMULH_R，执行寄存器的有符号高位乘法操作
            case SuperscalarInstructionType::ISMULH_R:
                r[instr.dst] = smulh(r[instr.dst], r[instr.src]);
                break;
            # 如果操作码是IMUL_RCP，执行寄存器的乘法操作，并将立即数的倒数乘以寄存器的值
            case SuperscalarInstructionType::IMUL_RCP:
                r[instr.dst] *= randomx_reciprocal(instr.getImm32());
                break;
            # 如果操作码不在以上情况中，表示出现了未知的操作码，抛出异常
            default:
                UNREACHABLE;
            }
        }
    }
# 闭合前面的函数定义
```