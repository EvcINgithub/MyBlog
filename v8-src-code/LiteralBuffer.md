# 工具类LiteralBuffer

字面量literal既包括字符串字面量也包括标识符等。LiteralBuffer的作用在于缓存字符直到能够读入完整的字面量。
unicode的好处是能够对多种语言进行统一编码标识但是utf-8足以表示英文字符但是却不够中文及其它的字符此时需要utf-16.为了节约空间此时LiteralBuffer采用了自动使用1字节或2字节存入buffer的形式。

LiteralBuffer正如其名时相当纯粹的buffer,最主要的作用是让构建字面量不必困于如何减少字面量内存大小只需要按部就班的加入字符，然后根据is_one_byte()函数把打包好的字面量或提取出原始底层数据或构建相应的v8::string
实现：

```CPP
  static constexpr int kInitialCapacity = 256;  //缓冲的起始容量
  static constexpr int kGrowthFactor = 4;// 增长参数，乘法 
  static constexpr int kMaxGrowth = 1 * MB;//最大大小
  base::Vector<uint8_t> backing_store_; ///类中真正存储的字符的vector
  int position_ = 0;//代表字符量在backing_store_中的位置
  bool is_one_byte_ = true;  //控制字面量的每个字符是1字节还是2字节

  //其它功能函数

  //添加构建字面量
  public: 
  V8_INLINE void AddChar(char code_unit){}//添加1字节的字符
  V8_INLINE void AddChar(base::uc32 code_unit){}//添加1或者2字节的字符，但是其实32可以放下4字节的字符但实际使用中并没有处理4字节字符。在需要添加2字节字符时该函数会调用ConvertToTwoByte改变当前字面量的编码方式并设置is_one_byte_=false，比上一种实现更好的应对不同的编码
  private:
  V8_INLINE void AddOneByteChar(uint8_t one_byte_char){}//单字节的实现方式，但是居然只能实现像单字节编码的字面量缓冲中加入（通过DCHECK(is_one_Byte())
  void AddTwoByteChar(base::uc32 code_unit);//2字节(1字节也被视作2字节)加入

  //维护字面量缓冲区的大小
  int LiteralBuffer::NewCapacity(int min_capacity){}//计算返回扩展缓冲区时目标大小
  void LiteralBuffer::ExpandBuffer(){}//扩展缓冲区，方式类似std:::vector不过大小NewCapacity的处理不同，有待后续考量
  void LiteralBuffer::ConvertToTwoByte(){}//字面量编码发失改变

  //返回字面量的原始存储
  template <typename Char>
  base::Vector<const Char> literal() const {}//根据模板决定返回的字面量，为什么要clone返回很奇怪
  
  //两个封装函数
  base::Vector<const uint16_t> two_byte_literal() const {
    return literal<uint16_t>();
  }

  base::Vector<const uint8_t> one_byte_literal() const {
    return literal<uint8_t>();
  }

  //  转换为V8的string
  template <typename IsolateT>
  DirectHandle<String> Internalize(IsolateT* isolate) const;//将收集的字符序列转换为 V8 内部的 String 对象

  /*
    
  template DirectHandle<String> LiteralBuffer::Internalize(
    Isolate* isolate) const;
  template DirectHandle<String> LiteralBuffer::Internalize(
    LocalIsolate* isolate) const;
    */
```

>提供了isolate的两种类型有待细究
