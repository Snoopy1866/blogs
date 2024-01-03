前几节我们介绍了 SELECT 语句的简单查询用法。事实上，SELECT 查询语句本身作为一种表达式（sql expression），自然可以嵌套在其他语句中，SELECT 语句的这种用法被称为子查询（Subqueries）。

子查询可以应用在 PROC SQL 的多个地方，下面介绍一些常见的用法。

### 插入观测

在创建数据集的同时初始化数据集时会用到这种用法，例如：

```sas
proc sql noprint;
    create table test1
        (mean num, std num, min num, max num);
    insert into test1
        set mean = (select mean(age) from sashelp.class),
            std  = (select std(age)  from sashelp.class),
            min  = (select min(age)  from sashelp.class),
            max  = (select max(age)  from sashelp.class);
quit;
```

![](./img/PROC%20SQL%20007/1.png)

上述代码使用 CREATE TABLE 语句创建了一个数据集 test1，然后使用 INSERT INTO 语句向数据集中新增了一条观测，变量 `mean`, `std`, `min`, `max` 的结果分别来自四条 SELECT 子查询的结果。

⚠ 仅当 SELECT 子查询的结果是一行一列时，上述代码才能正常运行，这也符合正常的逻辑（SET 语句一次只能对一个变量赋值为一个字面量）。

下面的代码运行后会在日志中显示错误，这是因为 SET 语句对变量 `mean` 赋值时，使用的 SELECT 子查询语句的查询结果不止一行：

```sas
proc sql noprint;
    create table test1
        (mean num, std num, min num, max num);
    insert into test1
        set mean = (select mean(age) from sashelp.class group by sex),
            std  = (select std(age)  from sashelp.class),
            min  = (select min(age)  from sashelp.class),
            max  = (select max(age)  from sashelp.class);
quit;
```

![](./img/PROC%20SQL%20007/2.png)

如果的确需要将 SELECT 子查询的多行多列结果插入到数据集中，可以改用下面的方法。在下面这个例子中，SELECT 子查询按照变量 `SEX` 分组计算统计量，INSERT INTO 语句将子查询的结果插入到数据集 test1 中：

```sas
proc sql noprint;
    create table test1
        (sex char(4), mean num, std num, min num, max num);
    insert into test1
        select
            sex, mean(age), std(age), min(age), max(age)
        from sashelp.class
        group by sex;
quit;
```

![](./img/PROC%20SQL%20007/3.png)

### 筛选观测

使用 WHERE 语句可以很方便地筛选符合条件的观测，可以将 SELECT 子查询应用在筛选条件中，从而实现对观测的动态筛选。

#### 子查询应用在比较操作符中

```sas
proc sql noprint;
    create table test2 as
        select * from sashelp.class
        where age > (select mean(age) from sashelp.class);
quit;
```

上述代码将子查询的结果作为比较操作符 `>` 的一个操作数，筛选年龄超过平均值的观测。在这个例子中，使用子查询动态筛选的好处是显而易见的：无需事先计算平均年龄，每次运行上述代码时，子查询语句的结果跟随数据集的更新而自动更新。

![](./img/PROC%20SQL%20007/4.png)

对于这个例子，也可以使用其他方法实现，但或多或少存在一些弊端

1. 使用聚集函数计算平均年龄并通过 INTO 语句创建宏变量。由于宏变量的值本质是是文本，INTO 语句将数值结果转化为文本存在精度丢失的问题（可以通过指定高精度的输出格式控制精度损失）；
2. 使用 PROC MEANS 过程计算平均年龄，通过 OUTPUT 语句输出平均年龄。这种方法会额外创建数据集，且使用 where 筛选的时候仍然无法逃避子查询问题。

#### 子查询应用在子集操作符中

```sas
proc sql noprint;
    create table test3 as
        select usubjid, aedecod from adam.adae
        where usubjid in (select usubjid from adam.adsl where arm = "试验组" and fasfl = "Y");
quit;
```

上述代码将子查询的结果作为子集操作符 `IN` 的一个操作数，筛选试验组、FAS 集、发生了不良事件的观测。通常操作符 `IN` 右侧的操作数是一个字面量集合，这里 SELECT 子查询语句先执行，将多行一列的查询结果视为一个集合作为 `IN` 操作符的右侧操作数。

使用下面等价的 DATA 步可以实现相同的功能：

```sas
data test3;
    merge adam.adae adam.adsl;
    by usubjid;
    if not missing(aedecod) and arm = "试验组" and fasfl = "Y";
    keep usubjid aedecod;
run;
```

如果需求稍微复杂一些，比如：筛选试验组、FAS 集、存在合并用药、实验室检查基线异常无临床意义的不良事件的观测，使用 DATA 步虽然也能实现，但会使代码更加复杂且难以理解，使用 SELECT 子查询语句就可以很直观地体现代码的真实意图：

```sas
proc sql noprint;
    create table test3 as
        select usubjid, aedecod from adam.adae
        where usubjid in (select usubjid from adam.adsl where arm = "试验组" and fasfl = "Y") and
              usubjid in (select usubjid from adam.adcm) and
              usubjid in (select usubjid from adam.adlb where bclsig = "异常无临床意义");
quit;
```

通过以上例子，不难发现子查询在取子集操作中具有以下优点：

1. 逻辑清晰，不存在歧义；
2. 无需使用 `by` 语句联结；
3. 无需事先排序；
4. 无需考虑变量如何 KEEP、DROP，需要保留的变量直接在最外侧 SELECT 语句中声明即可。

在实际应用中，当查询条件越复杂时，DATA 步需要考虑的干扰因素越多，子查询的优势越明显，它完全排除了不必要的干扰因素，代码含义非常明确。

### 子查询嵌套

子查询语句可以层层嵌套，当存在多层嵌套的子查询语句时，最内部的子查询语句先执行，逐层向外执行直到最外层的查询语句。

子查询嵌套在复杂查询需求中非常好用，例如：筛选发生了 3 级不良事件的受试者所在组别的所有合并用药记录，代码如下。

```sas
proc sql noprint;
    create table test4 as
        select usubjid, cmtrt from adam.adcm
        where usubjid in (select usubjid from adam.adsl
                          where arm in (select arm from adam.adsl
                                        where usubjid in (select usubjid from adam.adae where aesev = "3级")));
quit;
```

这里嵌套了 3 层子查询，从最内层子查询语句开始，具体执行流程如下：

Step1. 执行 `select usubjid from adam.adae where aesev = "3级"`，筛选发生了 3 级不良事件的受试者编号 `usubjid`；

Step2. 执行 `select arm from adam.adsl where usubjid in [Step1]`，筛选 Step1 查询到的受试者编号所在组别 `arm`，其中 `[Step1]` 为 Step1 子查询结果集合；

Step3. 执行 `select usubjid from adam.adsl where arm in [Step2]`，筛选 Step2 查询到的组别包含的受试者编号 `usubjid`，其中 `[Step2]` 为 Step2 子查询结果集合；

Step4. 执行 `select usubjid, cmtrt from adam.adcm where usubjid in [Step3]`，筛选 Step3 查询到的受试者编号相关的合并用药记录，其中 `[Step3]` 为 Step3 子查询结果集合。

### 关联子查询（Correlated Subqueries）

与一般的子查询不同，关联子查询引用了外部查询的变量，且在执行过程中，关联子查询与外部查询是同时执行的。

#### 关联子查询应用在 EXIST 操作符中

例如：筛选发生了不良事件的受试者信息，这里可以使用操作符 `EXISTS` 配合关联子查询完成。

```sas
proc sql noprint;
    create table test5 as
        select usubjid, sex, age from adam.adsl as a
        where exists (select usubjid from adam.adae as b where a.usubjid = b.usubjid);
quit;
```

上述代码中，子查询语句 `select usubjid from adam.adae as b where a.usubjid = b.usubjid` 引用了外部查询的 `a.usubjid` 列，对于 `adam.adsl` 的每一条观测，子查询语句都会执行一次。

例如：当执行到观测 `a.usubjid = "S01001"` 时，会首先执行一次子查询，此时子查询语句变为 `select usubjid from adam.adae as b where a.usubjid = "S01001"`，实际上这里可以看做退化为一个普通的子查询语句，将查询结果应用到操作符 `EXISTS` 上，然后执行外部查询。上述操作会在数据集 `adam.adsl` 的每一条观测上重复进行。

⚠ 上述例子只是为了说明 `EXISTS` 操作符的用法，实际上这里可以用 `IN` 操作符，将关联子查询改为普通子查询，执行效率更高：

```sas
proc sql noprint;
    create table test5 as
        select usubjid, sex, age from adam.adsl
        where usubjid in (select usubjid from adam.adae);
quit;
```

#### 关联子查询应用在比较操作符中

例如：查询发生日期早于随机日期的不良事件记录。

```sas
proc sql noprint;
    create table test6 as
        select usubjid, aedecod, aestdt from adam.adae as a
        where a.aestdt < (select randdt from adam.adsl as b where a.usubjid = b.usubjid);
quit;
```

上述代码中，关联子查询语句 `select randdt from adam.adsl as b where a.usubjid = b.usubjid` 对外部查询的每一条观测都查询一次随机日期，将查询结果传递给外部查询语句的 `where` 条件。

#### 关联子查询计算简单统计量

例如：查询 FAS 集中各受试者发生的不良事件次数。

```sas
proc sql noprint;
    create table test7 as
        select
            usubjid,
            (select count(aedecod) from adam.adae as b where a.usubjid = b.usubjid) as aen label = "AE次数"
        from adam.adsl as a where fasfl = "Y";
quit;
```

上述代码中，关联子查询语句 `select count(aedecod) from adam.adae as b where a.usubjid = b.usubjid` 对外部查询的每一条观测都计算一次 AE 次数，将查询结果赋值给变量 `aen`。

#### 关联子查询嵌套在普通子查询中

例如：查询不良事件发生日期早于随机日期的受试者的合并用药记录。

```sas
proc sql noprint;
    create table test8 as
        select usubjid, cmtrt from adam.adcm
        where usubjid in (select usubjid from adam.adae as a
                          where a.aestdt < (select randdt from adam.adsl as b where a.usubjid = b.usubjid));
quit;
```

上述代码中，普通子查询语句的 `where` 条件嵌套了一个关联子查询。对于这里的关联子查询，其外部查询是普通子查询，关联子查询语句 `select randdt from adam.adsl as b where a.usubjid = b.usubjid` 对普通子查询的每一条观测都查询一次随机日期，将结果传递给普通子查询语句的 `where` 条件。普通子查询语句执行完成后，将查询结果传递给外部查询语句的 `where` 条件。
