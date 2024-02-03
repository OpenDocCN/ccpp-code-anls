# `whisper.cpp\examples\wchess\libwchess\Chessboard.cpp`

```cpp
// 包含头文件 Chessboard.h
#include "Chessboard.h"

// 包含必要的标准库头文件
#include <array>
#include <vector>
#include <algorithm>
#include <cstring>
#include <set>
#include <list>
#include <chrono>

// 命名空间
namespace {
    // 棋盘位置数组
    constexpr std::array<const char*, 64> positions = {
        "a1", "b1", "c1", "d1", "e1", "f1", "g1", "h1",
        "a2", "b2", "c2", "d2", "e2", "f2", "g2", "h2",
        "a3", "b3", "c3", "d3", "e3", "f3", "g3", "h3",
        "a4", "b4", "c4", "d4", "e4", "f4", "g4", "h4",
        "a5", "b5", "c5", "d5", "e5", "f5", "g5", "h5",
        "a6", "b6", "c6", "d6", "e6", "f6", "g6", "h6",
        "a7", "b7", "c7", "d7", "e7", "f7", "g7", "h7",
        "a8", "b8", "c8", "d8", "e8", "f8", "g8", "h8",
    };
    // 无效位置标记
    constexpr char INVALID_POS = positions.size();
    // 行索引
    constexpr int R = 0;
    // 列索引
    constexpr int F = 1;
    // 定义宏，用于计算位置
    #define FILE (c[F] - '1')
    #define RANK (c[R] - 'a')
    // 字符串转换为位置
    constexpr char operator ""_P(const char * c, size_t size) {
        return size < 2 || RANK < 0 || RANK > 7 ||
            FILE < 0 || FILE > 7 ? INVALID_POS : FILE * 8 + RANK;
    }
    // 取消宏定义
    #undef FILE
    #undef RANK

    // 字符串视图结构体
    struct sview {
        const char * ptr = nullptr;
        size_t size = 0;

        // 构造函数
        sview() = default;
        sview(const char * p, size_t s) : ptr(p), size(s) {}
        sview(const std::string& s) : ptr(s.data()), size(s.size()) {}

        // 查找字符在字符串中的位置
        size_t find(char del, size_t pos) {
            while (pos < size && ptr[pos] != del) ++pos;
            return pos < size ? pos : std::string::npos;
        }
    };

    // 分割字符串
    std::vector<sview> split(sview str, char del) {
        std::vector<sview> res;
        size_t cur = 0;
        size_t last = 0;
        while (cur != std::string::npos) {
            if (str.ptr[last] == ' ') {
                ++last;
                continue;
            }
            cur = str.find(del, last);
            size_t len = cur == std::string::npos ? str.size - last : cur - last;
            res.emplace_back(str.ptr + last, len);
            last = cur + 1;
        }
        return res;
    }

    // 字符串转换为位置
    char strToPos(sview str) {
        return operator ""_P(str.ptr, str.size);
    }

    // 棋子名称数组
    constexpr std::array<const char*, 6> pieceNames = {
    # 字符串列表，包含国际象棋中的棋子名称
    "pawn", "knight", "bishop", "rook", "queen", "king",
};

// 黑色棋子的简称数组
static constexpr std::array<char, 6> blackShort =  {
    'p', 'n', 'b', 'r', 'q', 'k',
};
// 白色棋子的简称数组
static constexpr std::array<char, 6> whiteShort =  {
    'P', 'N', 'B', 'R', 'Q', 'K',
};

// 将字符串转换为棋子类型
char strToType(sview str) {
    // 在棋子名称数组中查找与给定字符串匹配的元素
    auto it = std::find_if(pieceNames.begin(), pieceNames.end(), [str] (const char* name) { return strncmp(name, str.ptr, str.size) == 0; });
    // 如果找到匹配的元素，则返回其索引，否则返回棋子名称数组的大小
    return it != pieceNames.end() ? it - pieceNames.begin() : pieceNames.size();
}

// 棋子移动的方向
using Direction = std::array<char, 2>;

// 定义各个方向的移动距离
constexpr Direction N   = {(char)  0, (char)  1};
constexpr Direction NNE = {(char)  1, (char)  2};
constexpr Direction NE  = {(char)  1, (char)  1};
constexpr Direction ENE = {(char)  2, (char)  1};
constexpr Direction E   = {(char)  1, (char)  0};
constexpr Direction ESE = {(char)  2, (char) -1};
constexpr Direction SE  = {(char)  1, (char) -1};
constexpr Direction SSE = {(char)  1, (char) -2};
constexpr Direction S   = {(char)  0, (char) -1};
constexpr Direction SSW = {(char) -1, (char) -2};
constexpr Direction SW  = {(char) -1, (char) -1};
constexpr Direction WSW = {(char) -2, (char) -1};
constexpr Direction W   = {(char) -1, (char)  0};
constexpr Direction WNW = {(char) -2, (char)  1};
constexpr Direction NW  = {(char) -1, (char)  1};
constexpr Direction NNW = {(char) -1, (char)  2};

// 根据当前位置和移动方向计算下一步位置
char makeStep(char pos, const Direction& d) {
    char next[2] = { char(positions[pos][R] + d[R]) , char(positions[pos][F] + d[F]) };
    return strToPos(sview{next, sizeof(next)});
}

// 根据移动方向和条件函数遍历棋盘
template<class Modifier>
char traverse(char pos, const Direction& d, const Modifier& m, int count = 8) {
    // 循环移动棋子，直到达到指定次数或满足条件
    while (--count >= 0) {
        pos = makeStep(pos, d);
        if (pos == INVALID_POS || m(pos)) break;
    }
    return pos;
}

// 标准化移动距离，使其只包含-1、0、1三种可能值
Direction normalize(const Direction& distance) {
    // 根据移动距离的正负情况进行标准化处理
    const int drp = distance[R] > 0 ? 1 : 0;
    const int drn = distance[R] < 0 ? 1 : 0;
    // 如果 F 点到目标点的距离大于 0，则 dfp 为 1，否则为 0
    const int dfp = distance[F] > 0 ? 1 : 0;
    // 如果 F 点到目标点的距离小于 0，则 dfn 为 1，否则为 0
    const int dfn = distance[F] < 0 ? 1 : 0;
    // 返回一个包含两个字符的元组，第一个字符为 drp - drn，第二个字符为 dfp - dfn
    return {char(drp - drn), char(dfp - dfn)};
}

// 定义 Pin 结构体，包含方向、固定棋子和被固定棋子
struct Pin {
    Direction d; // 方向
    Piece* pinner; // 固定棋子
    Piece* pinned; // 被固定棋子
};
// 使用别名定义 Pins 类型为 std::list<Pin>
using Pins = std::list<Pin>;
// 使用别名定义 Board 类型为 std::array<Piece*, 64>
using Board = std::array<Piece*, 64>;

// 过滤函数，根据固定方向和方向列表过滤出符合条件的方向
std::vector<Direction> filter(const Direction& pin, std::initializer_list<Direction> directions) {
    // 如果固定方向为0，则直接返回方向列表
    if (pin[R] == 0 && pin[F] == 0) return directions;
    // 否则创建结果向量
    std::vector<Direction> result;
    // 遍历方向列表
    for (auto& d : directions) {
        // 如果方向符合固定方向条件，则加入结果向量
        if ((d[R] == pin[R] || d[R] == -pin[R]) && (d[F] == pin[F] || d[F] == -pin[F])) result.push_back(d);
    }
    // 返回结果向量
    return result;
}

}

// 定义 Piece 类
class Piece {
public:
    // 定义枚举类型 Types 和 Colors
    enum Types : char {
        Pawn,
        Knight,
        Bishop,
        Rook,
        Queen,
        King,
        //
        NUM_PIECES
    };

    enum Colors : char {
        White,
        Black,
    };

    // 返回棋子名称
    const char* name() const;
    // 返回棋子初始
    char initial() const;
    // 返回棋子类型
    Types type() const { return m_type; }
    // 返回棋子颜色
    Colors color() const { return m_color; }
    // 返回棋子位置
    char pos() const { return m_pos; }
    // 设置棋子位置
    void setPos(char pos) {
        m_pos = pos;
        invalidate();
    }
    // 返回棋子坐标
    const char* coord() const;
    // 返回允许移动的位置
    const std::set<char>& allowed() const { return m_allowed; }
    // 判断是否可以到达指定位置
    bool canReach(char pos) const;
    // 移动模式
    virtual bool movePattern(char pos) const = 0;
    // 拿子
    void take();
    // 重新初始化
    virtual void reinit(const State& state) = 0;
    // 使棋子无效
    void invalidate();
protected:
    // 构造函数，初始化棋子类型、颜色、位置和允许移动的位置
    Piece(Types type, Colors color, char pos, std::set<char> allowed)
        : m_type(type), m_color(color), m_pos(pos), m_allowed(std::move(allowed)) {}
    Piece(const Piece&) = delete;
    ~Piece() = default;

    const Types m_type;
    const Colors m_color;
    char m_pos;
    std::set<char> m_allowed;
    bool m_update = false;
};

// 定义 Pawn 结构体，继承自 Piece
struct Pawn : public Piece {
    // 构造函数，初始化棋子颜色、位置和下一步可移动的位置
    Pawn(Colors color, char pos, std::set<char> next) : Piece(Types::Pawn, color, pos, std::move(next)) {}

    // 判断是否为第一步移动
    bool is_first_move() const {
        return m_color ? coord()[F] == '7' : coord()[F] == '2';
    }
    // 虚函数，移动棋子的模式，根据传入的位置参数
    virtual bool movePattern(char pos) const override {
        // 如果当前位置无效，则返回false
        if (m_pos == INVALID_POS) return false;
        // 获取当前位置和目标位置的坐标
        auto cur = coord();
        auto next = positions[pos];
        // 计算当前位置到目标位置的距离
        Direction distance = {char(next[R] - cur[R]), char(next[F] - cur[F])};
        // 根据棋子颜色确定前进方向
        char forward = m_color ? -1 : 1;
        // 判断是否符合移动规则
        return (forward == distance[F] && distance[R] * distance[R] <= 1)
            || (is_first_move() && 2 * forward == distance[F] && distance[R] == 0);
    }

    // 重新初始化状态
    virtual void reinit(const State& state) override;
// Knight 类，继承自 Piece 类，表示象棋中的“骑士”棋子
struct Knight : public Piece {
    // Knight 类的构造函数，初始化颜色、位置和可移动位置
    Knight(Colors color, char pos, std::set<char> next) : Piece(Types::Knight, color, pos, std::move(next)) {}

    // 重写移动模式函数，判断是否符合“骑士”棋子的移动规则
    virtual bool movePattern(char pos) const override {
        // 如果当前位置无效，则返回 false
        if (m_pos == INVALID_POS) return false;
        // 获取当前位置和目标位置的坐标
        auto cur = coord();
        auto next = positions[pos];
        // 计算横向和纵向的距离差
        Direction diff = {char(next[R] - cur[R]), char(next[F] - cur[F])};
        // 判断是否符合“骑士”棋子的移动规则（横向或纵向距离为2，另一方向距离为1）
        return diff[R]*diff[R] + diff[F]*diff[F] == 5;
    }

    // 重新初始化函数，根据当前状态重新初始化“骑士”棋子
    virtual void reinit(const State& state) override;
};

// Bishop 类，继承自 Piece 类，表示象棋中的“主教”棋子
struct Bishop : public Piece {
    // Bishop 类的构造函数，初始化颜色和位置
    Bishop(Colors color, char pos) : Piece(Types::Bishop, color, pos, {}) {}

    // 重写移动模式函数，判断是否符合“主教”棋子的移动规则
    virtual bool movePattern(char pos) const override {
        // 如果当前位置无效，则返回 false
        if (m_pos == INVALID_POS) return false;
        // 获取当前位置和目标位置的坐标
        auto cur = coord();
        auto next = positions[pos];
        // 判断是否符合“主教”棋子的移动规则（沿对角线移动）
        return cur[R] - cur[F] == next[R] - next[F] || cur[R] + cur[F] == next[R] + next[F];
    }

    // 重新初始化函数，根据当前状态重新初始化“主教”棋子
    virtual void reinit(const State& state) override;
};

// Rook 类，继承自 Piece 类，表示象棋中的“车”棋子
struct Rook : public Piece {
    // Rook 类的构造函数，初始化颜色和位置
    Rook(Colors color, char pos) : Piece(Types::Rook, color, pos, {}) {}

    // 重写移动模式函数，判断是否符合“车”棋子的移动规则
    virtual bool movePattern(char pos) const override {
        // 如果当前位置无效，则返回 false
        if (m_pos == INVALID_POS) return false;
        // 获取当前位置和目标位置的坐标
        auto cur = coord();
        auto next = positions[pos];
        // 判断是否符合“车”棋子的移动规则（沿横向或纵向移动）
        return cur[R] == next[R] || cur[F] == next[F];
    }

    // 重新初始化函数，根据当前状态重新初始化“车”棋子
    virtual void reinit(const State& state) override;
};

// Queen 类，继承自 Piece 类，表示象棋中的“皇后”棋子
struct Queen : public Piece {
    // Queen 类的构造函数，初始化颜色和位置
    Queen(Colors color, char pos) : Piece(Types::Queen, color, pos, {}) {}

    // 重写移动模式函数，判断是否符合“皇后”棋子的移动规则
    virtual bool movePattern(char pos) const override {
        // 如果当前位置无效，则返回 false
        if (m_pos == INVALID_POS) return false;
        // 获取当前位置和目标位置的坐标
        auto cur = coord();
        auto next = positions[pos];
        // 判断是否符合“皇后”棋子的移动规则（沿横向、纵向或对角线移动）
        return cur[R] == next[R] || cur[F] == next[F] || cur[R] - cur[F] == next[R] - next[F] || cur[R] + cur[F] == next[R] + next[F];
    }

    // 重新初始化函数，根据当前状态重新初始化“皇后”棋子
    virtual void reinit(const State& state) override;
};

// King 类，继承自 Piece 类，表示象棋中的“国王”棋子
struct King : public Piece {
    // King 类的构造函数，初始化颜色和位置
    King(Colors color, char pos) : Piece(Types::King, color, pos, {}) {}
    // 虚拟函数，移动棋子的模式
    virtual bool movePattern(char pos) const override {
        // 如果当前位置为无效位置，则返回false
        if (m_pos == INVALID_POS) return false;
        // 获取当前位置和下一个位置的坐标
        auto cur = coord();
        auto next = positions[pos];
        // 计算当前位置到下一个位置的水平和垂直方向的距离
        Direction diff = {char(next[R] - cur[R]), char(next[F] - cur[F])};
        // 判断距离是否小于等于2，如果是则返回true，否则返回false
        return diff[R]*diff[R] + diff[F]*diff[F] <= 2;
    }

    // 虚拟函数，重新初始化状态
    virtual void reinit(const State& state) override;
};

// 定义 PieceSet 结构体
struct PieceSet {
    // 返回指向第一个 Piece 的指针
    Piece* begin() { return &p1; }
    // 返回指向最后一个 Piece 后一个位置的指针
    Piece* end() { return &r2 + 1; }
    // 返回指向第一个 Piece 的常量指针
    const Piece* begin() const { return &p1; }
    // 返回指向最后一个 Piece 后一个位置的常量指针
    const Piece* end() const { return &r2 + 1; }
    // 重载 [] 运算符，返回指定位置的 Piece 引用
    Piece& operator[](int i) { return *(begin() + i); }
    // 重载 [] 运算符，返回指定位置的常量 Piece 引用
    const Piece& operator[](int i) const { return *(begin() + i); }

    // 定义各种棋子成员变量
    Pawn   p1;
    Pawn   p2;
    Pawn   p3;
    Pawn   p4;
    Pawn   p5;
    Pawn   p6;
    Pawn   p7;
    Pawn   p8;
    Rook   r1;
    Knight n1;
    Bishop b1;
    Queen  q;
    King   k;
    Bishop b2;
    Knight n2;
    Rook   r2;
};

// 定义 State 结构体
struct State {
    // 默认构造函数
    State();
    // 黑棋子集合
    PieceSet blacks;
    // 白棋子集合
    PieceSet whites;
    // 棋盘
    Board board;
    // 黑棋子的 pin 信息
    Pins blackPins;
    // 白棋子的 pin 信息
    Pins whitePins;
};

// 查找棋子的 pin 信息
Direction findPin(const Piece& piece, const State& state) {
    // 根据棋子颜色选择 pin 信息
    auto& pins = piece.color() ? state.blackPins : state.whitePins;
    // 在 pin 信息中查找当前棋子的 pin 信息
    auto it = std::find_if(pins.begin(), pins.end(), [&] (const Pin& pin) { return pin.pinned == &piece; });
    // 如果找到了 pin 信息，返回 pin 的方向
    if (it != pins.end()) return it->d;
    // 否则返回默认方向
    return {0, 0};
}

// 定义 Find 结构体
struct Find {
    // 构造函数，初始化棋盘
    Find(const Board& board) : m_board(board) {}
    // 重载 () 运算符，查找指定位置是否有棋子
    bool operator() (char pos) const { return m_board[pos]; }
    // 棋盘引用
    const Board& m_board;
};

// 定义 Add 结构体
struct Add {
    // 构造函数，初始化棋盘、移动集合和棋子颜色
    Add(const Board& board, std::set<char>& moves, Piece::Colors color) : m_board(board), m_moves(moves), m_color(color) {}
    // 重载 () 运算符，根据棋子颜色和棋盘信息添加可移动位置
    bool operator() (char pos) const {
        // 如果位置上没有棋子或者棋子颜色不同，将位置添加到移动集合中
        if (!m_board[pos] || m_board[pos]->color() != m_color) m_moves.insert(pos);
        return m_board[pos];
    }
    // 棋盘引用
    const Board& m_board;
    // 移动集合引用
    std::set<char>& m_moves;
    // 棋子颜色
    Piece::Colors m_color;
};

// 重新初始化 Pawn 棋子
void Pawn::reinit(const State& state) {
    // 如果棋子位置无效，直接返回
    if (m_pos == INVALID_POS) return;
    // 如果棋子不需要更新，直接返回
    if (!m_update) return;
    // 将更新标志位设为 false
    m_update = false;
    // 清空允许移动的位置集合
    m_allowed.clear();

    // 查找当前棋子的 pin 信息
    auto pin = findPin(*this, state);

    // 左右移动方向
    auto & left = m_color ? SW : NW;
    auto & right = m_color ? SE : NE;
    // 遍历过滤后的方向（左右），对每个方向进行操作
    for (auto& direction : filter(pin, { left, right })) {
        // 根据当前位置和方向计算新位置
        auto pos = makeStep(m_pos, direction);
        // 如果新位置有效且新位置上有棋子且棋子颜色与当前棋子颜色不同，则将新位置加入允许移动的位置集合
        if (pos != INVALID_POS && state.board[pos] && state.board[pos]->color() != m_color) m_allowed.insert(pos);
    }

    // 根据当前棋子颜色确定前进方向
    auto & forward = m_color ? S : N;
    // 如果前进方向不在过滤后的方向集合中
    if (!filter(pin, {forward}).empty()) {
        // 从当前位置开始沿着前进方向遍历
        traverse(m_pos, forward, [&] (char pos) {
                // 如果当前位置没有棋子，则将当前位置加入允许移动的位置集合
                if (!state.board[pos]) m_allowed.insert(pos);
                // 返回当前位置是否有棋子或者是否不是第一次移动
                return state.board[pos] || !is_first_move();
            }, 2);
    }
// 重新初始化 Knight 类的方法，根据当前状态更新可移动位置
void Knight::reinit(const State& state) {
    // 如果当前位置为无效位置，则返回
    if (m_pos == INVALID_POS) return;
    // 如果不需要更新，则返回
    if (!m_update) return;
    // 将更新标志设置为 false
    m_update = false;
    // 清空可移动位置集合
    m_allowed.clear();
    // 查找当前棋子的 pin 信息
    auto pin = findPin(*this, state);
    // 如果 R 或 F 方向有 pin，则返回
    if (pin[R] != 0 || pin[F] != 0) return;
    // 遍历所有可能的移动方向
    for (auto& direction : { NNE, ENE, ESE, SSE, SSW, WSW, WNW, NNW }) {
        // 计算移动后的位置
        auto pos = makeStep(m_pos, direction);
        // 如果位置有效且为空或者不是同色棋子，则加入可移动位置集合
        if (pos != INVALID_POS && (!state.board[pos] || state.board[pos]->color() != m_color)) m_allowed.insert(pos);
    }
}

// 重新初始化 Bishop 类的方法，根据当前状态更新可移动位置
void Bishop::reinit(const State& state) {
    // 如果当前位置为无效位置，则返回
    if (m_pos == INVALID_POS) return;
    // 如果不需要更新，则返回
    if (!m_update) return;
    // 将更新标志设置为 false
    m_update = false;
    // 清空可移动位置集合
    m_allowed.clear();
    // 查找当前棋子的 pin 信息
    auto pin = findPin(*this, state);
    // 遍历所有可能的移动方向
    for (auto& direction : filter(pin, { NE, SE, SW, NW })) {
        // 根据 pin 信息遍历路径，将可移动位置加入集合
        traverse(m_pos, direction, Add(state.board, m_allowed, m_color));
    }
}

// 重新初始化 Rook 类的方法，根据当前状态更新可移动位置
void Rook::reinit(const State& state) {
    // 如果当前位置为无效位置，则返回
    if (m_pos == INVALID_POS) return;
    // 如果不需要更新，则返回
    if (!m_update) return;
    // 将更新标志设置为 false
    m_update = false;
    // 清空可移动位置集合
    m_allowed.clear();
    // 查找当前棋子的 pin 信息
    auto pin = findPin(*this, state);
    // 遍历所有可能的移动方向
    for (auto& direction : filter(pin, { N, E, S, W })) {
        // 根据 pin 信息遍历路径，将可移动位置加入集合
        traverse(m_pos, direction, Add(state.board, m_allowed, m_color));
    }
}

// 重新初始化 Queen 类的方法，根据当前状态更新可移动位置
void Queen::reinit(const State& state) {
    // 如果当前位置为无效位置，则返回
    if (m_pos == INVALID_POS) return;
    // 如果不需要更新，则返回
    if (!m_update) return;
    // 将更新标志设置为 false
    m_update = false;
    // 清空可移动位置集合
    m_allowed.clear();
    // 查找当前棋子的 pin 信息
    auto pin = findPin(*this, state);
    // 遍历所有可能的移动方向
    for (auto& direction : filter(pin, { N, NE, E, SE, S, SW, W, NW })) {
        // 根据 pin 信息遍历路径，将可移动位置加入集合
        traverse(m_pos, direction, Add(state.board, m_allowed, m_color));
    }
}

// 重新初始化 King 类的方法，根据当前状态更新可移动位置
void King::reinit(const State& state) {
    // 如果当前位置为无效位置，则返回
    if (m_pos == INVALID_POS) return;
    // 如果不需要更新，则返回
    if (!m_update) return;
    // 将更新标志设置为 false
    m_update = false;
    // 清空可移动位置集合
    m_allowed.clear();
    // 获取敌方棋子集合
    auto& enemyPieces = m_color ? state.whites : state.blacks;
    // 根据棋子颜色确定左侧和右侧的攻击方向
    auto& pawnAttackLeft = m_color ? SW : NW;
    auto& pawnAttackRight = m_color ? SE : NE;
    // 遍历所有可能的移动方向
    for (auto& direction : { N, NE, E, SE, S, SW, W, NW }) {
        // 根据当前位置和移动方向计算新位置
        auto pos = makeStep(m_pos, direction);
        // 检查新位置是否有效且不是同色棋子
        bool accept = pos != INVALID_POS && !(state.board[pos] && state.board[pos]->color() == m_color);
        // 如果新位置符合条件
        if (accept) {
            // 遍历敌方棋子
            for (auto& p : enemyPieces) {
                // 如果敌方棋子无法移动到新位置，则继续下一个敌方棋子
                if (!p.movePattern(pos)) continue;
                // 如果敌方棋子是马或王，则不接受该移动
                if (p.type() == Piece::Knight || p.type() == Piece::King) {
                    accept = false;
                    break;
                }
                // 如果敌方棋子是兵
                else if (p.type() == Piece::Pawn) {
                    // 计算兵的移动方向
                    auto from = positions[pos];
                    auto to = p.coord();
                    Direction d {char(to[R] - from[R]), char(to[F] - from[F])};
                    // 如果兵的移动方向是左斜或右斜，则不接受该移动
                    if (d == pawnAttackLeft || d == pawnAttackRight) {
                        accept = false;
                        break;
                    }
                }
                // 如果敌方棋子是其他类型
                else {
                    // 计算敌方棋子的移动方向
                    auto from = positions[pos];
                    auto to = p.coord();
                    Direction d = normalize({char(to[R] - from[R]), char(to[F] - from[F])});
                    // 沿着移动方向遍历直到遇到障碍物
                    auto reached = traverse(pos, d, Find(state.board));
                    // 如果敌方棋子可以到达新位置，则不接受该移动
                    if (p.pos() == reached) {
                        accept = false;
                        break;
                    }
                }
            }
        }
        // 如果接受该移动，则将新位置添加到允许移动的位置集合中
        if (accept) m_allowed.insert(pos);
    }
// 返回棋子名称
const char* Piece::name() const {
    // 检查棋子名称数量与棋子类型数量是否匹配
    static_assert(pieceNames.size() == Piece::NUM_PIECES, "Mismatch between piece names and types");
    // 返回对应棋子类型的名称
    return pieceNames[m_type];
}

// 返回棋子的缩写
char Piece::initial() const {
    // 检查黑色棋子缩写数量与棋子类型数量是否匹配
    static_assert(blackShort.size() == Piece::NUM_PIECES, "Mismatch between piece names and types");
    // 检查白色棋子缩写数量与棋子类型数量是否匹配
    static_assert(whiteShort.size() == Piece::NUM_PIECES, "Mismatch between piece names and types");
    // 根据棋子颜色返回对应棋子类型的缩写
    return m_color ? blackShort[m_type] : whiteShort[m_type];
}

// 标记棋子需要更新
void Piece::invalidate() {
    // 设置更新标志为真
    m_update = true;
}

// 返回棋子的坐标
const char* Piece::coord() const {
    // 如果棋子位置无效，则返回空字符串
    if (m_pos == INVALID_POS) return "";
    // 返回对应位置的坐标
    return positions[m_pos];
}

// 判断棋子是否可以到达指定位置
bool Piece::canReach(char pos) const {
    // 判断是否符合移动规则并且目标位置在允许移动的位置集合中
    return movePattern(pos) && m_allowed.count(pos);
}

// 拿起棋子
void Piece::take() {
    // 将棋子位置设置为无效位置
    m_pos = INVALID_POS;
    // 清空允许移动的位置集合
    m_allowed = {};
}

// 初始化棋盘状态
State::State()
    : blacks {
        // 初始化黑色棋子的位置和允许移动的位置
        {Piece::Black, "a7"_P, {"a5"_P, "a6"_P} },
        {Piece::Black, "b7"_P, {"b5"_P, "b6"_P} },
        {Piece::Black, "c7"_P, {"c5"_P, "c6"_P} },
        {Piece::Black, "d7"_P, {"d5"_P, "d6"_P} },
        {Piece::Black, "e7"_P, {"e5"_P, "e6"_P} },
        {Piece::Black, "f7"_P, {"f5"_P, "f6"_P} },
        {Piece::Black, "g7"_P, {"g5"_P, "g6"_P} },
        {Piece::Black, "h7"_P, {"h5"_P, "h6"_P} },
        {Piece::Black, "a8"_P},
        {Piece::Black, "b8"_P, {"a6"_P, "c6"_P} },
        {Piece::Black, "c8"_P},
        {Piece::Black, "d8"_P},
        {Piece::Black, "e8"_P},
        {Piece::Black, "f8"_P},
        {Piece::Black, "g8"_P, {"f6"_P, "h6"_P} },
        {Piece::Black, "h8"_P},
    }
    // 定义白棋的初始位置和移动方式
    , whites {
        {Piece::White, "a2"_P, {"a3"_P, "a4"_P} },
        {Piece::White, "b2"_P, {"b3"_P, "b4"_P} },
        {Piece::White, "c2"_P, {"c3"_P, "c4"_P} },
        {Piece::White, "d2"_P, {"d3"_P, "d4"_P} },
        {Piece::White, "e2"_P, {"e3"_P, "e4"_P} },
        {Piece::White, "f2"_P, {"f3"_P, "f4"_P} },
        {Piece::White, "g2"_P, {"g3"_P, "g4"_P} },
        {Piece::White, "h2"_P, {"h3"_P, "h4"_P} },
        {Piece::White, "a1"_P},
        {Piece::White, "b1"_P, {"a3"_P, "c3"_P} },
        {Piece::White, "c1"_P},
        {Piece::White, "d1"_P},
        {Piece::White, "e1"_P},
        {Piece::White, "f1"_P},
        {Piece::White, "g1"_P, {"f3"_P, "h3"_P} },
        {Piece::White, "h1"_P},
    }
    // 定义棋盘，包括白棋和黑棋的位置
    , board {{
        &whites[ 8],  &whites[ 9],  &whites[10],  &whites[11],  &whites[12],  &whites[13],  &whites[14],  &whites[15],
        &whites[ 0],  &whites[ 1],  &whites[ 2],  &whites[ 3],  &whites[ 4],  &whites[ 5],  &whites[ 6],  &whites[ 7],
        nullptr,      nullptr,      nullptr,      nullptr,      nullptr,      nullptr,      nullptr,      nullptr,
        nullptr,      nullptr,      nullptr,      nullptr,      nullptr,      nullptr,      nullptr,      nullptr,
        nullptr,      nullptr,      nullptr,      nullptr,      nullptr,      nullptr,      nullptr,      nullptr,
        nullptr,      nullptr,      nullptr,      nullptr,      nullptr,      nullptr,      nullptr,      nullptr,
        &blacks[ 0],  &blacks[ 1],  &blacks[ 2],  &blacks[ 3],  &blacks[ 4],  &blacks[ 5],  &blacks[ 6],  &blacks[ 7],
        &blacks[ 8],  &blacks[ 9],  &blacks[10],  &blacks[11],  &blacks[12],  &blacks[13],  &blacks[14],  &blacks[15],
    }}
// Chessboard 类的默认构造函数，初始化 m_state 指针为新的 State 对象
Chessboard::Chessboard()
    : m_state(new State())
{
    // 调用 setGrammar 函数
    setGrammar();
}

// Chessboard 类的析构函数，默认实现
Chessboard::~Chessboard() = default;

// 设置提示信息的函数，参数为 prompt 字符串的引用
void Chessboard::setPrompt(const std::string& prompt) {
    // 将参数 prompt 赋值给成员变量 m_prompt
    m_prompt = prompt;
    // 调用 setGrammar 函数
    setGrammar();
}

// 设置语法规则的函数
void Chessboard::setGrammar() {
    // 清空语法规则
    m_grammar.clear();

    // 定义一个字符串变量 result
    std::string result;
    // 如果 m_prompt 为空
    if (m_prompt.empty()) {
        // 添加默认的移动语法规则到 result
        result += "move ::= \" \" ((piece | frompos) \" \" \"to \"?)? topos\n";
        //result += "move ::= \" \" frompos \" \" \"to \"? topos\n";
    }
    else {
        // 添加带有提示信息的移动语法规则到 result
        result += "move ::= prompt \" \" frompos \" \" \"to \"? topos\n"
        "prompt ::= \" " + m_prompt + "\"\n";
    }

    // 定义一些集合和变量
    std::set<Piece::Types> pieceTypes;
    std::set<char> from_pos;
    std::set<char> to_pos;
    // 根据当前移动次数选择黑子或白子
    auto& pieces =  m_moveCounter % 2 ? m_state->blacks : m_state->whites;
    std::set<size_t> flags;
    // 遍历棋子
    for (auto& p : pieces) {
        // 如果允许移动的位置为空，则继续下一个棋子
        if (p.allowed().empty()) continue;
        bool addPiece = false;
        // 如果不处于将军状态或者是国王，则将允许移动的位置加入到 to_pos 集合中
        if (!m_inCheck || p.type() == Piece::King) {
            to_pos.insert(p.allowed().begin(), p.allowed().end());
            addPiece = !p.allowed().empty();
        }
        else {
            // 如果处于将军状态，则只允许移动到被允许的位置
            for (auto move : p.allowed()) {
                if (m_allowedInCheck.count(move)) {
                    to_pos.insert(move);
                    addPiece = true;
                }
            }
        }
        // 如果该棋子可以移动，则将其类型和位置加入到相应的集合中
        if (addPiece) {
            pieceTypes.insert(p.type());
            from_pos.insert(p.pos());
        }
    }
    // 如果没有可移动的棋子，则返回
    if (pieceTypes.empty()) return;

    // 添加棋子类型的语法规则到 result
    result += "piece ::= (";
    for (auto& p : pieceTypes) result += " \"" + std::string(pieceNames[p]) + "\" |";
    result.pop_back();
    result += ")\n\n";

    // 添加起始位置的语法规则到 result
    result += "frompos ::= (";
    for (auto& p : from_pos) result += " \"" + std::string(positions[p]) + "\" |";
    result.pop_back();
    result += ")\n";

    // 添加目标位置的语法规则到 result
    result += "topos ::= (";
    for (auto& p : to_pos) result += " \"" + std::string(positions[p]) + "\" |";
    # 移除字符串末尾的一个字符（假设该字符是括号）
    result.pop_back();
    # 在字符串末尾添加一个右括号和换行符
    result += ")\n";

    # 将 result 移动到 m_grammar 中，result 不再拥有值
    m_grammar = std::move(result);
}

// 将棋盘状态转换为字符串表示
std::string Chessboard::stringifyBoard() {
    // 初始化结果字符串，预留足够的空间
    std::string result;
    result.reserve(16 + 2 * 64 + 16);
    // 遍历每个列，添加列标识到结果字符串
    for (char rank = 'a'; rank <= 'h'; ++rank) {
        result.push_back(rank);
        result.push_back(' ');
    }
    result.back() = '\n';
    // 遍历每个行，添加每个棋子的初始字符到结果字符串
    for (int i = 7; i >= 0; --i) {
        for (int j = 0; j < 8; ++j) {
            auto p = m_state->board[i * 8 + j];
            if (p) result.push_back(p->initial());
            else result.push_back((i + j) % 2 ? '.' : '*');
            result.push_back(' ');
        }
        result.push_back('0' + i + 1);
        result.push_back('\n');
    }
    return result;
}

// 处理棋盘上的移动命令
std::string Chessboard::process(const std::string& command) {
    // 记录处理开始时间
    const auto t_start = std::chrono::high_resolution_clock::now();
    // 确定当前颜色
    auto color = Piece::Colors(m_moveCounter % 2);
    Piece* piece = nullptr;
    auto pos_to = INVALID_POS;
    // 解析移动命令
    if (!parseCommand(command, piece, pos_to)) return "";

    auto pos_from = piece->pos();

    // 执行移动
    if (!move(*piece, pos_to)) return "";

    // 标记更新
    flagUpdates(pos_from, pos_to);

    // 检查是否将军
    detectChecks();

    // 获取敌方棋子，只需检查敌方的移动
    auto& enemyPieces = color ? m_state->whites : m_state->blacks;
    for (auto& p : enemyPieces) p.reinit(*m_state);

    // 生成移动结果字符串
    std::string result = {positions[pos_from][R], positions[pos_from][F], '-', positions[pos_to][R], positions[pos_to][F]};
    ++m_moveCounter;
    setGrammar();
    // 计算处理时间
    const auto t_end = std::chrono::high_resolution_clock::now();
    auto t_ms = std::chrono::duration_cast<std::chrono::milliseconds>(t_end - t_start).count();
    // 打印移动信息
    fprintf(stdout, "%s: Move '%s%s%s', (t = %d ms)\n", __func__, "\033[1m", result.data(), "\033[0m", (int) t_ms);
    if (m_grammar.empty()) result.push_back('#');
    return result;
}

// 解析移动命令
bool Chessboard::parseCommand(const std::string& command, Piece*& piece, char& pos_to) {
    auto color = Piece::Colors(m_moveCounter % 2);
    # 打印带有颜色的命令信息
    fprintf(stdout, "%s: Command to %s: '%s%.*s%s'\n", __func__, (color ? "Black" : "White"), "\033[1m", int(command.size()), command.data(), "\033[0m");

    # 如果命令为空，则返回 false
    if (command.empty()) return false;
    # 将命令按空格分割成 tokens
    auto tokens = split(command, ' ');
    auto pos_from = INVALID_POS;
    auto type = Piece::Types::NUM_PIECES;
    # 如果 tokens 的大小为 1，则设置 type 为 Pawn，pos_to 为 tokens 的第一个元素转换后的位置
    if (tokens.size() == 1) {
        type = Piece::Types::Pawn;
        pos_to = strToPos(tokens.front());
    }
    else {
        # 否则，将 tokens 的第一个元素转换后的位置赋给 pos_from
        pos_from = strToPos(tokens.front());
        # 如果 pos_from 为无效位置，则将 type 设置为 tokens 的第一个元素转换后的类型
        if (pos_from == INVALID_POS) type = Piece::Types(strToType(tokens.front()));
        # 将 tokens 的最后一个元素转换后的位置赋给 pos_to
        pos_to = strToPos(tokens.back());
    }
    # 如果 pos_to 为无效位置，则返回 false
    if (pos_to == INVALID_POS) return false;
    # 如果 pos_from 为无效位置
    if (pos_from == INVALID_POS) {
        # 如果 type 为 NUM_PIECES，则返回 false
        if (type == Piece::Types::NUM_PIECES) return false;
        # 根据颜色选择对应的棋子集合
        auto& pieces = color ? m_state->blacks : m_state->whites;
        # 遍历棋子集合，找到符合条件的棋子
        for (auto& p : pieces) {
            if (p.type() == type && p.canReach(pos_to)) {
                pos_from = p.pos();
                break;
            }
        }
    }
    # 如果 pos_from 为无效位置，则返回 false
    if (pos_from == INVALID_POS) return false;
    # 如果 pos_from 对应的位置为空，则返回 false
    if (m_state->board[pos_from] == nullptr) return false;
    # 将 pos_from 对应的棋子赋给 piece
    piece = m_state->board[pos_from];
    # 如果棋子颜色与当前颜色不匹配，则返回 false
    if (piece->color() != color) return false;
    # 返回 true
    return true;
// 标记棋盘上的更新，根据移动的起始位置和目标位置
void Chessboard::flagUpdates(char pos_from, char pos_to) {
    // 获取当前移动的颜色
    auto color = Piece::Colors(m_moveCounter % 2);
    // 根据颜色选择敌方棋子集合和己方棋子集合
    auto& enemyPieces = color ? m_state->whites : m_state->blacks;
    auto& ownPieces = color ? m_state->blacks : m_state->whites;
    // 遍历敌方棋子集合
    for (auto& p : enemyPieces) {
        // 如果敌方棋子可以移动到目标位置或起始位置，则更新针对该棋子的信息
        if (p.movePattern(pos_to) || p.movePattern(pos_from)) {
            updatePins(p);
            p.invalidate();
        }
    }

    // 遍历己方棋子集合
    for (auto& p : ownPieces) {
        // 如果己方棋子可以移动到目标位置或起始位置，则更新针对该棋子的信息
        if (p.movePattern(pos_to) || p.movePattern(pos_from)) {
            updatePins(p);
            p.invalidate();
        }
    }
}

// 更新棋子的针对信息
void Chessboard::updatePins(Piece& piece) {
    // 如果棋子是兵、马或国王，则直接返回
    if (piece.type() == Piece::Pawn || piece.type() == Piece::Knight || piece.type() == Piece::King) return;
    // 根据棋子颜色选择敌方棋子集合和针对集合
    auto& enemyPieces = piece.color() ? m_state->whites : m_state->blacks;
    auto& enemyPins = piece.color() ? m_state->whitePins : m_state->blackPins;
    auto& king = enemyPieces.k;
    // 查找是否有针对该棋子的针对信息
    auto it = std::find_if(enemyPins.begin(), enemyPins.end(), [&] (const Pin& pin) { return pin.pinner == &piece; });
    // 如果存在针对信息，则使被针对的棋子无效，并移除针对信息
    if (it != enemyPins.end()) {
        it->pinned->invalidate();
        enemyPins.erase(it);
    }
    // 如果棋子可以移动到敌方国王的位置
    if (piece.movePattern(king.pos())) {
        // 计算移动方向
        auto to = positions[king.pos()];
        auto from = piece.coord();
        Direction d = normalize({char(to[R] - from[R]), char(to[F] - from[F]});

        // 遍历棋盘，查找是否有棋子可以攻击敌方国王
        auto reached = traverse(piece.pos(), d, Find(m_state->board));
        auto foundPiece = m_state->board[reached];
        if (&king == foundPiece) {
            // 如果找到敌方国王，则标记为将军
            king.invalidate();
        }
        else if (foundPiece && foundPiece->color() != piece.color()) {
            // 如果找到敌方棋子，标记为针对信息
            reached = traverse(reached, d, Find(m_state->board));
            if (&king == m_state->board[reached]) {
                enemyPins.push_back({d, &piece, foundPiece});
                foundPiece->invalidate();
            }
        }
    }
}

// 检测是否有将军
void Chessboard::detectChecks() {
    // 获取当前移动的颜色
    auto color = Piece::Colors(m_moveCounter % 2);
    // 根据颜色选择敌方棋子和己方棋子
    auto& enemyPieces = color ? m_state->whites : m_state->blacks;
    auto& ownPieces = color ? m_state->blacks : m_state->whites;
    // 获取敌方国王
    auto& king = enemyPieces.k;
    // 根据颜色选择左侧和右侧的兵的攻击方向
    auto& pawnAttackLeft = color ? SW : NW;
    auto& pawnAttackRight = color ? SE : NE;
    // 遍历己方棋子
    for (auto& p : ownPieces) {
        // 如果棋子不能移动到敌方国王的位置，则继续下一个棋子
        if (!p.movePattern(king.pos())) continue;
        // 获取目标位置和起始位置
        auto to = positions[king.pos()];
        auto from = p.coord();

        // 如果棋子类型为Knight
        if (p.type() == Piece::Knight) {
            // 如果不处于将军状态，则将当前棋子位置加入允许的将军位置集合
            if (!m_inCheck) {
                m_allowedInCheck = { p.pos() };
            }
            else {
                m_allowedInCheck.clear();
            }
            // 设置将军状态为true
            m_inCheck = true;
        }
        // 如果棋子类型为Pawn
        else if (p.type() == Piece::Pawn) {
            // 计算棋子移动的方向
            Direction d {char(to[R] - from[R]), char(to[F] - from[F])};
            // 如果移动方向为左侧或右侧攻击方向
            if (d == pawnAttackLeft || d == pawnAttackRight) {
                // 如果不处于将军状态，则将当前棋子位置加入允许的将军位置集合
                if (!m_inCheck) {
                    m_allowedInCheck = { p.pos() };
                }
                else {
                    m_allowedInCheck.clear();
                }
                // 设置将军状态为true
                m_inCheck = true;
            }
        }
        // 其他类型的棋子
        else {
            // 标准化移动方向
            Direction d = normalize({char(to[R] - from[R]), char(to[F] - from[F])});
            std::set<char> tmp;
            // 沿着移动方向遍历棋盘，找到是否可以直接攻击到敌方国王的位置
            auto pos = traverse(p.pos(), d, Add(m_state->board, tmp, king.color()));
            // 如果可以直接攻击到敌方国王的位置
            if (pos == king.pos()) {
                tmp.insert(p.pos());
                // 如果不处于将军状态，则将当前棋子位置加入允许的将军位置集合
                if (!m_inCheck) {
                    m_allowedInCheck = std::move(tmp);
                }
                else {
                    m_allowedInCheck.clear();
                }
                // 设置将军状态为true
                m_inCheck = true;
            }
        }
    }
// 棋子移动函数，接受一个棋子对象和目标位置作为参数
bool Chessboard::move(Piece& piece, char pos_to) {
    // 获取棋子可以移动到的位置集合
    auto& allowed = piece.allowed();

    // 如果目标位置不在允许移动的位置集合中，或者当前处于被将军状态且移动的棋子不是国王且目标位置不在允许在将军状态下移动的位置集合中，则返回移动失败
    if (allowed.count(pos_to) == 0 || (m_inCheck && piece.type() != Piece::King && m_allowedInCheck.count(pos_to) == 0)) return false;
    // 如果目标位置上有棋子且颜色与移动的棋子相同，则返回移动失败
    if (m_state->board[pos_to] && m_state->board[pos_to]->color() == piece.color()) return false;
    // 如果目标位置上有棋子，则将其移除
    if (m_state->board[pos_to]) m_state->board[pos_to]->take();
    // 将当前位置的棋子置空
    m_state->board[piece.pos()] = nullptr;
    // 将棋子移动到目标位置
    m_state->board[pos_to] = &piece;
    // 更新棋子的位置信息
    piece.setPos(pos_to);

    // 重置将军状态和允许在将军状态下移动的位置集合
    m_inCheck = false;
    m_allowedInCheck.clear();

    // 返回移动成功
    return true;
}
```