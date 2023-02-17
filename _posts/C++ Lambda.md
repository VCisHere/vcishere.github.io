---
layout: post
title: "C++ Lambda"
date: 2022-10-17 16:03:15 +0800
categories: 计算机 C++
math: false
typora-root-url: ..

---

完整的声明格式：
`[capture list] (params list) mutable exception-> return type { function body }`
例子：
`[](int a, int b) -> bool { return a < b; }`

捕获：
1. 值捕获  
```cpp
int main() {
  int a = 123;
  auto f = [a] { cout << a << endl; }; 
  a = 321;
  f(); // 输出：123
}
```

2. 引用捕获  
```cpp
int main() {
  int a = 123;
  auto f = [&a] { cout << a << endl; }; 
  a = 321;
  f(); // 输出：321
}
```

3. 隐式值捕获
```cpp
int main() {
  int a = 123;
  auto f = [=] { cout << a << endl; };
  f(); // 输出：123
}
```

4. 隐式引用捕获  
```cpp
int main() {
  int a = 123;
  auto f = [&] { cout << a << endl; };
  a = 321;
  f(); // 输出：321
}
```

在 Lambda 表达式中，如果以传值方式捕获外部变量，则函数体中不能修改该外部变量，否则会引发编译错误。这时就需要使用 mutable 关键字，该关键字用以说明表达式体内的代码可以修改值捕获的变量：
```cpp
int main() {
  int a = 123;
  auto f = [a]() mutable { cout << ++a; };
  cout << a << endl; // 输出：123
  f(); // 输出：124
}
```

参数：
1. 参数列表中不能有默认参数
2. 不支持可变参数
3. 所有参数必须有参数名

编译产物：
不带捕获时，会多一个 retType，用于当作函数指针传递。
```cpp
// auto add = [](int a, int b) -> int { return a + b; };

class __lambda_6_14 {
 public:
  inline int operator()(int a, int b) const { return a + b; }

  using retType_6_14 = auto(*)(int, int) -> int;
  inline constexpr operator retType_6_14() const noexcept {
    return __invoke;
  };

 private:
  static inline int __invoke(int a, int b) {
    return __lambda_6_14{}.operator()(a, b);
  }
};
```

带捕获(值捕获或引用捕获)时，不能当作函数指针传递，所以自然就不包含 retType。
```cpp
// auto add = [&](int a, int b) -> int { return a + b; };

class __lambda_10_14 {
 public:
  inline int operator()(int a, int b) const { return a + b; }
  };
```
