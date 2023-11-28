SQL 全称 Strucured Query Language，即结构化查询语言，广泛应用于关系型数据库中。

SAS Base 使用 PROC SQL 提供了对 SQL 的实现。PROC SQL 过程可以帮助我们完成以下任务：

- 创建数据集、视图和索引
- 删除数据集、视图和索引
- 修改数据集的结构
- 修改数据集的观测
- 从数据集或视图中获取观测
- 从数据集或视图中合并观测
- 汇总统计

上述任务可以用四个字简要概括：<font color=red><b>增、删、改、查</b></font>。

在这一节中，我们主要介绍如何使用 SQL 创建和删除数据集、视图和索引。

## 创建数据集

使用 `CREATE TABLE` 语句可以创建一个数据集。例如：下述代码创建了一个新的数据集。

```sql
proc sql noprint;
    create table DM
        (USUBJID char(20), SITEID char(10), SEX char(4), AGE num, HEIGHT num, WEIGHT num);
quit;
```

该数据集名称为 DM，包含 3 个字符型变量和 3 个数值型变量。变量的定义包括变量名和变量类型，变量名可以是任何合法的 SAS 名称，变量类型可以是 `CHARACTER` 和 `NUMERIC`，可以分别简写为 `CHAR` 和 `NUM`。

此外，在定义变量的时候还可以额外定义变量修饰符，包括：变量标签、输入格式、输出格式等，例如：

```sql
proc sql noprint;
    create table DM
        (USUBJID char(20) informat = $20. format = $20.     label = "受试者唯一标识符",
         SITEID  char(10) informat = $10. format = $10.     label = "中心编号",
         SEX     char(4)  informat = $4.  format = $4.      label = "性别",
         AGE     num      informat = 8.   format = 8.       label = "年龄(岁)",
         HEIGHT  num      informat = 8.2  format = 8.2      label = "身高(cm)",
         WEIGHT  num      informat = 8.2  format = 8.2      label = "体重(kg)");
quit;
```

运行后查看数据集 DM 属性：

![img](./img/PROC%20SQL%20001/create-table-output.png)

`CREATE TABLE` 语句还支持基于现有数据集结构创建空白数据集，只需使用 `LIKE` 关键字即可：

```sql
proc sql noprint;
    create table DM1 like DM;
quit;
```

上述代码将会创建一个名为 DM1 的数据集，其结构与数据集 DM 完全一致，但不含任何观测。

## 创建视图

视图本质上是一段 PROC SQL 的查询语句，本身并不包含任何数据集中的任何数据，当在 SAS 过程或 DATA 步中使用视图时，视图包含的查询语句将会自动执行。这意味着每次访问视图时，查询到的数据都有可能是不同的，即视图是动态更新的。

当某个查询语句被频繁使用时，可以为其创建视图，以供其他查询语句访问，而无需每次访问时编写重复的查询语句。

使用 `CREATE VIEW` 可以创建一个视图。例如：下述代码创建了一个视图，该视图查询了数据集 DM 中年龄 ≥ 60 岁的受试者信息。

```sql
proc sql noprint;
    create view age_gt60 as
        select usubjid, siteid, sex, age from dm
            where age >= 60;
quit;
```

注：上述代码使用了 `SELECT` 语句进行数据库的查询，我们将在未来的章节中介绍它。

## 创建索引

索引是一种数据结构，可以将其看做书的目录，它存储了指向特定观测的指针，以便可以通过指针快速定位特定数据。使用索引可以提高 PROC SQL 在特定情况下执行查询语句的效率。

索引又分为简单索引（_simple index_）和复合索引（_composite index_）。

**简单索引**：在一个变量上创建，索引名称必须与变量名相同；
**复合索引**: 在两个或多个变量上创建，索引名称不能与数据集中的任何变量名相同。

使用 `CREATE INDEX` 可以创建一个索引。例如：下述代码为数据集 DM 中的受试者唯一编号建立了索引。

```sql
proc sql noprint;
    create index usubjid
        on DM(USUBJID);
quit;
```

复合索引适用于无法使用单一变量标识唯一观测的数据集中（ADLB，ADAE 等），例如：

```sql
proc sql noprint;
    create index aeindex
        on adae(usubjid, aeseq);
quit;
```

在数据集属性信息的“索引”标签中可以查看已定义的索引信息：

![img](./img/PROC%20SQL%20001/create-index-output.png)

## 删除数据集、视图、索引

使用 `DROP` 语句可以删除数据集、视图和索引。

```sql
proc sql noprint;
    drop table dm1;
    drop view age_gt60;
    drop index usubjid from dm;
quit;
```

删除操作需要注意以下问题:

1. 删除数据集或视图后，所有引用了该数据集或视图的视图都将失效；
2. 删除含有索引的数据集后，该数据集中的所有索引也将一并被删除；
3. 删除一个复合索引后，其所关联的变量都将失去该复合索引，但会保留其他索引。
