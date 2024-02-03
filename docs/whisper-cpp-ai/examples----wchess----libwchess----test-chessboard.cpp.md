# `whisper.cpp\examples\wchess\libwchess\test-chessboard.cpp`

```cpp
// 包含 Chessboard 类的头文件

#include "Chessboard.h"

// 定义断言宏，用于检查条件是否成立，若不成立则输出错误信息并退出程序
#define ASSERT(x) \
    do { \
        if (!(x)) { \
            fprintf(stderr, "ASSERT: %s:%d: %s\n", __FILE__, __LINE__, #x); \
            fflush(stderr); \
            exit(1); \
        } \
    } while (0)

// 主函数
int main() {
    {
        // 创建一个 Chessboard 对象
        Chessboard chess;

        // 断言棋盘处理 "pawn to d4" 的结果为 "d2-d4"
        ASSERT(chess.process("pawn to d4") == "d2-d4");
        // 断言棋盘处理 "e5" 的结果为 "e7-e5"
        ASSERT(chess.process("e5") == "e7-e5");
        // 断言棋盘处理 "c1 h6" 的结果为 "c1-h6"
        ASSERT(chess.process("c1 h6") == "c1-h6");
        // 断言棋盘处理 "queen h4" 的结果为 "d8-h4"
        ASSERT(chess.process("queen h4") == "d8-h4");
        // 断言棋盘处理 "bishop to g5" 的结果为 "h6-g5"
        ASSERT(chess.process("bishop to g5") == "h6-g5");
        // 断言棋盘处理 "bishop to b4" 的结果为 "f8-b4"
        ASSERT(chess.process("bishop to b4") == "f8-b4");
        // 断言棋盘处理 "c4" 的结果为空字符串
        ASSERT(chess.process("c4") == "");
        // 断言棋盘处理 "knight c3" 的结果为 "b1-c3"
        ASSERT(chess.process("knight c3") == "b1-c3");
        // 断言棋盘处理 "knight c6" 的结果为 "b8-c6"
        ASSERT(chess.process("knight c6") == "b8-c6");
        // 断言棋盘处理 "f3" 的结果为空字符串
        ASSERT(chess.process("f3") == "");
    }

    {
        // 创建一个新的 Chessboard 对象
        Chessboard chess;

        // 断言棋盘处理 "d4" 的结果为 "d2-d4"
        ASSERT(chess.process("d4") == "d2-d4");
        // 断言棋盘处理 "e5" 的结果为 "e7-e5"
        ASSERT(chess.process("e5") == "e7-e5");
        // 断言棋盘处理 "e4" 的结果为 "e2-e4"
        ASSERT(chess.process("e4") == "e2-e4");
        // 断言棋盘处理 "queen h4" 的结果为 "d8-h4"
        ASSERT(chess.process("queen h4") == "d8-h4");
        // 断言棋盘处理 "queen h5" 的结果为 "d1-h5"
        ASSERT(chess.process("queen h5") == "d1-h5");
        // 断言棋盘处理 "f5" 的结果为空字符串
        ASSERT(chess.process("f5") == "");
        // 断言棋盘处理 "g6" 的结果为 "g7-g6"
        ASSERT(chess.process("g6") == "g7-g6");
        // 断言棋盘处理 "knight e2" 的结果为 "g1-e2"
        ASSERT(chess.process("knight e2") == "g1-e2");
        // 断言棋盘处理 "f5" 的结果为 "f7-f5"
        ASSERT(chess.process("f5") == "f7-f5");
        // 断言棋盘处理 "knight g3" 的结果为 "e2-g3"
        ASSERT(chess.process("knight g3") == "e2-g3");
        // 断言棋盘处理 "g5" 的结果为空字符串
        ASSERT(chess.process("g5") == "");
        // 断言棋盘处理 "king e7" 的结果为 "e8-e7"
        ASSERT(chess.process("king e7") == "e8-e7");
        // 断言棋盘处理 "f4" 的结果为 "f2-f4"
        ASSERT(chess.process("f4") == "f2-f4");
        // 断言棋盘处理 "g5" 的结果为 "g6-g5"
        ASSERT(chess.process("g5") == "g6-g5");
    }
}
    {
        // 创建一个棋盘对象
        Chessboard chess;

        // 断言处理棋盘移动"e4"的结果为"e2-e4"
        ASSERT(chess.process("e4") == "e2-e4");
        // 断言处理棋盘移动"c5"的结果为"c7-c5"
        ASSERT(chess.process("c5") == "c7-c5");
        // 断言处理棋盘移动"e5"的结果为"e4-e5"
        ASSERT(chess.process("e5") == "e4-e5");
        // 断言处理棋盘移动"c4"的结果为"c5-c4"
        ASSERT(chess.process("c4") == "c5-c4");
        // 断言处理棋盘移动"e6"的结果为"e5-e6"
        ASSERT(chess.process("e6") == "e5-e6");
        // 断言处理棋盘移动"c3"的结果为"c4-c3"
        ASSERT(chess.process("c3") == "c4-c3");
        // 断言处理棋盘移动"e7"的结果为空字符串
        ASSERT(chess.process("e7") == "");
        // 断言处理棋盘移动"f7"的结果为"e6-f7"
        ASSERT(chess.process("f7") == "e6-f7");
        // 断言处理棋盘移动"d2"的结果为空字符串
        ASSERT(chess.process("d2") == "");
        // 断言处理棋盘移动"king to f7"的结果为"e8-f7"
        ASSERT(chess.process("king to f7") == "e8-f7");
        // 断言处理棋盘移动"f4"的结果为"f2-f4"
        ASSERT(chess.process("f4") == "f2-f4");
        // 断言处理棋盘移动"d2"的结果为"c3-d2"
        ASSERT(chess.process("d2") == "c3-d2");
        // 断言处理棋盘移动"f5"的结果为空字符串
        ASSERT(chess.process("f5") == "");
        // 断言处理棋盘移动"king to e2"的结果为"e1-e2"
        ASSERT(chess.process("king to e2") == "e1-e2");
        // 断言处理棋盘移动"king to g6"的结果为"f7-g6"
        ASSERT(chess.process("king to g6") == "f7-g6");
        // 断言处理棋盘移动"f5"的结果为"f4-f5"
        ASSERT(chess.process("f5") == "f4-f5");
        // 断言处理棋盘移动"e6"的结果为空字符串
        ASSERT(chess.process("e6") == "");
        // 断言处理棋盘移动"king to h5"的结果为"g6-h5"
        ASSERT(chess.process("king to h5") == "g6-h5");
        // 断言处理棋盘移动"g4"的结果为"g2-g4"
        ASSERT(chess.process("g4") == "g2-g4");
        // 断言处理棋盘移动"king to g5"的结果为"h5-g5"
        ASSERT(chess.process("king to g5") == "h5-g5");
        // 断言处理棋盘移动"h4"的结果为"h2-h4"
        ASSERT(chess.process("h4") == "h2-h4");
        // 断言处理棋盘移动"king to h5"的结果为空字符串
        ASSERT(chess.process("king to h5") == "");
        // 断言处理棋盘移动"king to g6"的结果为空字符串
        ASSERT(chess.process("king to g6") == "");
        // 断言处理棋盘移动"king to h6"的结果为"g5-h6"
        ASSERT(chess.process("king to h6") == "g5-h6");
        // 断言处理棋盘移动"bishop to d2"的结果为"c1-d2"
        ASSERT(chess.process("bishop to d2") == "c1-d2");
        // 断言处理棋盘移动"king to g5"的结果为空字符串
        ASSERT(chess.process("king to g5") == "");
        // 断言处理棋盘移动"g5"的结果为"g7-g5"
        ASSERT(chess.process("g5") == "g7-g5");
    }

    {
        // 创建一个新的棋盘对象
        Chessboard chess;
        // 断言处理棋盘移动"f4"的结果为"f2-f4"
        ASSERT(chess.process("f4") == "f2-f4");
        // 断言处理棋盘移动"e5"的结果为"e7-e5"
        ASSERT(chess.process("e5") == "e7-e5");
        // 断言处理棋盘移动"g4"的结果为"g2-g4"
        ASSERT(chess.process("g4") == "g2-g4");
        // 断言处理棋盘移动"queen to h4"的结果为"d8-h4#"
        ASSERT(chess.process("queen to h4") == "d8-h4#");
        // 断言处理棋盘移动"knight f3"的结果为空字符串
        ASSERT(chess.process("knight f3") == "");
        // 断言棋盘的语法为空
        ASSERT(chess.grammar().empty());
    }
    {
        // 创建一个棋盘对象
        Chessboard chess;
        // 断言处理棋盘移动指令"f4"的结果为"f2-f4"
        ASSERT(chess.process("f4") == "f2-f4");
        // 断言处理棋盘移动指令"e5"的结果为"e7-e5"
        ASSERT(chess.process("e5") == "e7-e5");
        // 断言处理棋盘移动指令"g4"的结果为"g2-g4"
        ASSERT(chess.process("g4") == "g2-g4");
        // 断言处理棋盘移动指令"d5"的结果为"d7-d5"
        ASSERT(chess.process("d5") == "d7-d5");
        // 断言处理棋盘移动指令"g1 f3"的结果为"g1-f3"
        ASSERT(chess.process("g1 f3") == "g1-f3");
        // 断言处理棋盘移动指令"queen to h4"的结果为"d8-h4"
        ASSERT(chess.process("queen to h4") == "d8-h4");
        // 断言棋盘语法不为空
        ASSERT(!chess.grammar().empty());
    }
    
    {
        // 创建一个新的棋盘对象
        Chessboard chess;
        // 断言处理棋盘移动指令"knight c3"的结果为"b1-c3"
        ASSERT(chess.process("knight c3") == "b1-c3");
        // 断言处理棋盘移动指令"knight c6"的结果为"b8-c6"
        ASSERT(chess.process("knight c6") == "b8-c6");
        // 断言处理棋盘移动指令"knight b5"的结果为"c3-b5"
        ASSERT(chess.process("knight b5") == "c3-b5");
        // 断言处理棋盘移动指令"knight f6"的结果为"g8-f6"
        ASSERT(chess.process("knight f6") == "g8-f6");
        // 断言处理棋盘移动指令"knight d6"的结果为空字符串
        ASSERT(chess.process("knight d6") == "");
        // 断言处理棋盘移动指令"d6"的结果为"c7-d6"
        ASSERT(chess.process("d6") == "c7-d6");
        // 断言处理棋盘移动指令"e4"的结果为"e2-e4"
        ASSERT(chess.process("e4") == "e2-e4");
        // 断言处理棋盘移动指令"knight d4"的结果为"c6-d4"
        ASSERT(chess.process("knight d4") == "c6-d4");
        // 断言处理棋盘移动指令"d3"的结果为"d2-d3"
        ASSERT(chess.process("d3") == "d2-d3");
        // 断言处理棋盘移动指令"knight e4"的结果为"f6-e4"
        ASSERT(chess.process("knight e4") == "f6-e4");
        // 断言处理棋盘移动指令"king to e2"的结果为空字符串
        ASSERT(chess.process("king to e2") == "");
        // 断言处理棋盘移动指令"king to d2"的结果为空字符串
        ASSERT(chess.process("king to d2") == "");
    }
# 闭合大括号，表示代码块的结束
```