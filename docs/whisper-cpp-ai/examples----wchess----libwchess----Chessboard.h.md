# `whisper.cpp\examples\wchess\libwchess\Chessboard.h`

```cpp
#pragma once
#include <string>
#include <set>
#include <memory>

// just basic validation
// fixme: missing en passant, castling, promotion, etc.
// 定义 State 结构体，Piece 类和 Chessboard 类，用于表示棋盘状态和棋子
struct State;
class Piece;
class Chessboard {
public:
    // Chessboard 类的构造函数
    Chessboard();
    // Chessboard 类的析构函数
    ~Chessboard();
    // 处理用户输入的命令并返回结果
    std::string process(const std::string& command);
    // 将棋盘状态转换为字符串形式
    std::string stringifyBoard();
    // 返回语法规则
    const std::string& grammar() { return m_grammar; }
    // 返回提示信息
    const std::string& prompt() { return m_prompt; }
    // 设置提示信息
    void setPrompt(const std::string& prompt);
private:
    // 解析用户输入的命令，获取棋子和目标位置
    bool parseCommand(const std::string& command, Piece*& piece, char& pos_to);
    // 移动棋子到目标位置
    bool move(Piece& piece, char pos);
    // 标记更新
    void flagUpdates(char pos_from, char pos_to);
    // 更新棋子的针对性
    void updatePins(Piece& piece);
    // 检测是否将军
    void detectChecks();
    // 设置语法规则
    void setGrammar();

    // 棋盘状态的智能指针
    std::unique_ptr<State> m_state;
    // 允许在将军状态下的操作
    std::set<char> m_allowedInCheck;
    // 是否处于将军状态
    bool m_inCheck = false;
    // 移动计数器
    int m_moveCounter = 0;
    // 语法规则
    std::string m_grammar;
    // 提示信息
    std::string m_prompt;
};
```