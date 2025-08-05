+++
title = "PostgreSQL JDBC 中 stringtype=unspecified 背后的设计趣事"
date = 2025-08-05
description = "分享一个JDBC驱动有趣的小彩蛋：JDBC 的 stringtype=unspecified 参数和 PostgreSQL 类型系统的奇妙互动。"
draft = false
[taxonomies]
categories = ["技术杂谈"]
tags = ["PostgreSQL", "JDBC", "趣事", "类型系统", "设计理念", "驱动"]
+++

有时候，写代码就像在解谜。你以为一切都按部就班，结果 PostgreSQL 给你来一句：

```
ERROR: column "meta" is of type jsonb but expression is of type character varying
```

这时候，网上一搜，大家都在说：在 JDBC 连接串里加上 `stringtype=unspecified` 就好了。

你试一试，果然，世界安静了。

这事儿有点意思。明明只是多了一个参数，数据库和驱动之间的气氛就突然变得融洽起来。你有没有好奇过，这背后到底发生了什么？

<!--more-->

### 类型的"执拗"与"默契"

PostgreSQL 是个很有原则的家伙。它对类型的执着，像极了老派工匠：jsonb 就是 jsonb，varchar 就是 varchar，别想糊弄过去。哪怕你传进去的字符串长得再像 JSON，也得明说"我这是 JSON"。

JDBC 驱动呢？它有点像个中间人，默认情况下很谨慎：你说是 String，我就发 varchar，绝不自作主张。

但 `stringtype=unspecified` 这个小开关一开，JDBC 就像说："这回我不管了，PostgreSQL 你自己看着办吧。" 于是数据库一看，哦，目标字段是 jsonb，那我就当 JSON 处理。

---

### 驱动实现里的秘密

如果你好奇 JDBC 驱动是怎么实现这个"不管了"的，可以看看源代码。在 PostgreSQL JDBC 驱动的 GitHub 仓库里，这个功能主要在几个关键文件中实现：

**1. 连接参数处理** ([PgConnection.java](https://github.com/pgjdbc/pgjdbc/blob/master/pgjdbc/src/main/java/org/postgresql/jdbc/PgConnection.java#L327))
```java

String stringType = PGProperty.STRING_TYPE.getOrDefault(info);
if (stringType != null) {
    if ("unspecified".equalsIgnoreCase(stringType)) {
        bindStringAsVarchar = false;
    } else if ("varchar".equalsIgnoreCase(stringType)) {
        bindStringAsVarchar = true;
    } else {
        throw new PSQLException(
            GT.tr("Unsupported value for stringtype parameter: {0}", stringType),
            PSQLState.INVALID_PARAMETER_VALUE);
    }
} else {
    bindStringAsVarchar = true;
}
```

**2. 参数类型设置** ([PgPreparedStatement.java](https://github.com/pgjdbc/pgjdbc/blob/master/pgjdbc/src/main/java/org/postgresql/jdbc/PgPreparedStatement.java#L388))
```java

public void setString(@Positive int parameterIndex, @Nullable String x) throws SQLException {
    checkClosed();
    setString(parameterIndex, x, getStringType());
}

private int getStringType() {
    return connection.getStringVarcharFlag() ? Oid.VARCHAR : Oid.UNSPECIFIED;
}
```

**3. 协议层实现** ([QueryExecutorImpl.java](https://github.com/pgjdbc/pgjdbc/blob/master/pgjdbc/src/main/java/org/postgresql/core/v3/QueryExecutorImpl.java#L1659))
```java

private void sendBind(SimpleQuery query, SimpleParameterList params, @Nullable Portal portal,
	 boolean noBinaryTransfer) throws IOException {
    // Send Bind.
    // 当参数类型为 Oid.UNSPECIFIED (0) 时
    // PostgreSQL 会根据目标字段类型自动推断
}
```

默认情况下，当你调用 `ps.setString(1, "{\"name\": \"Ben\"}")` 时，驱动会发送：
```
BIND
  parameter: 1
  type: 1043 (varchar)
  value: {"name": "Ben"}
```

但设置了 `stringtype=unspecified` 后，驱动会发送：
```
BIND
  parameter: 1
  type: 0 (unspecified)
  value: {"name": "Ben"}
```

这个 `type: 0` 就是关键。PostgreSQL 看到类型是 0（unspecified），就会根据目标字段的类型来决定怎么处理这个参数。

---

### PostgreSQL 的 JSON 解析魔法

当 PostgreSQL 收到类型为 unspecified 的参数时，它会调用相应的类型转换函数。对于 jsonb 字段，会调用 `jsonb_in()` 函数。

在 PostgreSQL 的源码中，这个处理过程主要在以下文件中：

**1. JSON 解析实现** ([jsonb.c](https://github.com/postgres/postgres/blob/master/src/backend/utils/adt/jsonb.c#L73))
```c

Datum
jsonb_in(PG_FUNCTION_ARGS)
{
    char       *json = PG_GETARG_CSTRING(0);
    // 1. 检查字符串是否是有效的 JSON
    // 2. 解析 JSON 结构  
    // 3. 转换成内部的 jsonb 格式
    return jsonb_from_cstring(json, strlen(json), false, fcinfo->context);
}
```

**2. 类型转换逻辑** ([jsonfuncs.c](https://github.com/postgres/postgres/blob/master/src/backend/utils/adt/jsonfuncs.c#L518))
```c

bool
pg_parse_json_or_errsave(JsonLexContext *lex, const JsonSemAction *sem,
						 Node *escontext)
{
	JsonParseErrorType result;

	result = pg_parse_json(lex, sem);
	if (result != JSON_SUCCESS)
	{
		json_errsave_error(result, lex, escontext);
		return false;
	}
	return true;
}
```

所以当你传入 `"{\"name\": \"Ben\"}"` 时，PostgreSQL 会：
- 发现目标字段是 jsonb
- 调用 jsonb_in() 解析这个字符串
- 验证 JSON 格式正确
- 成功插入

这就是为什么同样的字符串，有时候会报错，有时候又能成功的原因。

---

### 小参数，大世界

说到底，这就是个参数的事儿。但仔细想想，还挺有意思的。

一个参数，就能让两个系统之间的对话方式完全改变。JDBC 从"我帮你决定类型"变成"你自己看着办"，PostgreSQL 从"我不接受模糊"变成"我来推断一下"。

这种设计上的小细节，有时候比那些宏大的架构图更有意思。

---

### 下次遇到，不妨会心一笑

所以，下次再遇到 `stringtype=unspecified`，别急着吐槽"又出幺蛾子了"。也许，这正是数据库世界里的一点小幽默。

毕竟，写代码嘛，除了让它跑起来，偶尔乐在其中，也挺好。 
