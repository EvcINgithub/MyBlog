# 工具类

类**Utf16CharacterStream**实现了对大块Utf16字符串的的分块读取，如下图所示每次只处理一块占据一个Buffer，其中buffer_*_函数描述了当前buffer块的属性:缓冲在内存中的开始位置的值pos，缓冲在内存中的指针start,end以及下一个要读的character的指针cursor。而pos()函数利用绝对地址和指针的相对便宜计算出某一character的在字符串绝对地址。

```CPP
protected:
  // Fields describing the location of the current buffer physically in memory,
  // and semantically within the source string.
  //
  //                  0              buffer_pos_   pos()
  //                  |                        |   |
  //                  v________________________v___v_____________
  //                  |                        |        |        |
  //   Source string: |                        | Buffer |        |
  //                  |________________________|________|________|
  //                                           ^   ^    ^
  //                                           |   |    |
  //                   Pointers:   buffer_start_   |    buffer_end_
  //                                         buffer_cursor_
  const uint16_t* buffer_start_;
  const uint16_t* buffer_cursor_;
  const uint16_t* buffer_end_;
  size_t buffer_pos_;
  RuntimeCallStats* runtime_call_stats_ = nullptr;
  bool has_parser_error_ = false;
public:
  static constexpr base::uc32 kEndOfInput = static_cast<base::uc32>(-1);  //用以描述串的结尾
```

**Utf16CharacterStream**的一些功能函数:

```CPP
//访存函数
base::uc32 Peek(){}//返回当前buffer_cursor_指向单元的字的值，在cursor超出当前块范围时会读下一块
//更专业的来说,查看流中的「下一个元素」（或下 N 个元素），但不改变流的读取位置（不消费该元素）

base::uc32 Advance(){}//调用peek(),并且使得cursor指向下一位

template<typename FunctionType>
base::uc32 AdvanceceUntil(FunctionType check){}//找到满足check函数的值并返回，cursor指向返回值下一个待读字,该函数会从当前块开始一直读到流末尾

void Back(){}//advance函数的逆函数

bool ReadBlockChecked(size_t position){}//读块并检查读块合法性检查

virtual bool ReadBlock(size_t position) = 0;//虚函数
```

一些其它装态函数，目前尚未清楚作用

```CPP
bool has_parser_error_;//还有一些控制该参数的函数
bool can_be_cloned_for_parallel_access() const {
    return can_be_cloned() && !can_access_heap();
  }
  virtual bool can_be_cloned() const = 0;
  virtual std::unique_ptr<Utf16CharacterStream> Clone() const = 0;
  virtual bool can_access_heap() const = 0;
  RuntimeCallStats* runtime_call_stats() const { return runtime_call_stats_; }
  void set_runtime_call_stats(RuntimeCallStats* runtime_call_stats) {
    runtime_call_stats_ = runtime_call_stats;
  }
```
