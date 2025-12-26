# V8 scanner部分

[TOC]

## 1.基础知识Unicode

统一码（Unicode）是统一码联盟制定的字符编码国际标准，涵盖字符集及UTF-8、UTF-16、UTF-32等编码方案。该标准通过为全球各语言字符分配唯一编码，解决传统编码的兼容性问题，支持跨语言、跨平台文本处理，其中UTF-8兼容ASCII编码并采用可变字节设计。标准编码范围覆盖基本多文种平面至第16辅助平面，包含现代及历史文字、符号，采用代理区机制处理辅助平面字符。
>在 Unicode 字符集 中，Code Point（码点） 是核心概念，本质是：给每个 “字符”（包括字母、汉字、符号、表情等）分配的唯一数值标识符，用于在全球范围内统一标识字符，避免不同编码体系的冲突。
类说明

## 2.辅助类（作用域仅在Scanner内部）

内部工具类**ErrorState**:作用域限定的辅助工具，用于保存和恢复扫描器的错误状态。该工具适用于带标签的模板字面量场景 —— 在此场景下，通常被禁止的转义序列是允许的。

```cpp
class Scanner::ErrorState {
 public:
  ErrorState(MessageTemplate* message_stack, Scanner::Location* location_stack){/*...*/}//没什么特别的构造函数
  ~ErrorState() {
    *message_stack_ = old_message_;
    *location_stack_ = old_location_;
  }
  void MoveErrorTo(TokenDesc* dest) {
    if (*message_stack_ == MessageTemplate::kNone) {
      return;
    }
    if (dest->invalid_template_escape_message == MessageTemplate::kNone) {
      dest->invalid_template_escape_message = *message_stack_;
      dest->invalid_template_escape_location = *location_stack_;
    }
    *message_stack_ = MessageTemplate::kNone;
    *location_stack_ = Location::invalid();
  }

 private:
  MessageTemplate* const message_stack_;
  MessageTemplate const old_message_;
  Scanner::Location* const location_stack_;
  Scanner::Location const old_location_;
};

```

Scanner的内部工具类**BookmarkScope**实现:围绕着bookmark_展开作为scanner的书签使用，bookmark_存在三种状态kNoBookmark,kBookmarkWasApplied和确定的position值。该类主要在Apply()函数种利用scanner的SeekNext函数跳跃到目标书签处

```cpp
    void Set(size_t bookmark);
    void Apply();
    bool HasBeenSet() const;
    bool HasBeenApplied() const;

    Scanner* scanner_;
    size_t bookmark_;
    bool had_parser_error_;

```

Scanner的内部数据结构体**Location**实现，该结构体用来统一描述token在内存中的真实位置:

```cpp
    int length() const { return end_pos - beg_pos; }
    bool IsValid() const { return base::IsInRange(beg_pos, 0, end_pos); }

    static Location invalid() { return Location(-1, 0); }

    // 代表了词法单元在源码中的位置满足左闭右开,[beg_pos,end_pos)即该词法单元的最后一个字符   存疑！！！！！！！！！！！！！！！！！！！！！！！！！！！！！！
    int beg_pos;
    int end_pos;

```

Scanner的内部数据结构体**TokenDecs**实现，封装了描述一个token的所有属性:

```cpp
  // The current and look-ahead tokens.
  struct TokenDesc {
    Location location = {0, 0};//token定位
    LiteralBuffer literal_chars;//用来存放转义后打包好的字面量
    LiteralBuffer raw_literal_chars;//用来存放原始不转义的打包好的字面量
    Token::Value token = Token::kUninitialized;//token的属性，关键字名/标识符/...
    MessageTemplate invalid_template_escape_message = MessageTemplate::kNone;  //标志位，标志该token是否存在不合法的转义字符
    Location invalid_template_escape_location;//对不合法转义字符的位置定位
    NumberKind number_kind;//NumberKind为数字类型的enum
    uint32_t smi_value = 0;
    bool after_line_terminator = false;

#ifdef DEBUG
    bool CanAccessLiteral() const {}  //检查当前token是否需要解析为字面量，反例“+”不需要进literalbuffer进行字面量解析 
    bool CanAccessRawLiteral() const {} //检查当前token是否需要保留原始格式（不处理转义)
#endif  // DEBUG
  };

```

## 3.Scanner的成员变量

```cpp
  //这4个TokenDesc 意味着scanner中最多预读3个词法单元,next_天然地自动往前看,next_next_和next_next_next_都要在调用了相应的函数后才会预读。
  //这意味者通常而言source_(即Utf16CharacterStream)的buffer_curser_通常指向next_.location.beg_pos即第一个尚未读入的字符
  TokenDesc* current_;    // desc for current token (as returned by Next())  
  TokenDesc* next_;       // desc for next token (one token look-ahead)
  TokenDesc* next_next_;  // desc for the token after next (after peek())
  TokenDesc* next_next_next_;  // desc for the token after next of next (after
                               // PeekAhead())

  // Input stream. Must be initialized to an Utf16CharacterStream.
  Utf16CharacterStream* const source_;

  // One Unicode character look-ahead; c0_ < 0 at the end of the input.
  //c0_是一个在source中已经消耗掉但是尚未进行词法分析的字符
  base::uc32 c0_;

  TokenDesc token_storage_[4];

  // Whether this scanner encountered an HTML comment.
  bool found_html_comment_;

  // Values parsed from magic comments.
  LiteralBuffer source_url_;
  LiteralBuffer source_mapping_url_;
  LiteralBuffer debug_id_;
  bool saw_source_mapping_url_magic_comment_at_sign_ = false;
  bool saw_magic_comment_compile_hints_all_ = false;
  bool saw_non_comment_ = false;
  std::vector<int> per_function_compile_hint_positions_;
  size_t per_function_compile_hint_positions_idx_ = 0;

  // Last-seen positions of potentially problematic tokens.
  Location octal_pos_;
  MessageTemplate octal_message_;

  MessageTemplate scanner_error_;
  Location scanner_error_location_;

```

## 4.方法说明

**初始化函数**说明

```cpp
  //初始化
  void Initialize();
  //主要完成三步
  //  Init();这一步除了scanner的关键变量进行初始化外还会调用Advance()，调用Utf16CharacterStream的advance，为什么？
  //  next().after_line_terminator = true; //为什么？
  //  Scan();

```

---
关于**移动类型的操作**，可以分为对“原料”和对“产品”的移动

```cpp

  // 返回当前读的字符c0_的真实内存地址，source->pos()返回的是下一个要读的字符的真实位置,kCharacterLookaheadBufferSize为定值1
  int source_pos() {
    return static_cast<int>(source_->pos()) - kCharacterLookaheadBufferSize;
  }
  //-----------------这几个函数都是在操作缓冲区的字符-----------------

  //直接调用Utf16CharacterStream的advance,向前读取一个字符并消耗
  //这里被读取的字符存入c0中
  //capture_row用来控制是否触发AddRawLiteralChar(c0_);即将转义字符的不转义串写入Token的raw_literal_chars缓存中,注意这里写入的是当前转义字符，在写入之后才会步进到下一个待读取字符
  void Advance<bool capture_row>(){}
  //完完全全是Advance的逆操作,ch是被消耗的词素
  //ch被重新写进c0
  void PushBack(char ch){} 
  //向前查看一个字符调用source_的Peek(),注意和查看token的peek()区分
  base::uc32 Peek() const{}


  //-----------------这几个函数都是在操作扫描好的词法单元-----------------

  //同advance相似的消费函数,通过重新对current_,next_,next_next_,
  // next_next_next_ 赋值（即改变这几个指针指向的token_storage_的
  // 内存来实现返回一个token并继续向前扫描新的token
  Token::Value Scanner::Next() {
    TokenDesc* previous = current_;
    current_ = next_;
    if (V8_LIKELY(next_next().token == Token::kUninitialized)) {
      DCHECK(next_next_next().token == Token::kUninitialized);
      next_ = previous;
      previous->after_line_terminator = false;
      Scan(previous);
    } else {
      next_ = next_next_;

      if (V8_LIKELY(next_next_next().token == Token::kUninitialized)) {
        next_next_ = previous;
      } else {
        next_next_ = next_next_next_;
        next_next_next_ = previous;
      }

      previous->token = Token::kUninitialized;
      DCHECK_NE(Token::kUninitialized, current().token);
    }
    return current().token;
  } 

  Token::value peek(){}//查看next_但是不消耗
  Token::value PeekAhead(){} //查看next_next_但是不消耗
  Token::value PeekAheadAhead(){} //查看next_next_next_但是不消耗


  //-----------------这两个函数都是在进行跳跃动作-----------------

  void Scanner::SeekForward(int pos) {
    // After this call, we will have the token at the given position as
    // the "next" token. The "current" token will be invalid.
    if (pos == next().location.beg_pos) return;
    int current_pos = source_pos();
    DCHECK_EQ(next().location.end_pos, current_pos);
    // Positions inside the lookahead token aren't supported.
    DCHECK(pos >= current_pos);
    if (pos != current_pos) {
      source_->Seek(pos);
      Advance();
      // This function is only called to seek to the location
      // of the end of a function (at the "}" token). It doesn't matter
      // whether there was a line terminator in the part we skip.
      next().after_line_terminator = false;
    }
    Scan();
  }

  //理论上来说是
  void Scanner::SeekNext(int pos){}
```

---
**扫描类函数SkipMultiLineComment**通过存在嵌套结构的自动机实现，下面以解析多行注释为例。
从解析多行注释看状态机:
![状态机](src\img\状态机.jpg "多行解释的状态机")
从解析多行注释看**next().after_line_terminator**。多行注释再ECMA中的定义：注释可以是单行或多行。
> 多行注释不能嵌套。
> 由于单行注释可包含除 LineTerminator 码点外的任意 Unicode 码点，并且根据“一段记号总是尽可能长”的通则，单行注释总是从 // 标记到该行末端的全部码点。然而，行末的 LineTerminator 不视为单行注释的一部分；它被词法语法单独识别并进入供句法语法使用的输入元素流。这一点很重要，因为这意味着单行注释的有无不影响自动分号插入。
> 注释的行为类似空白并被丢弃；但若 MultiLineComment 含有行终止符码点，则整个注释在句法解析目的上被视为一个 LineTerminator。

```cpp
Token::Value Scanner::SkipMultiLineComment() {
  DCHECK_EQ(c0_, '*');

  // Until we see the first newline, check for * and newline characters.
  //如果已经把下一个词法单元的after_line_terminator正确设置那直接去找结尾的的 */ 就好了
  if (!next().after_line_terminator) {
    do {
      AdvanceUntil([](base::uc32 c0) {
        if (V8_UNLIKELY(static_cast<uint32_t>(c0) > kMaxAscii)) {
          return unibrow::IsLineTerminator(c0);
        }
        uint8_t char_flags = character_scan_flags[c0];
        return MultilineCommentCharacterNeedsSlowPath(char_flags);
      }); //找到下一个 * 或者 行末终止符

      while (c0_ == '*') {
        Advance();
        if (c0_ == '/') {
          Advance();
          return Token::kWhitespace;
        }
      }//这里之所以要跳过连续的*是因为javascript有自带的JsDoc注释

      if (unibrow::IsLineTerminator(c0_)) {
        next().after_line_terminator = true;
        break;
      }//多行注释内换行了标记下一个词法单元的属性
    } while (c0_ != kEndOfInput);
  }

  // After we've seen newline, simply try to find '*/'.
  while (c0_ != kEndOfInput) {
    AdvanceUntil([](base::uc32 c0) { return c0 == '*'; });
    while (c0_ == '*') {
      Advance();
      if (c0_ == '/') {
        Advance();
        return Token::kWhitespace;
      }
    }
  }
  return Token::kIllegal;
}

```

从**扫描类函数ScanIdentifierOrKeywordInner**看快慢路径的应用,需要注意的是这个函数并没有任何先验知识所以是在扫描新的Token。
首先在第14行有个特殊的右移处理，该处理将**kCannotBeKeywordStart**状态转换为**kCannotBeKeyword**状态。此处处理确实会将状态位打乱(但可以合理假设scanflags前四个状态有且仅有一个状态被置位)。之后第15行的Dcheck检查最后一位是否存在错误地右移(奇怪的是为什么多行注释慢检查为什么会进入这个函数)

```cpp {.line-numbers}
V8_INLINE Token::Value Scanner::ScanIdentifierOrKeywordInner() {
  DCHECK(IsIdentifierStart(c0_));
  bool escaped = false;
  bool can_be_keyword = true;

  static_assert(arraysize(character_scan_flags) == kMaxAscii + 1);
  if (V8_LIKELY(static_cast<uint32_t>(c0_) <= kMaxAscii)) {  //大多数情况下编译的代码中大部分是Ascii字符
    //非转义字符的处理
    if (V8_LIKELY(c0_ != '\\')) {
      uint8_t scan_flags = character_scan_flags[c0_];
      DCHECK(!TerminatesLiteral(scan_flags));
      static_assert(static_cast<uint8_t>(ScanFlags::kCannotBeKeywordStart) ==
                    static_cast<uint8_t>(ScanFlags::kCannotBeKeyword) << 1);
      scan_flags >>= 1;
      /*scanflags的状态位情况
      enum class ScanFlags : uint8_t {
        kTerminatesLiteral = 1 << 0,
        kCannotBeKeyword = 1 << 1,
        kCannotBeKeywordStart = 1 << 2,
        kStringTerminator = 1 << 3,
        kIdentifierNeedsSlowPath = 1 << 4,
        kMultilineCommentCharacterNeedsSlowPath = 1 << 5,
      };
      */
      DCHECK(!IdentifierNeedsSlowPath(scan_flags));//检查
      AddLiteralChar(static_cast<char>(c0_));

      //AdvanceUntil开始块扫描指导扫到需要慢路径处理的字符或者已经扫描完了，每个被扫过的asscii字符都将存入next中去
      AdvanceUntil([this, &scan_flags](base::uc32 c0) {
        if (V8_UNLIKELY(static_cast<uint32_t>(c0) > kMaxAscii)) {
          scan_flags |=
              static_cast<uint8_t>(ScanFlags::kIdentifierNeedsSlowPath);
          return true;
        }
        uint8_t char_flags = character_scan_flags[c0];
        scan_flags |= char_flags;
        if (TerminatesLiteral(char_flags)) {
          return true;
        } else {
          AddLiteralChar(static_cast<char>(c0));
          return false;
        }
      });
      
      //上述快扫结果中如果是token终止符则停下来检查是keyword还是identifier
      if (V8_LIKELY(!IdentifierNeedsSlowPath(scan_flags))) {
        if (!CanBeKeyword(scan_flags)) return Token::kIdentifier;
        // Could be a keyword or identifier.
        base::Vector<const uint8_t> chars =
            next().literal_chars.one_byte_literal();
        return KeywordOrIdentifierToken(chars.begin(), chars.length());
      }

      //上述快扫发现不能快扫了要进入慢路径是置位标志符canbekeyword
      can_be_keyword = CanBeKeyword(scan_flags);
    } else { 
      //否则我们需要处理转义的情况,此时直接开始慢扫
      // Special case for escapes at the start of an identifier.
      escaped = true;

      //把碰到的转义的情况扫出来
      base::uc32 c = ScanIdentifierUnicodeEscape();
      DCHECK(!IsIdentifierStart(Invalid()));
      if (c == '\\' || !IsIdentifierStart(c)) {
        return Token::kIllegal;
      }
      AddLiteralChar(c);
      can_be_keyword = CharCanBeKeyword(c);
    }
  }
  //上述代码全跑完说明代码中间存在需要慢路径处理的token,故进入慢扫
  return ScanIdentifierOrKeywordInnerSlow(escaped, can_be_keyword);
}
```

---

其它函数

```cpp
  //错误相关
  //第一类error:解析error
  // Sets the Scanner into an error state to stop further scanning and terminate
  // the parsing by only returning kIllegal tokens after that.
  V8_INLINE void set_parser_error() {
    if (!has_parser_error()) {
      c0_ = kEndOfInput;
      source_->set_parser_error();
      for (TokenDesc& desc : token_storage_) {
        if (desc.token != Token::kUninitialized) desc.token = Token::kIllegal;
      }
    }
  }
  V8_INLINE void reset_parser_error_flag() {}
  V8_INLINE bool has_parser_error() const {}
  //第二类error:这个error位专门为hex和unicode的转义序列而设置
  bool has_error() const { return scanner_error_ != MessageTemplate::kNone; }
  MessageTemplate error() const { return scanner_error_; }
  const Location& error_location() const { return scanner_error_location_; }
  //第三类error:下面这几个函数围绕着current这个token的invalid_template_escape_message位展开
  bool has_invalid_template_escape() const {}
  MessageTemplate invalid_template_escape_message() const {}
  void clear_invalid_template_escape_message() {}
  Location invalid_template_escape_location() const {}
```

> 第二类和第三类error函数的联系:en

## scanner-inl 中的一些特殊的辅助函数

**GetoneCharToken**函数用来获得单字符可以推断出来的token的类型,例如:'('直接返回**Token::LeftParen**.
其中比较重要的两项如下,其中IsAsciiIdentifer(c)会将小写字母的token判定为kIllegal(IsAsciiIdentier只接受大写字母,'_'和'$'):

```cpp
// IsDecimalDigit must be tested before IsAsciiIdentifier
      IsDecimalDigit(c) ? Token::kNumber :
      IsAsciiIdentifier(c) ? Token::kIdentifier :
```

---


