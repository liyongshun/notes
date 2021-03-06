#Clean Code That Works#
##1. 为什么要CleanCode？##

“Clean Code That Works”，来自于*Ron Jeffries*这句箴言指导我们写的代码要整洁有效，*Kent Beck*把它作为TDD（Test Driven Development）追求的目标，*BoB大叔*（Robert C. Martin）甚至写了一本书来阐述他的理解。 
你可能不这么认为：**Works**不就Ok了么？为什么还要**CleanCode**呢?
*kent Beck*给出了如下原因：
+ It is a predictable way to develop. You know when you are finished, without having to worry about a long bug trail.
+ It gives you a chance to learn all of the lessons that the code has to teach you. If you only slap together the first thing you think of, then you never have time to think of a second, better thing.
+ It improves the lives of the users of your software.
+ It lets your teammates count on you, and you on them.
+ It feels good to write it.

我们分析他的观点，主要如下几个要点：
 + 整洁的代码质量更好，故障就像白墙上的一只苍蝇，如此的显眼，往往令其无处藏身。开发的高质量，缩短验证的周期，让交付更容易被估算。
 + 最容易想到的方法往往是糟糕的方法，*CleanCode*的代码需要花费心思去推敲，更容易抓住业务的本质，一次性把事情做对。
 + 整洁的代码可读性更好，风格统一，开发小组之间更容易产生信任关系

## 2. 如何做到整洁有效的代码 ##
### 原则：简单设计四原则： ###
> 
+ Passes the tests
+ Reveals intention
+ No duplication
+ Fewest elements

<!-- 
优先级如下：
![SimpleDesignPrinciple][simple-design-principle]
-->

### 实践：测试驱动开发 ###
![Tdd][tdd]
### CleanCode关注点###
![Mind map][clean-code]

## 3. 练习 ##
技术教练认证初试题目：翻转棋(Reversi，详见[wiki](https://en.wikipedia.org/wiki/Reversi))
给定棋局（初始棋盘）：  
![Othello Board][othello-init]
**需求：**  
 1. 求解所有黑棋可落子位置（e.g. 黑棋可落子位置为：**d3, c4, f5, e6 ** ）  
 ![Dark Moves][othello-dark-moves]
 
 2. 在某一位置落子后，翻转吃掉的子（e.g. 落子到 **d3** 后，白子 **d4** 被吃掉）  
 ![After dark play][after-dark-play]
 
 3. 打印出所有可能落子后对应的棋局

### 3.1 如何启动？###
这可能是我们所有人遇到的很重要的一个问题。
也许你会说：“先别急着动手，我们需要抽象出语义模型，找出问题背后的本质概念”；又或者：“还磨叽什么，直接开写，TDD，Write a test that fails”。这是我在开发过程中见的最多的两种观点。
我的观点有些折中： 
+ 理清业务需求，根据经验，找到当前上下文下自己能找到的最好设计
+ 演进式的设计，通过Fail-Pass-Refactor方式逐渐找到问题的本质

我们快速把需求翻译成测试用例：
```cpp
//ReversiTest.cpp
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
回到问题，我们要实现翻转棋中基础功能，我们先分析下有哪些概念：
+ 棋盘（Board）: 
	棋盘由8X8方格(Grid)组成，每个空格可以有三种状态，空闲，放黑棋，放白棋，默认棋盘在e4, d5位置(Position)放置黑棋，在d4, e5位置放置白棋
+ 棋子（Disk）：
	黑棋（Black），白棋（White）
+ 规则（Rules）:
	Rule-1. 一方在某个位置落子必须能够夹对方一个或多个连续棋子
	Rule-2. 可以横着夹，竖着夹，斜着夹
	Rule-3. 被夹住的棋子需要进行翻转
+ 位置控制
	可以向上，下，左，右，左上，左下，右上，右下 8个方向进行搜索，在棋盘上搜索空间最大为7，超出则越界。
    
### 3.2 TDD
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

---

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
---
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
#include "Board.h"

Board::Board()
{
    for(GridStatus& status: grids)
    {
        status = EMPTY;
    }

    grids[e4] = BLACK;
    grids[d5] = BLACK;
    grids[d4] = WHITE;
    grids[e5] = WHITE;
}

GridStatus Board::at(Position p) const
{
    return grids[p];
}

bool Board::isOccupied(Position p) const
{
    if( ! onBoard(p)) return false;

    return grids[p] == BLACK || grids[p] == WHITE;
}

bool Board::onBoard(Position p) const
{
    return p >= a1 && p <= h8;
}

void Board::place(Position p, GridStatus s)
{
    grids[p] = s;
}

void Board::turnOver(Position p)
{
    if( ! isOccupied(p)) return;

    grids[p] = (grids[p] == BLACK) ? WHITE : BLACK;
}
```
---
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
#include "Grid.h"

Grid::Grid()
{
    reset();
}

void Grid::place(GridStatus s)
{
    if(s == EMPTY) return;
    status = s;
}

void Grid::reset()
{
    status = EMPTY;
}

void Grid::turnOver()
{
    if( ! isOccupied()) return;

    status = isBlack() ? WHITE : BLACK;
}

bool Grid::isOccupied() const
{
    return isBlack() || isWhite();
}

bool Grid::isBlack() const
{
    return status == BLACK;
}

bool Grid::isWhite() const
{
    return status == WHITE;
}

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
#include "Board.h"

Board::Board()
{
    for(Grid& status: grids)
    {
        status.reset();
    }

    grids[e4].place(BLACK);
    grids[d5].place(BLACK);
    grids[d4].place(WHITE);
    grids[e5].place(WHITE);
}

Grid Board::at(Position p) const
{
	if( ! onBoard(p)) return Grid();
    return grids[p];
}

bool Board::isOccupied(Position p) const
{
    if( ! onBoard(p)) return false;
    return grids[p].isOccupied();
}

bool Board::onBoard(Position p) const
{
    return p >= a1 && p <= h8;
}

void Board::place(Position p, GridStatus s)
{
    grids[p].place(s);
}

void Board::turnOver(Position p)
{
    grids[p].turnOver();
}

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
```	if( ! onBoard(p)) return Grid();```当给定位置无效时，每次都会构造用一个对象，使用static NullObject即可.**此处加入性能2/8原则解释**  
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

---
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
#include "Positions.h"
#include <iostream>

Positions::Positions(std::initializer_list<Position> positions)
{
    for(auto position : positions)
    {
        list.push_back(position);
    }
}

bool Positions::isEmpty() const
{
    return list.size() == 0;
}

bool Positions::operator==(const Positions& rhs) const
{
    if(list.size() != rhs.list.size()) return false;

    for(auto item : rhs.list)
    {
        if( ! contains(item)) return false;
    }

    return true;
}

bool Positions::operator!=(const Positions& rhs) const
{
    return !operator==(rhs);
}

bool Positions::contains(Position p) const
{
    for(auto item : list)
    {
        if(item == p) return true;
    }

    return false;
}

void Positions::push(Position p)
{
    list.push_back(p);
}

Position Positions::Positions::pop()
{
    Position front = list.front();
    list.pop_front();
    return front;
}

void Positions::clear()
{
    list.clear();
}
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
**PASS**

```cpp
```


[tdd]: images/clean-code-that-works/tdd.gif
[simple-design-principle]: images/clean-code-that-works/SimpleDesignPrinciple.png
[clean-code]: images/clean-code-that-works/cleanCode.png "CleanCode MindMap"
[othello-init]: images/clean-code-that-works/othelloInit.png
[othello-dark-moves]: images/clean-code-that-works/darkMoves.png
[after-dark-play]: images/clean-code-that-works/afterDarkPlay.png

