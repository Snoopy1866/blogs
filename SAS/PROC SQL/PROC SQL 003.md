前两节中，我们介绍了如何使用 SQL 创建、删除数据集、修改数据集结构，以及如何新增、删除和更新数据集的观测。前两节所涉及到的内容都是对数据集的增、删、改的操作，从本节开始，我们将对 SQL 中最常见，也最灵活的查询操作进行详细的介绍。

## 查询语句

SQL 的查询操作是通过 `SELECT` 语句实现的。SELECT 语句包含很多子句，本节介绍的是最简单的查询语句。

### 查询一个或多个变量

```sql
proc sql;
    select USUBJID, SITEID, SEX, AGE
        from DM;
quit;
```

上述代码使用 SELECT 语句从数据集 DM 中查询了 4 个变量，`FROM` 子句指定查询的源，即从哪个数据集或视图中进行查询。

### 查询所有变量

```sql
proc sql;
    select * from DM;
quit;
```

上述代码使用 SELECT 语句从数据集 DM 中查询了所有变量，其中 `*` 代表 FROM 子句指定的数据集 DM 中的所有变量，这在数据集变量名未知的情况下非常有用。

💡 不可滥用 `*` ，当查询的数据源包含大量变量和观测时，使用 `*` 会造成性能的严重下降。通常来说，建议使用最小化原则，需要使用什么变量就查询什么变量，无需使用的变量不应当出现在 SELECT 语句中。

### 查询不重复的观测

默认情况下，SELECT 语句会查询并返回所有匹配的观测，某些变量在不同的观测上可能存在相同的值，如果我们只需要变量的值不重复的观测，可以使用 `DISTINCT` 或 `UNIQUE` 关键字去除重复的观测，DISTINCT 与 UNIQUE 效果相同。

```sql
proc sql;
    select distinct USUBJID, SITEID, SITENAME, ARM from AE;
quit;
```

上述代码使用 SELECT 语句从数据集 AE 中查询所有发生了不良事件的受试者信息，由于同一受试者可能发生多次不良事件，因此使用 DISTINCT 关键字去除重复的观测，每位受试者仅保留一条观测。

💡 一条 SELECT 语句中只能有一个 DISTINCT 关键字，DISTINCT 关键字必须出现在所有变量名的前面。虽然 DISTINCT 在 SELECT 语句中看起来只对紧随其后的第一个变量生效，但事实上，它作用于 SELECT 语句中出现的所有变量（包括未命名的变量）。这意味着，只有当 SELECT 语句中的变量组合存在重复的观测时，查询结果中才会去除这些重复的观测，而并非在查询的第一个变量出现重复值的观测时就立即起作用。

💡 由于数值精度的问题，SELECT DISTINCT 查询的结果可能会出现变量的显示值完全一致的观测。

### 使用查询结果创建新的数据集

在第一节中，我们提到创建数据集的两种方法：

1. 通过直接定义变量属性的方式创建数据集
2. 使用 `LIKE` 关键字基于其他数据集的结构创建数据集

其实，我们也可以利用 SELECT 语句的查询结果来创建数据集。当需要对 SELECT 语句的查询结果作进一步处理时，就可以将查询结果存储为新的数据集。

```sql
proc sql;
    create table AE_UID as
        select distinct USUBJID, SITEID, SITENAME, ARM from AE;
quit;
```

上述代码使用 SELECT 语句从数据集 AE 中查询所有发生了不良事件的受试者信息，并存储在新的数据集 AE_UID 中。

### 限定数据集名称

前面几个例子中，我们在 SELECT 语句中仅指定了查询的变量名，而没有指定变量所在的数据集，在这种情况下，FROM 子句的返回结果将作为 SELECT 语句查询的数据源。在以后的章节中，我们会看到在某些情况下，不指定数据集名称可能会造成歧义，此时必须在 SELECT 语句中限定变量名所在的数据集名称，使用 `数据集.变量名` 的格式引用指定数据集中的变量。

```sql
proc sql;
    select
        DM.USUBJID,
        DM.SITEID,
        DM.SEX,
        DM.AGE
    from DM;
quit;
```
