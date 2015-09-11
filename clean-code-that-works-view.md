# Simple Design
liyongshun, zte.com.cn Inc.

![cover](http://www.dvd-ppt-slideshow.com/images/ppt-background/background-6.jpg)
***

## 引言

** 念念不忘,必有回响,有一口气,点一盏灯 ** -- 《一代宗师》


***

## 翻转棋([Reversi](https://en.wikipedia.org/wiki/Reversi))
黑白棋，又叫反棋（Reversi），游戏通过相互翻转对方的棋子，最后以棋盘上谁的棋子多来判断胜负。它的游戏规则简单，因此上手很容易，但是它的变化又非常复杂。
给定棋局（初始棋盘）：  ![Othello Board][othello-init]
[othello-init]: images/clean-code-that-works/othelloInit.png

***

## 需求
 1. 求解所有黑棋可落子位置（e.g. 黑棋可落子位置为：**d3, c4, f5, e6 ** ）
 2. 在某一位置落子后，翻转吃掉的子（e.g. 落子到 **d3** 后，白子 **d4** 被吃掉）
 3. 打印出所有可能落子后对应的棋局
 	![Dark Moves][othello-dark-moves] ![After dark play][after-dark-play]
[othello-dark-moves]: images/clean-code-that-works/darkMoves.png
[after-dark-play]: images/clean-code-that-works/afterDarkPlay.png

***

## 如何启动
“先别急着动手，我们需要抽象出语义模型，找出问题背后的本质概念”；
“还磨叽什么，直接开写，TDD，Write a test that fails”。
我的理解：
- 理清业务需求，根据经验，找到当前自己能找到的最好设计
- 演进式的设计，通过Fail-Pass-Refactor方式逐渐找到问题的本质

***

## 我们快速把需求翻译成测试用例
```cpp
TEST_F(ReversiTest, should_get_a_board_from_reversi_equals_a_init_board)
{
    const Board EXPECT_INIT_BOARD;
    ASSERT_EQ(EXPECT_INIT_BOARD, Reversi().getBoard());
}

TEST_F(ReversiTest, should_get_available_positions_given_a_valid_position_in_the_board)
{
    const Positions EXPECT_POSITIONS_OF_e4 = {c4, e6};
    ASSERT_EQ(EXPECT_POSITIONS_OF_e4, Reversi().gitAvailablePositions(e4));

    const Positions EXPECT_POSITIONS_OF_d5 = {d3, f5};
    ASSERT_EQ(EXPECT_POSITIONS_OF_d5, Reversi().gitAvailablePositions(d5));
}

struct ReversiWithSpecificSetTest : ReversiTest
{
    ReversiWithSpecificSetTest()
    {
        buildSet();
        reversi.refresh(set);
    }

    void buildSet()
    {
        set.place(Positions{a1, c3, e3, f4, e8}, BLACK);
        set.place(Positions{a2, d4, e4, e5, f5, b8, c8, d8}, WHITE);
    }

    void ASSERT_BOARD(const Position move, const Positions& positionChanged)
    {
        Board EXPECT_SET(set);
        EXPECT_SET.place(positionChanged, BLACK);
        ASSERT_EQ(EXPECT_SET, reversi.capture(move));
        EXPECT_SET.print();
    }

protected:
    Board set;
};

namespace
{
    const Positions EXPECT_POSITIONS_ALL_BLACKS = {a3, f6, c5, e6, c4, d6, a8};
}

TEST(ReversiWithSpecificSetTest, should_get_all_available_positions_given_a_disk_with_a_specific_set)
{
    Reversi reversi;
    reversi.refresh(set);

    ASSERT_EQ(EXPECT_POSITIONS_ALL_BLACKS, reversi.getAllAvailablePositions(BLACK));
}

TEST(ReversiWithSpecificSetTest, should_turn_over_the_captured_disk_given_a_available_position)
{
    ASSERT_BOARD(a3, Positions{a2, a3});
    ASSERT_BOARD(f6, Positions{d4, e5, f5, f6});
    ASSERT_BOARD(e6, Positions{e4, e5, e6});
    ASSERT_BOARD(c5, Positions{d4, c5});
    ASSERT_BOARD(c4, Positions{c4, d4, e4});
    ASSERT_BOARD(d6, Positions{d6, e5});
    ASSERT_BOARD(a8, Positions{a8, b8, c8, d8});
}
```

***

## 回到问题, 分析下有哪些概念
+ 棋盘（Board）: 棋盘由8X8方格(Grid)组成，每个空格可以有三种状态，空闲，放黑棋，放白棋，默认棋盘在e4, d5位置(Position)放置黑棋，在d4, e5位置放置白棋
+ 棋子（Disk）：黑棋（Black），白棋（White）
+ 规则（Rules）:
	Rule-1. 一方在某个位置落子必须能够夹对方一个或多个连续棋子
	Rule-2. 可以横着夹，竖着夹，斜着夹
	Rule-3. 被夹住的棋子需要进行翻转
+ 位置控制：可以向上，下，左，右，左上，左下，右上，右下 8个方向进行搜索，在棋盘上搜索空间最大为7，超出则越界。

***

## 实践：测试驱动开发
![Tdd][tdd]
[tdd]: images/clean-code-that-works/tdd.gif
***

## TDD
#### 3.2.1 Write a test that fails - FAIL
创建第一个文件BoardTest.cpp,完成一个测试用例:
```cpp
//BoardTest.cpp
#include "gtest/gtest.h"
#include "Board.h"

TEST(BoardTest, should_init_board_with_black_in_e4_d5_and_white_in_d4_e5)
{
    Board board;
    ASSERT_EQ(BLACK, board.at(e4));
    ASSERT_EQ(BLACK, board.at(d5));
    ASSERT_EQ(WHITE, board.at(d4));
    ASSERT_EQ(WHITE, board.at(e5));
}
```
#### 3.2.2 Make the code work - PASS
```cpp
#ifndef _INCL_BOARD_H_
#define _INCL_BOARD_H_

enum Position
{
    a1, b1, c1, d1, e1, f1, g1, h1,
    a2, b2, c2, d2, e2, f2, g2, h2,
    a3, b3, c3, d3, e3, f3, g3, h3,
    a4, b4, c4, d4, e4, f4, g4, h4,
    a5, b5, c5, d5, e5, f5, g5, h5,
    a6, b6, c6, d6, e6, f6, g6, h6,
    a7, b7, c7, d7, e7, f7, g7, h7,
    a8, b8, c8, d8, e8, f8, g8, h8,
};

enum GridStatus { BLACK, WHITE };

struct Board
{
    Board()
    {
        for(GridStatus& status: grids)
        {
            status = BLACK;
        }

        grids[d4] = WHITE;
        grids[e5] = WHITE;
    }
    
    GridStatus at(Position p) const
    {
        return grids[p];
    }
private:
    GridStatus grids[h8+1];
};

#endif
```
看到我们期待的画面
> 
[ RUN      ] BoardTest.should_init_board_with_black_in_e4_d5_and_white_in_d4_e5
[       OK ] BoardTest.should_init_board_with_black_in_e4_d5_and_white_in_d4_e5 (0 ms)
[----------] 1 test from BoardTest (0 ms total)
 

#### 3.2.3 Eliminate redundancy - REFACTOR
整体看下，代码中没有明显的重复，看下CleanCode的关注点，发现还时有一些问题，开启我们的重构模式吧  
**命名**  
我们根据题目的语义，从测试驱动角度，名字已经比较OK了，但是文件里面依然有些问题：h8+1 可能让人迷惑，另外作为数组最大值，可以预见到后面还会被使用。
```cpp
GridStatus grids[h8+1];
```
改为：
```cpp
enum {MAX_GRID_NUM = h8+1 };
GridStatus grids[MAX_GRID_NUM];
```
**物理设计**  
1. Position与Board为两个独立概念，应将单独拆分出去  
2. Board头文件*构造函数*及*at*方法inline实现，目前不是性能瓶颈，我们应将其移入源文件  
3. GridStatus目前与Board关系紧密，暂时不动  

```cpp
//Position.h
#ifndef _INCL_POSITION_H_
#define _INCL_POSITION_H_

enum Position
{
    a1, b1, c1, d1, e1, f1, g1, h1,
    a2, b2, c2, d2, e2, f2, g2, h2,
    a3, b3, c3, d3, e3, f3, g3, h3,
    a4, b4, c4, d4, e4, f4, g4, h4,
    a5, b5, c5, d5, e5, f5, g5, h5,
    a6, b6, c6, d6, e6, f6, g6, h6,
    a7, b7, c7, d7, e7, f7, g7, h7,
    a8, b8, c8, d8, e8, f8, g8, h8,
};

#endif
```
```cpp
//Board.h
#ifndef _INCL_BOARD_H_
#define _INCL_BOARD_H_

#include "Position.h"

enum GridStatus { BLACK, WHITE };

struct Board
{
    Board();
    GridStatus at(Position p) const;

private:
    enum {MAX_GRID_NUM = h8+1 };
    GridStatus grids[MAX_GRID_NUM];
};

#endif

```
```cpp
//Board.cpp
#include "Board.h"

Board::Board()
{
    for(GridStatus& status: grids)
    {
        status = BLACK;
    }

    grids[d4] = WHITE;
    grids[e5] = WHITE;

}

GridStatus Board::at(Position p) const
{
    return grids[p];
}
```

运行，测试通过（**本轮TDD完成，后续我们一直按照三步走，不在赘述**）

**FAIL:**  
```cpp
#include "gtest/gtest.h"
#include "Board.h"

TEST(BoardTest, should_init_board_with_black_in_e4_d5_and_white_in_d4_e5)
{
    Board board;
    ASSERT_EQ(BLACK, board.at(e4));
    ASSERT_EQ(BLACK, board.at(d5));
    ASSERT_EQ(WHITE, board.at(d4));
    ASSERT_EQ(WHITE, board.at(e5));
}

TEST(BoardTest, should_not_occupied_except_e4_d5_d4_e5)
{
    Board board;
    ASSERT_FALSE(board.isOccupied(a1));
    ASSERT_FALSE(board.isOccupied(h8));
}

TEST(BoardTest, should_return_ture_given_a1_to_h8)
{
    Board board;
    ASSERT_TRUE(board.onBoard(a1));
    ASSERT_TRUE(board.onBoard(h8));
}

TEST(BoardTest, should_return_false_given_out_of_range_of_a1_to_h8)
{
    Board board;
    Position lessThana1 = static_cast<Position>(a1-1);
    Position moreThanh8 = static_cast<Position>(h8+1);
    ASSERT_FALSE(board.onBoard(lessThana1));
    ASSERT_FALSE(board.onBoard(moreThanh8));
    ASSERT_FALSE(board.isOccupied(lessThana1));
    ASSERT_FALSE(board.isOccupied(moreThanh8));
}

TEST(BoardTest, should_place_disk_given_a_positon_in_the_board)
{
    Board board;
    board.place(a1, WHITE);
    ASSERT_TRUE(board.isOccupied(a1));
    ASSERT_EQ(WHITE, board.at(a1));
}

TEST(BoardTest, should_turn_over_given_a_valied_positon_which_is_occupided)
{
    Board board;
    board.place(a1, WHITE);
    board.turnOver(a1);
    ASSERT_TRUE(board.isOccupied(a1));
    ASSERT_EQ(B, board.at(a1));
}
```

**PASS:**  
```cpp
//Board.h
#ifndef _INCL_BOARD_H_
#define _INCL_BOARD_H_

#include "Position.h"

enum GridStatus { EMPTY, BLACK, WHITE };

struct Board
{
    Board();
    void place(Position, GridStatus);
    void turnOver(Position);
    
    GridStatus at(Position p) const;
    bool isOccupied(Position) const;
    bool onBoard(Position) const;
private:
    enum {MAX_GRID_NUM = h8+1 };
    GridStatus grids[MAX_GRID_NUM];
};

#endif
//Board.cpp 
...
```
**REFACTOR**  
我们发现Board中```grids[e4] = BLACK;```,```grids[p] == BLACK```等语义不是很好，应该抽象出Grid类，专门处理棋盘格状态

```cpp
#include "gtest/gtest.h"
#include "Grid.h"

TEST(GirdTest, should_not_occupied_in_init_state)
{
    Grid grid;
    ASSERT_FALSE(grid.isOccupied());
}

TEST(GirdTest, should_occupied_by_black_disk_when_place_black)
{
    Grid grid;
    grid.place(BLACK);
    ASSERT_TRUE(grid.isOccupied());
    ASSERT_TRUE(grid.isBlack());
}

TEST(GirdTest, should_occupied_by_white_disk_when_place_white)
{
    Grid grid;
    grid.place(WHITE);
    ASSERT_TRUE(grid.isOccupied());
    ASSERT_TRUE(grid.isWhite());
}

TEST(GirdTest, should_not_occupied_when_reset)
{
    Grid grid;
    grid.place(WHITE);
    grid.reset();
    ASSERT_FALSE(grid.isOccupied());
}

TEST(GirdTest, should_turn_over_when_grid_is_occupided)
{
    Grid grid;
    grid.turnOver();
    ASSERT_FALSE(grid.isOccupied());

    grid.place(WHITE);
    grid.turnOver();
    ASSERT_TRUE(grid.isBlack());
    grid.turnOver();
    ASSERT_TRUE(grid.isWhite());
}

//Grid.h
#ifndef _INCL_GRID_H_
#define _INCL_GRID_H_

enum GridStatus { EMPTY, BLACK, WHITE };

struct Grid
{
    Grid();

    void place(GridStatus);
    void reset();
    void turnOver();

    bool isOccupied() const;
    bool isBlack() const;
    bool isWhite() const;
private:
    GridStatus status;
};

#endif

//Grid.cpp
...
//Adapt BoardTest.cpp
#include "gtest/gtest.h"
#include "Board.h"

TEST(BoardTest, should_init_board_with_black_in_e4_d5_and_white_in_d4_e5)
{
    Board board;
    ASSERT_TRUE(board.at(e4).isBlack());
    ASSERT_TRUE(board.at(d5).isBlack());
    ASSERT_TRUE(board.at(d4).isWhite());
    ASSERT_TRUE(board.at(e5).isWhite());
}

TEST(BoardTest, should_not_occupied_except_e4_d5_d4_e5)
{
    Board board;
    ASSERT_FALSE(board.isOccupied(a1));
    ASSERT_FALSE(board.isOccupied(h8));
}

TEST(BoardTest, should_return_ture_given_a1_to_h8)
{
    Board board;
    ASSERT_TRUE(board.onBoard(a1));
    ASSERT_TRUE(board.onBoard(h8));
}

TEST(BoardTest, should_return_false_given_out_of_range_of_a1_to_h8)
{
    Board board;
    Position lessThana1 = static_cast<Position>(a1-1);
    Position moreThanh8 = static_cast<Position>(h8+1);
    ASSERT_FALSE(board.onBoard(lessThana1));
    ASSERT_FALSE(board.onBoard(moreThanh8));
    ASSERT_FALSE(board.isOccupied(lessThana1));
    ASSERT_FALSE(board.isOccupied(moreThanh8));
    ASSERT_FALSE(board.at(lessThana1).isOccupied());
    ASSERT_FALSE(board.at(moreThanh8).isOccupied());
}

TEST(BoardTest, should_place_disk_given_a_positon_in_the_board)
{
    Board board;
    board.place(a1, WHITE);
    ASSERT_TRUE(board.at(a1).isOccupied());
    ASSERT_TRUE(board.at(a1).isWhite());
}

TEST(BoardTest, should_turn_over_given_a_valied_positon_which_is_occupided)
{
    Board board;
    board.place(a1, WHITE);
    board.turnOver(a1);
    ASSERT_TRUE(board.at(a1).isOccupied());
    ASSERT_TRUE(board.at(a1).isBlack());
}

//Board.h
#ifndef _INCL_BOARD_H_
#define _INCL_BOARD_H_

#include "Position.h"
#include "Grid.h"

struct Board
{
    Board();
    void place(Position, GridStatus);
    void turnOver(Position);

    Grid at(Position p) const;
    bool isOccupied(Position) const;
    bool onBoard(Position) const;

private:
    enum {MAX_GRID_NUM = h8+1 };
    Grid grids[MAX_GRID_NUM];
};

#endif

//Board.cpp
...
```

```bool onBoard(Position) const;```不会修改类的成员，所以应该为static

```cpp
//Board.h
...
static bool onBoard(Position);
...
//Board.cpp
...
bool Board::onBoard(Position p)
{
    return p >= a1 && p <= h8;
}
...
```
```	if( ! onBoard(p)) return Grid();```当给定位置无效时，每次都会构造用一个对象，使用static NullObject即可.
```cpp
Grid Board::at(Position p) const
{
	if( ! onBoard(p)) return Grid();
    return grids[p];
}
```
改为：
```cpp
Grid Board::at(Position p) const
{
    if( ! onBoard(p))
    {
        static Grid nullGrid;
        return nullGrid;
    }
    
    return grids[p];
}

```
到目前为止，```Board```类基本功能已经完成，后续如果有新的需求，根据需要再增加.


接下来我们实现```Positions```类，```Positions```类为```Position```的集合，所以他具有集合类拥有的通用接口
**FAIL**
```cpp
#include "gtest/gtest.h"
#include "Positions.h"

TEST(PositionsTest, should_empty_given_a_init_positions)
{
    ASSERT_TRUE(Positions().isEmpty());
}

TEST(PositionsTest, should_init_positions_given_position_lists)
{
    Positions one = {a1, b2};
    Positions another{c3, d4};
}

TEST(PositionsTest, should_equals_given_a_same_position_list_which_the_order_is_not_sensitive)
{
    Positions one {a1, b2, c3};
    Positions another {b2, c3, a1};
    
    ASSERT_EQ(one, another);
    ASSERT_FALSE(one.isEmpty());
}

TEST(PositionsTest, should_not_equals_given_different_position_list)
{
    Positions one {a1, b2};
    Positions another {a1};
    Positions third {b2, a2};
    
    ASSERT_NE(one, another);
    ASSERT_NE(one, third);
    ASSERT_NE(another, third);
}

TEST(PositionsTest, should_push_position_to_positions)
{
    Positions EXPECT {a1, b2};

    Positions positions;
    ASSERT_NE(EXPECT, positions);

    positions.push(a1);
    positions.push(b2);

    ASSERT_EQ(EXPECT, positions);
}

TEST(PositionsTest, should_pop_the_position_given_not_empty_positions)
{
    Positions GIVEN {a1, b2};
    ASSERT_FALSE(GIVEN.isEmpty());

    ASSERT_EQ(a1, GIVEN.pop());
    ASSERT_EQ(b2, GIVEN.pop());
    
    ASSERT_TRUE(GIVEN.isEmpty());
}

TEST(PositionsTest, should_clear_the_positions_given_not_empty_positions)
{
    Positions GIVEN {a1, b2};
    ASSERT_FALSE(GIVEN.isEmpty());

    GIVEN.clear();

    ASSERT_TRUE(GIVEN.isEmpty());
}
```

**PASS**
```cpp
//Positions.h
#ifndef _INCL_OTHELLO_POSITIONS_H_
#define _INCL_OTHELLO_POSITIONS_H_

#include "Position.h"
#include <list>

struct Positions
{
    Positions(std::initializer_list<Position> list = {});  

    void push(Position);
    Position pop();
    void clear();

    bool isEmpty() const; 
    bool operator==(const Positions& rhs) const;
    bool operator!=(const Positions& rhs) const;

private:
    bool contains(Position p) const;

private:
    std::list<Position> list;

};

#endif

//Positions.cpp
...
```
**REFACTOR**
```CPP
Positions::Positions(std::initializer_list<Position> positions)
{
    for(auto position : positions)
    {
        list.push_back(position);
    }
}

void Positions::push(Position p)
{
    list.push_back(p);
}
```
```list.push_back(p);``` 存在重复，将其消除:
```cpp
Positions::Positions(std::initializer_list<Position> positions)
{
    for(auto position : positions)
    {
        push(position);
    }
}
```
---
游戏规则及位置控制部分，我们目前还不清楚具体需求，我们先从游戏本身入手，发现职责不明晰时再进行抽离
**FAIL**
```cpp
#include "gtest/gtest.h"
#include "Reversi.h"

struct ReversiTest : testing::Test
{
protected:
    Reversi reversi;
    
};

TEST_F(ReversiTest, should_get_a_board_from_reversi_equals_a_init_board)
{
    const Board EXPECT_INIT_BOARD;
    ASSERT_EQ(EXPECT_INIT_BOARD, Reversi().getBoard());
}

TEST_F(ReversiTest, should_get_available_positions_given_a_valid_position_in_the_board)
{
    const Positions EXPECT_POSITIONS_OF_e4 = {c4, e6};
    ASSERT_EQ(EXPECT_POSITIONS_OF_e4, Reversi().gitAvailablePositions(e4));

    const Positions EXPECT_POSITIONS_OF_d5 = {d3, f5};
    ASSERT_EQ(EXPECT_POSITIONS_OF_d5, Reversi().gitAvailablePositions(d5));
}
```
** PASS ** && ** REFACTOR **
```cpp
#ifndef _INCL_REVERSI_H_
#define _INCL_REVERSI_H_

#include "Board.h"
#include "Positions.h"

struct Reversi
{
    const Board& getBoard() const;
    const Positions& gitAvailablePositions(Position);

private:
    bool hasNext(Position curr, Position moves) const;
    typedef Position (*MoveFun)(Position p);
    void find(Position p, MoveFun);

private:
    Board board;

private:
    Positions availablePositions;
};

#endif

//Reversi.cpp
#include "Reversi.h"

const Board& Reversi::getBoard() const
{
    return board;
}

namespace
{
    const Position INVALID_POSITION = static_cast<Position>(h8+1);

    Position up(Position p)
    {
        return static_cast<Position>(p-8);
    }

    Position down(Position p)
    {
        return static_cast<Position>(p+8);
    }

    bool onARow(Position l, Position r)
    {
        return l/8 == r/8;
    }

    Position left(Position p)
    {
        Position leftPos = static_cast<Position>(p-1);
        return onARow(p, leftPos) ? leftPos : INVALID_POSITION;
    }

    Position right(Position p)
    {
        Position rightPos = static_cast<Position>(p+1);
        return onARow(p, rightPos) ? rightPos : INVALID_POSITION;
    }

    Position leftUp(Position p)
    {
        return left(up(p));
    }

    Position leftDown(Position p)
    {
        return left(down(p));
    }

    Position rightUp(Position p)
    {
        return right(up(p));
    }

    Position rightDown(Position p)
    {
        return right(down(p));
    }
}

bool Reversi::hasNext(Position curr, Position moves) const
{
    return board.isOccupied(moves) && board.at(moves) != board.at(curr);
}

void Reversi::find(Position p, MoveFun move)
{
    if( ! board.isOccupied(p)) return;

    Position next = p;
    do
    {
        next =  move(next);
    }while(hasNext(p, next));

    if( ! board.onBoard(next)) return;

    if( ! board.isOccupied(next) && next != move(p))
    {
        availablePositions.push(next);
    }
}

const Positions& Reversi::gitAvailablePositions(Position p)
{
    availablePositions.clear();

    find(p, up);
    find(p, down);
    find(p, left);
    find(p, right);
    find(p, leftUp);
    find(p, leftDown);
    find(p, rightUp);
    find(p, rightDown);

    return availablePositions;
}
```
至此，需求一实现完成，接下来我们实现需求二
```cpp
struct ReversiWithSpecificSetTest : ReversiTest
{
    ReversiWithSpecificSetTest()
    {
        buildSet();
        reversi.refresh(set);
    }

    void buildSet()
    {
        set.place(Positions{d4, e4, d5, e5}, EMPTY);
        set.place(Positions{a1, c3, e3, f4, e8}, BLACK);
        set.place(Positions{a2, d4, e4, e5, f5, b8, c8, d8}, WHITE);
    }
protected:
    Board set;
};

namespace
{
    const Positions EXPECT_POSITIONS_ALL_BLACKS = {a3, f6, c5, e6, c4, d6, a8};
}

TEST_F(ReversiWithSpecificSetTest, should_get_all_available_positions_given_a_disk_with_a_specific_set)
{
    ASSERT_EQ(EXPECT_POSITIONS_ALL_BLACKS, reversi.getAllAvailablePositions(BLACK));
}

//Reversi.h
...
const Positions& getAllAvailablePositions(GridStatus);
...
//Reversi.cpp
...
void Reversi::refresh(const Board& newBoard)
{
    board = newBoard;
}

const Positions& Reversi::getAllAvailablePositions(GridStatus gridStatus)
{
    allPositions.clear();
    for(int i = a1; i < MAX_GRID_NUM; ++i)
    {
        Position p = static_cast<Position>(i);
        if(board.at(p).getStatus() == gridStatus)
        {
            allPositions = allPositions + gitAvailablePositions(p);
        }
    }

    return allPositions;
}
```

需求二：翻转被夹的棋子：
```cpp

struct ReversiWithSpecificSetTest : ReversiTest
{
 ...
    void ASSERT_BOARD(const Position move, const Positions& positionChanged)
    {
        Board EXPECT_SET(set);
        EXPECT_SET.place(positionChanged, BLACK);
        ASSERT_EQ(EXPECT_SET, reversi.capture(move));
        // EXPECT_SET.print();
        reversi.retract();
    }

protected:
    Board set;
};
...
TEST_F(ReversiWithSpecificSetTest, should_turn_over_the_captured_disk_given_a_available_position)
{
    reversi.getAllAvailablePositions(BLACK);

    ASSERT_BOARD(a3, Positions{a2, a3});
    ASSERT_BOARD(f6, Positions{d4, e5, f5, f6});
    ASSERT_BOARD(e6, Positions{e4, e5, e6});
    ASSERT_BOARD(c5, Positions{d4, c5});
    ASSERT_BOARD(c4, Positions{c4, d4, e4});
    ASSERT_BOARD(d6, Positions{d6, e5});
    ASSERT_BOARD(a8, Positions{a8, b8, c8, d8});
}
```
```cpp
...
struct Reversi
{
...
    const Board& capture(Position move);
    const Board& retract();

private:
...
    bool isReachable(Position from, Position to, MoveFun);
    void turn(Position original, Position moves);
    void doTurn(Position from, Position to, MoveFun);

private:
    Board board;
    Board lastBoard;

private:
    Positions availablePositions;
    Positions allPositions;
    Positions movesOriginalPosition[MAX_GRID_NUM];
};

#endif
//Reversi.cpp
bool Reversi::isReachable(Position from, Position to, MoveFun move)
{
    enum { MAX_STEP = 8 };
    for(int i = a1; i <  MAX_STEP; ++i)
    {
        from = move(from);
        if( ! Board::onBoard(from)) return false;
        if(from == to) return true;
    }

    return false;
}

void Reversi::doTurn(Position from, Position to, MoveFun move)
{
    if(isReachable(from, to, move))
    {
        Position next = move(from);
        while(next != to)
        {
            board.turnOver(next);
            next = move(next);
        }
    }
}

void Reversi::turn(Position original, Position moves)
{
    doTurn(original, moves, up);
    doTurn(original, moves, down);
    doTurn(original, moves, left);
    doTurn(original, moves, right);
    doTurn(original, moves, leftUp);
    doTurn(original, moves, leftDown);
    doTurn(original, moves, rightUp);
    doTurn(original, moves, rightDown);
}

const Board& Reversi::capture(Position movePosition)
{
    lastBoard = board;
    while( ! movesOriginalPosition[movePosition].isEmpty())
    {
        Position originalPosition = movesOriginalPosition[movePosition].pop();
        board.place(movePosition, board.at(originalPosition).getStatus());
        turn(originalPosition, movePosition);
    }

    return board;
}

const Board& Reversi::retract()
{
    board = lastBoard;
    return board;
}
```

最后一个需求：打印出棋局：
```cpp
//Board.cpp
void Board::print() const
{
    std::cout <<  "\n" << "a b c d e f g h" << std::endl;

    enum { COLUMN_NUM = 8 };
    for(int i = a1; i < MAX_GRID_NUM; ++i)
    {
        if(i % COLUMN_NUM == 0 && i != 0)
        {
            std::cout << " " << i/COLUMN_NUM << "\n";
        }
        std::cout << to_c(grids[i]) << " ";
    }

    std::cout << " " << COLUMN_NUM << "\n";
}

```
a b c d e f g h
B _ _ _ _ _ _ _  1
B _ _ _ _ _ _ _  2
B _ B _ B _ _ _  3
_ _ _ W W B _ _  4
_ _ _ _ W W _ _  5
_ _ _ _ _ _ _ _  6
_ _ _ _ _ _ _ _  7
_ W W W B _ _ _  8

a b c d e f g h
B _ _ _ _ _ _ _  1
W _ _ _ _ _ _ _  2
_ _ B _ B _ _ _  3
_ _ _ B W B _ _  4
_ _ _ _ B B _ _  5
_ _ _ _ _ B _ _  6
_ _ _ _ _ _ _ _  7
_ W W W B _ _ _  8
...

至此需求已经全部完成，设计是否OK了，看看如下代码：
```cpp
   const Position INVALID_POSITION = static_cast<Position>(MAX_GRID_NUM);

    Position up(Position p)
    {
        return static_cast<Position>(p-8);
    }

    Position down(Position p)
    {
        return static_cast<Position>(p+8);
    }

    bool onARow(Position l, Position r)
    {
        return l/8 == r/8;
    }

    Position left(Position p)
    {
        Position leftPos = static_cast<Position>(p-1);
        return onARow(p, leftPos) ? leftPos : INVALID_POSITION;
    }

    Position right(Position p)
    {
        Position rightPos = static_cast<Position>(p+1);
        return onARow(p, rightPos) ? rightPos : INVALID_POSITION;
    }

    Position leftUp(Position p)
    {
        return left(up(p));
    }

    Position leftDown(Position p)
    {
        return left(down(p));
    }

    Position rightUp(Position p)
    {
        return right(up(p));
    }

    Position rightDown(Position p)
    {
        return right(down(p));
    }
```

**REFACTOR**
```cpp
//Direction.h
#ifndef INCLUDE_DIRECTION_H_
#define INCLUDE_DIRECTION_H_

#include "Position.h"

struct Removable
{
	virtual ~Removable() {}
	virtual Position move(Position) const = 0;
};

struct Direction : Removable
{
	virtual Position move(Position) const;

	static Direction& up();
	static Direction& down();
	static Direction& left();
	static Direction& right();

private:
	Direction(int factor);
private:
	int factor;
};

struct JoinMovable : Removable
{
	JoinMovable(const Removable&, const Removable&);
	virtual Position move(Position) const;

private:
	const Removable& left;
	const Removable& right;
};

#define _up Direction::up()
#define _down Direction::down()
#define _left Direction::left()
#define _right Direction::right()
#define _left_up JoinMovable(_left, _up)
#define _left_down JoinMovable(_left, _down)
#define _right_up JoinMovable(_right, _up)
#define _right_down JoinMovable(_right, _down)

//Direction.cpp
#include "Direction.h"

namespace
{
	enum { LEFT_FACTOR = -1, RIGHT_FACTOR = 1, UP_FACTOR = -8, DOWN_FACTOR = 8 };
}

Direction::Direction(int factor) : factor(factor)
{
}

namespace
{
	bool onARow(Position l, Position r)
	{
		return l/8 == r/8;
	}
}

Position Direction::move(Position p) const
{
	Position moved = static_cast<Position>(p + factor);

	if(factor == LEFT_FACTOR || factor == RIGHT_FACTOR)
	{
		moved = onARow(p, moved) ? moved : MAX_POSITION_NUM;
	}

	return moved;
}

Direction& Direction::up()
{
	static Direction up(UP_FACTOR);
	return up;
}

Direction& Direction::down()
{
	static Direction down(DOWN_FACTOR);
	return down;
}

Direction& Direction::left()
{
	static Direction left(LEFT_FACTOR);
	return left;
}

Direction& Direction::right()
{
	static Direction right(RIGHT_FACTOR);
	return right;
}

JoinMovable::JoinMovable(const Removable& left, const Removable& right) : left(left), right(right)
{
}

Position JoinMovable::move(Position p) const
{
	Position r = right.move(left.move(p));
	return r;
}

//Reversi.cpp
const Positions& Reversi::gitAvailablePositions(Position p)
{
    availablePositions.clear();

    find(p, _up);
    find(p, _down);
    find(p, _left);
    find(p, _right);
    find(p, _left_up);
    find(p, _left_down);
    find(p, _right_up);
    find(p, _right_down);

    return availablePositions;
}
```
***

## 为什么要简单设计

利润 ＝ 收益 － 成本
成本：内在成本，偶发成本
+ 写代码
+ 改代码
+ 读代码（理解）
+ 分析、定位、解决故障
+ 重复测试

***

## 价值观－演进式设计

问题思考 － 建模 － 细节 － 解决问题 － 重构 －>

经验 ＋ 解决问题

好的架构是浮现出来的

***

## 如何做到简单设计
简单设计四原则：
OLD:
+ Run all testes
+ ...

***

## 原则－简单设计四原则

+ Passes the tests
+ Reveals intention
+ No duplication
+ Fewest elements

***

## 实践－TDD理解

- 设计即解决问题
- 面相接口编程
- 代码天生具有可测试性
- 缩短故障反馈周期
- 用例设计很重要
***

## 测试驱动开发是否可以驱动出领域模型
经验 ＋ 用例设计 

***
## 实践－重构的理解
打扫房间

***
## 实践－CleanCode关注点
![Mind map][clean-code]
[clean-code]: images/clean-code-that-works/cleanCode.png "CleanCode MindMap"

***

## ClanCode能否解决业务本质复杂度
更容易产生领域模型
***

## CleanCode的价值

减少偶发成本

***

## 最后一个问题

TEST（通过今天的沙龙大家设计能力会有很大提高么）
{
}

*** 
## 别急，重构下你的答案
+ 方向指导
+ 悟，修炼


***

## 软件能力能力模型

***

## 
  #Thanks