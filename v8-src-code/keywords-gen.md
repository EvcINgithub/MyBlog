# 工具 keywords-gen

gperf 是一个完美哈希函数生成器，它可以为一组固定的字符串,keywords-gen中包含gperf生成的用于对关键字进行哈希编码的辅助类PerfectKeywordHash

```cpp
class PerfectKeywordHash {
 private:
  static inline unsigned int Hash(const char* str, int len);

 public:
  static inline Token::Value GetToken(const char* str, int len);
};
```

其中Hash函数用于计算str对应的hash值，GetToken则会返回str的类别(**keyword** or **identifier**)