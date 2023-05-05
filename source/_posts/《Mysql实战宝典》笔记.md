---
title: 《Mysql实战宝典》笔记
date: 2023-05-05 21:01:46
tags: [数据库,mysql]
categories:  数据库
---

# 数字类型

![image-20230505212034763](https://raw.githubusercontent.com/fwm1/PicturesRepo/master/blog-images/20230505212036.png)

- 不推荐使用整型类型的属性 unsigned，若非要使用，参数 sql_mode 务必额外添加上选项 NO_UNSIGNED_SUBTRACTION；
- 自增整型类型做主键，务必使用类型 BIGINT，当达到自增整型类型的上限值时，再次自增插入，MySQL 数据库会报重复错误；
- **注意**：MySQL 8.0 版本前，自增整型会有回溯问题；
- **不要**使用浮点类型 Float、Double，MySQL 后续版本将不再支持上述两种类型；
- 账户金额字段，可以考虑用整型类型，而不是 DECIMAL 类型，这样性能更好，存储更紧凑。



# 字符类型

- CHAR 和 VARCHAR 虽然分别用于存储定长和变长字符，但对于变长字符集（如 GBK、UTF8mb4），其本质是一样的，都是变长，**设计时完全可以用 VARCHAR 替代 CHAR；
- 排序规则很重要，用于字符的比较和排序，但大部分场景不需要用区分大小写的排序规则；
- 修改表中已有列的字符集，使用命令 **ALTER **TABLE  table_1 **CONVERT** **TO** CHARSET utf8mb4；
- 用户性别，运行状态等有限值的列，MySQL 8.0.16 版本可以使用 CHECK 约束机制，之前的版本可使用 ENUM 枚举字符串类型，外加 SQL_MODE 的严格模式；
- 加密字段处理。简单的MD5算法可以进行暴力破解，推荐使用动态盐+动态加密算法进行隐私数据的存储。比如动态拼接：$salt$encry_type$encryped_content 
