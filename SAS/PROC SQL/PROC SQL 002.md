上一节中，我们介绍了如何使用 SQL 创建和删除数据集、视图和索引。这一节我们介绍如何使用 SQL 修改数据集的结构，以及更新、新增和删除数据集中的观测。

## 修改数据集的结构

使用 `ALTER TABLE` 可以修改数据集的结构，包括增加、删除变量、修改变量属性，以及对数据完整性约束（_integrity constraints_）的操作。

数据完整性约束涉及到较高级的概念，我们将在未来的章节中介绍它，这一节我们只介绍对变量的增加、删除和修改操作。

### 新增变量

使用 `ADD` 子句可以新增一个变量，我们可以在新增变量的同时指定变量的属性。

```sql
proc sql noprint;
    alter table dm
        add BRTHDAT num label = "出生日期" format = yymmdd10.;
quit;
```

上述代码在数据集 DM 中新增了一个变量 BRTHDAT，并指定了标签和输出格式。

可以在一个 ADD 子句中新增多个变量：

```sql
proc sql noprint;
    alter table dm
        add BRTHDAT num       label = "出生日期" format = yymmdd10.,
            RANDDT   num      label = "随机日期" format = yymmdd10.,
            RNUMBER  num      label = "随机号",
            ARM      char(10) label = "组别";
quit;
```

![img](./img/PROC%20SQL%20002/add-var-output.png)

### 删除变量

使用 `DROP` 子句可以删除一个变量，用法与 `ADD` 子句类似。

```sql
proc sql noprint;
    alter table dm
        drop BIRTHDAT, BMI;
quit;
```

<font color=red><b>注意</b></font>：删除某个变量时，使用该变量定义的索引（包括简单索引和复合索引）都将一并被删除。

### 修改变量属性

使用 `MODIFY` 子句可以修改一个变量的属性，用法与 `ADD` 子句类似。

```sql
proc format;
    value $sex
        "M" = "男"
        "F" = "女";
run;

proc sql noprint;
    alter table dm
        modify RANDDT format = e8601da10.,
               SEX char(10) format = $sex.;
quit;
```

上述代码修改了数据集 DM 中变量 RANDDT 和 SEX 的属性，分别将它们的输出格式修改为 `e8601da10.` 和 `$sex.`，同时将变量 SEX 的长度修改为 10。

<font color=red><b>注意</b></font>：ALTER TABLE 语句无法修改变量名，如需修改这些属性，请使用 CREATE TABLE 或 SELECT 语句间接完成，这将在未来的章节中进一步介绍。

## 更新数据集观测

使用 `UPDATE` 语句可以对数据集中的观测进行更新。`SET` 子句指定更新的变量和数据，`WHERE` 子句指定筛选条件，符合 WHERE 条件的观测才会被更新。如果未指定 WHERE 子句，则会更新所有观测。

```sql
proc sql noprint;
    update dm
        set BMI = HEIGHT/100/WEIGHT**2
        where RANDFL = "Y";
quit;
```

上述代码更新了数据集 DM 中所有已入组（`RANDFL = "Y"`）的受试者的体重指数（BMI）。

SET 子句可以指定 SQL 表达式作为更新后的值，但该 SQL 表达式不能包含逻辑运算符。有关 SQL 表达式的内容将在未来的章节中详细介绍。

## 新增数据集观测

使用 `INSERT` 语句可以在数据集中新增观测。`INTO` 子句指定需新增观测的数据集名称，有两种新增观测的方式：使用 `SET` 或 `VALUES` 子句。

使用 `SET` 子句允许在为变量赋值时，无需考虑变量赋值的顺序；而 `VALUE` 子句在为变量赋值时，赋值顺序必须与 `INSERT INTO` 指定的变量顺序或变量在数据集中的顺序一致。

```sql
proc sql noprint;
    insert into dm (USUBJID, SITEID, SEX, AGE)
        set SEX = "F",
            AGE = 23,
            USUBJID = "S0101",
            SITEID = "01";
    insert into dm (USUBJID, SITEID, SEX, AGE, HEIGHT, WEIGHT)
        values("S0102", "01", "M", 34, 166, 55)
        values("S0201", "02", "F", 45, 173, 65);
quit;
```

上述代码为数据集 DM 新增了 3 行观测，分别为 SET 子句新增的一条观测和 VALUES 子句新增的两条观测。

在新增观测时，INSERT 语句未指定但存在于数据集中的变量将会被赋予缺失值，若 INSERT 语句未指定任何变量，则相当于指定了数据集中的所有变量。例如，下述代码可以将一条完整的观测追加到数据集 DM 的最后一条观测中。

```sql
proc sql noprint;
    insert into dm
        values("S0102", "01", "M", 34, 166, 55, "03AUG2023"d, 0102, "试验组");
quit;
```

## 删除数据集观测

使用 `DELETE` 语句可以对数据集中的观测进行删除。`WHERE` 子句指定筛选条件，符合条件的观测才会被删除。如果未指定 WHERE 子句，则会删除所有观测。

```sql
proc sql noprint;
    delete from dm
        where RANDFL ^= "Y";
quit;
```

上述代码删除了数据集 DM 中所有未入组的受试者（`RANDFL ^= "Y"`）的观测。
