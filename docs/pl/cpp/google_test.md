# Google Test

谷歌测试框架(Google Test)是一个用于编写和运行 C++ 单元测试的强大且灵活的测试框架。它由 Google 开发并广泛应用于各种 C++ 项目中。它主要的功能如下：

- 断言：支持多种类型的断言，如`ASSERT_EQ`、`EXPECT_TRUE`等。
- 测试夹具：允许在每个测试之前和之后设置和清理测试环境
- 参数化测试：支持参数化方式运行相同的测试用例
- 死亡测试：支持测试程序在特定条件下是否会正确终止

假如有项目结构如下所示：
```SHELL
goole_test/
├── CMakeLists.txt
├── src
│   ├── add.cpp
│   └── add.h
└── tests
    └── test_add.cpp
```

其中源代码文件`src/add.cpp`和头文件`src/add.h`如下所示：

```CPP
// add.h
#ifndef ADD_H
#define ADD_H

int add(int a, int b);

#endif // ADD_H

// add.cpp
#include "add.h"

int add(int a, int b) {
    return a + b;
}
```

测试文件`tests/test_add.cpp`如下所示：

```CPP
#include "gtest/gtest.h"
#include "add.h"


TEST(AdditionTest, PositiveNumbers) {
    EXPECT_EQ(add(2, 3), 5);
}

TEST(AdditionTest, NegativeNumbers) {
    EXPECT_EQ(add(-2, -3), -5);
}

TEST(AdditionTest, MixedNumbers) {
    EXPECT_EQ(add(-2, 3), 1);
}
```

CMakeLists.txt 文件如下所示：

```CMAKE
cmake_minimum_required(VERSION 3.10)
project(MyProject)

# 查找Google Test库
find_package(GTest REQUIRED)

# 包含头文件目录
include_directories(src)

# 添加可执行文件
add_executable(test tests/test_add.cpp src/add.cpp)

# 链接Google Test库（包括默认main函数）和线程库
target_link_libraries(test GTest::GTest GTest::Main)
```