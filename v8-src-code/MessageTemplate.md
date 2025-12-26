# 枚举类MessageTemplate

该类枚举全部可能的错误(类似Token)，实现：

```CPP
enum class MessageTemplate {
//name为错误类型，string为对应的报错语句
#define TEMPLATE(NAME, STRING) k##NAME,
  MESSAGE_TEMPLATES(TEMPLATE)
#undef TEMPLATE
      kMessageCount
};

//查询int对应的错误类型
inline MessageTemplate MessageTemplateFromInt(int message_id) {
  DCHECK_LT(static_cast<unsigned>(message_id),
            static_cast<unsigned>(MessageTemplate::kMessageCount));
  return static_cast<MessageTemplate>(message_id);
}

```
