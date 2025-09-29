---
title: "刷题笔记"
date: "2025-09-29"
slug: blog-post-slug
tags:
  - 标签
categories:
  - 分类
description: 描述
draft: true
state: "0"
---
## 一、Lambda 表达式是什么？
Lambda 表达式（匿名函数）是 C++11 引入的核心特性，本质是 **“可以在函数内部定义的、无需显式命名的临时函数对象”** 。它的核心价值是：  
- 简化代码：避免为“仅局部使用的简单逻辑”单独定义全局/类成员函数；  
- 配合 STL：高效结合 `for_each`、`sort`、`find_if` 等 STL 算法，实现“逻辑内联”；  
- 捕获外部变量：能灵活访问定义它的“外部作用域变量”（普通函数无法直接做到）。


## 二、Lambda 的完整语法结构
Lambda 的语法看似复杂，实则有固定范式，完整结构如下：
```cpp
[capture-list] (parameter-list) mutable noexcept -> return-type { function-body }
```
各部分的含义与作用如下表所示，其中**仅「捕获列表」和「函数体」是必选的**，其他部分可根据需求省略：

| 组成部分          | 含义与作用                                                                 | 可省略情况                          |
|-------------------|----------------------------------------------------------------------------|-----------------------------------|
| `[capture-list]`  | 「捕获列表」：指定 Lambda 能访问的“外部作用域变量”，是 Lambda 的核心特性之一 | 不可省略（空列表 `[]` 也需显式写） |
| `(parameter-list)`| 「参数列表」：与普通函数的参数列表一致，传递输入参数                         | 无参数时可省略（如 `[] { ... }`）  |
| `mutable`         | 允许修改「值捕获」的变量副本（默认值捕获的变量是 `const` 的）               | 无需修改值捕获变量时可省略         |
| `noexcept`        | 声明 Lambda 不会抛出异常（C++11 后支持）                                   | 不保证无异常时可省略               |
| `-> return-type`  | 「返回类型尾随说明符」：指定 Lambda 的返回值类型                           | 函数体仅单个 `return` 时可自动推导 |
| `{ function-body }`| 「函数体」：Lambda 的核心逻辑，与普通函数一致                               | 不可省略                          |


## 三、核心部分：捕获列表（`[capture-list]`）
捕获列表是 Lambda 与普通函数的**最大区别**，它决定了 Lambda 能访问哪些外部变量，以及访问方式（值/引用）。常见的捕获方式如下：

| 捕获方式         | 语法        | 含义                                       |
| ------------ | --------- | ---------------------------------------- |
| 空捕获          | `[]`      | 不捕获任何外部变量（如题目中的 `isVowel`）               |
| 值捕获          | `[x]`     | 捕获外部变量 `x` 的副本（Lambda 内修改的是副本，不影响外部原变量）  |
| 引用捕获         | `[&x]`    | 捕获外部变量 `x` 的引用（Lambda 内修改会直接影响外部原变量）     |
| 捕获所有外部变量（值）  | `[=]`     | 以“值捕获”方式捕获 Lambda 所在作用域的**所有外部变量**       |
| 捕获所有外部变量（引用） | `[&]`     | 以“引用捕获”方式捕获 Lambda 所在作用域的**所有外部变量**      |
| 混合捕获         | `[x, &y]` | 值捕获 `x`，引用捕获 `y`（可组合值/引用）                |
| 捕获 `this` 指针 | `[this]`  | 类成员函数中使用，捕获当前对象的 `this` 指针（可访问类的成员变量/函数） |


### 示例：结合“元音统计”理解捕获列表
假设我们修改需求：统计字符串中元音的**总数量**，并通过 Lambda 实现（需要捕获外部计数器）：
```cpp
#include <iostream>
#include <string>
using namespace std;

int main() {
    string s = "abciiidef";
    int vowel_total = 0; // 外部变量：统计元音总数

    // Lambda：引用捕获 vowel_total（需修改外部变量），无参数
    auto countVowel = [&vowel_total](char c) {
        if (c == 'a' || c == 'e' || c == 'i' || c == 'o' || c == 'u') {
            vowel_total++; // 引用捕获：修改会影响外部的 vowel_total
        }
    };

    // 遍历字符串，调用 Lambda 统计
    for (char c : s) {
        countVowel(c);
    }

    cout << "元音总数：" << vowel_total << endl; // 输出 3（a + i + i + i → 实际是 4？哦 s是abciiidef：a、i、i、i → 4，代码正确）
    return 0;
}
```
- 这里 Lambda 用 `[&vowel_total]` 引用捕获外部变量，因此在 Lambda 内修改 `vowel_total` 会直接同步到外部，实现统计功能。
- 若改为值捕获 `[vowel_total]`，则 Lambda 内修改的是副本，外部 `vowel_total` 始终为 0（无意义）。


## 四、题目中的 Lambda 深度解析：`isVowel`
回顾“定长子串元音最大数”问题中的 Lambda：
```cpp
auto isVowel = [](char c) {
    return c == 'a' || c == 'e' || c == 'i' || c == 'o' || c == 'u';
};
```
我们逐部分拆解，理解它为何是最优选择：

### 1. 各部分解析
| 组成部分          | 代码中的实现       | 原因分析                                                                 |
|-------------------|--------------------|--------------------------------------------------------------------------|
| `[capture-list]`  | `[]`（空捕获）     | 无需访问任何外部变量，仅需参数 `c` 即可判断是否为元音，因此空捕获足够。   |
| `(parameter-list)`| `(char c)`         | 需要接收一个字符作为输入，判断其是否为元音，因此参数列表为 `char c`。     |
| `return-type`     | 省略（自动推导）   | 函数体仅单个 `return` 语句（返回 `bool` 类型），编译器可自动推导返回值。 |
| `function-body`   | 逻辑判断           | 通过 `||` 直接判断字符是否为 5 个元音之一，逻辑简洁高效。                 |


### 2. 为什么用 Lambda 而不用其他方式？
对比“普通函数”“函数对象”“`unordered_set` 查找”，Lambda 的优势显而易见：

| 实现方式          | 代码示例                                                                 | 缺点                                                                 |
|-------------------|--------------------------------------------------------------------------|----------------------------------------------------------------------|
| 普通函数          | `bool isVowel(char c) { ... }`                                           | 需在类外/函数外单独定义，若仅在 `maxVowels` 中使用，会“污染命名空间”。 |
| `unordered_set`   | `unordered_set<char> vowels{'a','e','i','o','u'}; return vowels.count(c);` | 哈希表查找有额外开销（比直接逻辑判断慢），且需额外创建容器。           |
| 函数对象          | 定义一个结构体并重载 `operator()`：`struct IsVowel { bool operator()(char c) { ... } };` | 代码冗余，仅为简单逻辑需定义结构体，性价比低。                       |
| **Lambda**        | 如题目中的实现                                                           | 代码内联、无额外开销、不污染命名空间，完美适配局部场景。               |


## 五、Lambda 的进阶用法
掌握基础后，我们扩展 Lambda 的常见进阶场景，应对更复杂的需求：


### 1. `mutable`：修改值捕获的变量副本
值捕获的变量默认是 `const` 的，若需在 Lambda 内修改**副本**（不影响外部原变量），需加 `mutable`：
```cpp
#include <iostream>
using namespace std;

int main() {
    int x = 10;
    // 值捕获 x，加 mutable 允许修改副本
    auto modifyX = [x]() mutable {
        x++; // 仅修改 Lambda 内的 x 副本
        cout << "Lambda 内 x：" << x << endl; // 输出 11
    };

    modifyX();
    cout << "外部 x：" << x << endl; // 输出 10（原变量未变）
    return 0;
}
```


### 2. 指定返回类型（`-> return-type`）
当函数体有多个 `return` 语句，且返回类型可能不一致时，需显式指定返回类型：
```cpp
// 若不指定 -> int，编译器无法推导返回类型（一个 return int，一个 return double）
auto addOrMultiply = [](int a, int b, bool isAdd) -> int {
    if (isAdd) {
        return a + b; // 返回 int
    } else {
        return a * b; // 返回 int（若写成 a * 1.0，则需改为 -> double）
    }
};

cout << addOrMultiply(2, 3, true) << endl;  // 5
cout << addOrMultiply(2, 3, false) << endl; // 6
```


### 3. 配合 STL 算法
Lambda 最常用的场景是配合 STL 算法，实现“逻辑内联”。例如，用 `for_each` 遍历字符串并统计元音：
```cpp
#include <iostream>
#include <string>
#include <algorithm> // for_each
using namespace std;

int main() {
    string s = "aeiou";
    int vowel_count = 0;

    // for_each + Lambda：遍历字符串，统计元音
    for_each(s.begin(), s.end(), [&vowel_count](char c) {
        if (c == 'a' || c == 'e' || c == 'i' || c == 'o' || c == 'u') {
            vowel_count++;
        }
    });

    cout << "元音数：" << vowel_count << endl; // 输出 5
    return 0;
}
```


## 六、Lambda 的注意事项
1. **生命周期**：Lambda 是“临时函数对象”，若需长期保存（如存入容器），需确保捕获的外部变量（尤其是引用）生命周期不短于 Lambda；  
   - 错误示例：返回一个捕获局部变量引用的 Lambda，外部调用时变量已销毁，导致未定义行为。
2. **捕获 `this` 指针**：在类成员函数中，`[this]` 捕获当前对象的 `this` 指针，可访问类的成员变量/函数，但需确保对象在 Lambda 调用时未被销毁。
3. **性能**：Lambda 本身无额外性能开销（与普通函数相当），但引用捕获需注意避免悬空引用，值捕获需注意拷贝开销（捕获大对象时建议用引用）。