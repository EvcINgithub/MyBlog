# V8 Parser

## 一、严重错误的底层调用

**V8_Fatal**函数在logging.cc中实现主要实现：1.错误信息存储在栈区的FailureMessage变量，2.根据错误的 “性质”（是否为 “无害的受控崩溃”）输出不同的错误提示信息，3.打印错误信息及FailureMesage变量在栈中的地址,4.(可选)打印追踪栈。FailureMssage的结构如下:

```Cpp
  uintptr_t start_marker_ = kStartMarker;
  char message_[kMessageBufferSize];
  uintptr_t end_marker_ = kEndMarker;
```

## 二、Token实现

**Token.h**定义了两个重要的结构：1.TOKEN_LIST,宏定义了所有的Token类型，包含**name, string, precedence**三个属性。2.定义了Token类,用以描述全部的token类型。结构如下:

```Cpp
//argument
public:
  //每种token对应一个uint8类型的整数,总数为kNumTokens=120
  enum Value : uint8_t { TOKEN_LIST(T, T) kNumTokens };
private:
  //属性存储
  static const char* const name_[kNumTokens];
  static const char* const string_[kNumTokens];
  static const uint8_t string_length_[kNumTokens];
  //2为accept_in的可选项
  static const int8_t precedence_[2][kNumTokens];
  static const uint8_t token_flags[kNumTokens];
//method
public:
  //这四个方法都是get类型的方法
  static const char* Name(Value token){/*...*/} 
  static const char* String(Value token){/*...*/} 
  static uint8_t StringLength(Value token){/*...*/} 
  static int Precedence(Value token, bool accept_IN){/*...*/}
  
  //特别地,这个函数的作用是将复合赋值运算符转换为对应的二元操作符,例:+=   ->   +
  static Value BinaryOpForAssignment(Value op) {
    DCHECK(base::IsInRange(op, kAssignNullish, kAssignSub));
    //复合赋值运算符token与对应的二元运算符token在定义时是 “连续且偏移量一致” 的。
    //value(+) - value(-)  = value(+=) - value(-=)
    Value result = static_cast<Value>(op - kAssignNullish + kNullish);
    DCHECK(IsBinaryOp(result));
    return result;
  }

  //还有一些判断类型的函数利用的也是token_list的同一类token的数值是连续的
  //...
```

**Token.cc**实现了对静态类Token的private属性的全部初始化
