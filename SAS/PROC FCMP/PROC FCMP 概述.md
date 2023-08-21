# PROC FCMP
## 概述
PROC FCMP 可用于自定义函数（*funcion*）和子程序（*subroutines*）。自定义函数和子程序的名称的最大长度为 32，长度超过 32 的名称虽然可以定义，但无法调用。

创建自定义函数和子程序的优点：
- 使程序易读、易修改
- 使函数和子程序独立于外部环境，其内部实现不影响外部环境
- 使函数和子程序可复用，任何有权限访问存储函数和子程序的数据集的程序均可调用它们

PROC FCMP 定义函数和子程序的时遵循 DATA 步中的语法，定义后的函数和子程序被存储在 SAS 数据集中，可以被其他 SAS 语句调用。

PROC FCMP 是交互式过程，必须使用 QUIT 语句进行终止。
> *PROC FCMP is an **interactive procedure**. You must terminate the procedure with a QUIT statement.*

PROC FCMP 定义的函数和子程序可以被使用在：
- DATA 步
- WHERE 语句
- ODS
- 部分 PROC 步，具体如下：

    - PROC CALIS
    - <font color = red><b>PROC FCMP</b></font>
    - PROC FORMAT
    - PROC GA
    - PROC GENMOD
    - PROC GLIMMIX
    - PROC MCMC
    - PROC MODEL
    - PROC NLIN
    - PROC NLMIXED
    - PROC NLP
    - PROC OPTLSO
    - PROC OPTMODEL
    - PROC PHREG
    - PROC QUANTREG
    - <font color = red><b>PROC REPORT COMPUTE blocks</b></font>
    - SAS Risk Dimensions procedures
    - PROC SEVERITY
    - PROC SIMILARITY
    - <font color = red><b>PROC SQL（不支持带有数组参数的函数）</b></font>
    - PROC SURVEYPHREG
    - PROC TMODEL
    - PROC VARMAX

## 程序包（Package）

通常建议将功能相关的函数和子程序存储在同一个 SAS 数据集中的同一个包（Package）中，包名语法：
*`libname.dataset.package`*。

- `libname` : 逻辑库名称
- `dataset` : 数据集名称
- `package` : 包名

一个数据集中可包含多个包，包名不可重复，同一个包下的函数或子程序名称不可重复，但不同包下的函数或子程序名称可以相同。

为了避免歧义，当指定某个函数或子程序时，如果在不同包下存在相同名称的函数或子程序时，应当额外指定包名，例如：`mufunc1.inverse、myfunc2.inverse`，否则，SAS 会使用最后定义的那个函数，并在日志中发出警告。

![img](./img/PROC%20FCMP%20概述/name-ambiguity-warning.png)

<font color=red><b>注意：</b></font> *`package.function`* 的语法仅在 PROC FCMP 内部生效，在 DATA 步中无法使用该语法，因此，尽量避免出现函数或子程序的重名。

![img](./img/PROC%20FCMP%20概述/two-level-not-valid-in-data-step.png)

## Function 和 Subroutine 的区别
1. function 必须有返回值，subroutine 没有返回值；
2. function 内部无法访问其外部变量，subroutine 可以访问并修改其外部变量的值；
3. 在 DATA 步和 PROC 步中，function 直接使用名称进行调用，subroutine 使用关键字 `CALL` + 名称进行调用；
4. 在宏程序中，function 使用 `%sysfunc()` 进行调用，subroutine 使用 `%syscall` 进行调用；

## Fucntion 和 Subroutine 的声明

PROC FCMP 的语法如下：
```sas
proc fcmp outlib = libname.dataset.package inlib = libname.dataset;
routine-declarations
```

`OUTLIB` 选项指定存储函数和子程序的包名，使用 `INLIB` 选项指定读取函数和子程序的包名。

`routine-declaration` 指定函数和子程序的具体声明内容，一个 PROC FCMP 内部可以同时声明多个函数和子程序。

<font color=red><b>注意：</b></font>创建的函数和子程序名称不应当与内置的 SAS 函数和子程序名称相同。

### 函数的声明
函数的声明由以下四个部分组成：

- 函数名
- 参数（一个或多个）
- 函数体
- 返回值（`RETURN` 语句）

```sas
fucntion name(argument-1 <, argument-2, ...>);
    program-statements;
    return(expression);
endsub;
```
1. 参数包括数值参数和字符串参数，声明字符串参数需要在参数名后面加一个 <font color=red><b>`$`</b></font> 符号；
2. 所有参数都是通过 <font color=red><b>值传递 (passed by value)</b></font> 的，这意味着在调用该函数时，传入函数的实际参数值都是从外部环境直接复制的，这样可以保证函数内部对参数的修改不会影响到外部环境的原始变量值。

### 子程序的声明
子程序的声明与函数大致相同，不同的是，子程序没有返回值。

```sas
subroutine name(<argument-1, argument-2, ...>);
    outargs <out-argument-1, out-argument-2, ...>;
    program-statements;
    return;
endsub;
```

1. 使用 `OUTARGS` 语句声明的参数是通过 <font color=red><b>引用传递 (passed by reference)</b></font> 的，这意味着子程序内部任何对这些参数的修改都会导致外部环境对应变量的值的修改，因为这些参数的值并非来自外部环境的直接复制，事实上，这些变量在子程序的内部和外部共享同一个引用。
    > 当在外部环境与子程序之间存在大量数据的传递时，减少变量的直接复制可以提高性能。(*Reducing the number of copies can improve performance when you pass parge amounts of data between a CALL routine and the calling environment.*)
2. `RETURN` 语句是可选的，当 RETURN 语句执行时，程序立即返回至调用者所处的环境，但 RETURN 语句并未返回任何值。

## 应用实例
例如：ADAE 数据集衍生 `AESTDT` 时，需要基于不良事件结束日期 (`AEENDTC`) 和治疗开始日期 (`TRTSDTC`) 对不良事件开始日期 (`AESTDTC`) 进行填补。

示例数据：
```sas
data ae;
    input AESTDTC :$10. AEENDTC :$10. TRTSDTC :$10.;
cards;
2023-07-UK 2023-08-14 2023-07-11
2023-07-UK 2023-07-07 2023-07-11
2023-08-UK 2023-08-14 2023-07-11
2023-UK-UK 2023-08-14 2023-07-11
2023-UK-UK 2023-07-07 2023-07-11
UKUK-UK-UK 2023-08-14 2023-07-11
UKUK-UK-UK 2023-07-07 2023-07-11
run;
```
下面分别使用 DATA 步、函数、子程序完成数据填补：
### DATA 步
```sas
data ae_data;
    set ae;

    /*拆分年月日*/
    AESTDTC_y = upcase(scan(AESTDTC, 1, "-"));
    AESTDTC_m = upcase(scan(AESTDTC, 2, "-"));
    AESTDTC_d = upcase(scan(AESTDTC, 3, "-"));

    TRTSDTC_y = upcase(scan(TRTSDTC, 1, "-"));
    TRTSDTC_m = upcase(scan(TRTSDTC, 2, "-"));
    TRTSDTC_d = upcase(scan(TRTSDTC, 3, "-"));

    /*进行缺失填补*/
    if AESTDTC_y ^= "UKUK" and AESTDTC_m ^= "UK" and AESTDTC_d = "UK" then do; /*日缺失*/
        if AESTDTC_y = TRTSDTC_y and AESTDTC_m = TRTSDTC_m then do; /*年、月相同*/
            if input(AEENDTC, yymmdd10.) > input(TRTSDTC, yymmdd10.) then do;
                AESTDT = input(TRTSDTC, yymmdd10.);
            end;
            else do;
                AESTDT = mdy(input(AESTDTC_m, 8.), 1, input(AESTDTC_y, 8.));
            end;
        end;
        else do;
            AESTDT = mdy(input(AESTDTC_m, 8.), 1, input(AESTDTC_y, 8.));
        end;
    end;
    else if AESTDTC_y ^= "UKUK" and AESTDTC_m = "UK" and AESTDTC_d = "UK" then do; /*月、日缺失*/
        if AESTDTC_y = TRTSDTC_y then do; /*年相同*/
            if input(AEENDTC, yymmdd10.) > input(TRTSDTC, yymmdd10.) then do;
                AESTDT = input(TRTSDTC, yymmdd10.);
            end;
            else do;
                AESTDT = mdy(1, 1, input(AESTDTC_y, 8.));
            end;
        end;
        else do;
            AESTDT = mdy(1, 1, input(AESTDTC_y, 8.));
        end;
    end;
    else do; /*年、月、日均缺失*/
        if input(AEENDTC, yymmdd10.) > input(TRTSDTC, yymmdd10.) then do;
            AESTDT = input(TRTSDTC, yymmdd10.);
        end;
    end;

    format AESTDT yymmdd10.;
    drop AESTDTC_y AESTDTC_m AESTDTC_d TRTSDTC_y TRTSDTC_m TRTSDTC_d;
run;
```

运行结果：

![img](./img/PROC%20FCMP%20概述/data-step-impute-example.png)

### 函数
```sas
/*自定义函数*/
proc fcmp outlib = sasuser.func.impute;
    function impute_ae(AESTDTC $, AEENDTC $, TRTSDTC $);
        /*拆分年月日*/
        AESTDTC_y = upcase(scan(AESTDTC, 1, "-"));
        AESTDTC_m = upcase(scan(AESTDTC, 2, "-"));
        AESTDTC_d = upcase(scan(AESTDTC, 3, "-"));

        TRTSDTC_y = upcase(scan(TRTSDTC, 1, "-"));
        TRTSDTC_m = upcase(scan(TRTSDTC, 2, "-"));
        TRTSDTC_d = upcase(scan(TRTSDTC, 3, "-"));

        /*进行缺失填补*/
        if AESTDTC_y ^= "UKUK" and AESTDTC_m ^= "UK" and AESTDTC_d = "UK" then do; /*日缺失*/
            if AESTDTC_y = TRTSDTC_y and AESTDTC_m = TRTSDTC_m then do; /*年、月相同*/
                if input(AEENDTC, yymmdd10.) > input(TRTSDTC, yymmdd10.) then do;
                    AESTDT = input(TRTSDTC, yymmdd10.);
                end;
                else do;
                    AESTDT = mdy(input(AESTDTC_m, 8.), 1, input(AESTDTC_y, 8.));
                end;
            end;
            else do;
                AESTDT = mdy(input(AESTDTC_m, 8.), 1, input(AESTDTC_y, 8.));
            end;
        end;
        else if AESTDTC_y ^= "UKUK" and AESTDTC_m = "UK" and AESTDTC_d = "UK" then do; /*月、日缺失*/
            if AESTDTC_y = TRTSDTC_y then do; /*年相同*/
                if input(AEENDTC, yymmdd10.) > input(TRTSDTC, yymmdd10.) then do;
                    AESTDT = input(TRTSDTC, yymmdd10.);
                end;
                else do;
                    AESTDT = mdy(1, 1, input(AESTDTC_y, 8.));
                end;
            end;
            else do;
                AESTDT = mdy(1, 1, input(AESTDTC_y, 8.));
            end;
        end;
        else do; /*年、月、日均缺失*/
            if input(AEENDTC, yymmdd10.) > input(TRTSDTC, yymmdd10.) then do;
                AESTDT = input(TRTSDTC, yymmdd10.);
            end;
        end;

        return(AESTDT);
    endsub;
quit;
```

自定义函数结束后，可直接在 DATA 步中调用：
```sas
options cmplib = sasuser.func;
data ae_fcmp;
    set ae;

    AESTDT = impute_ae(AESTDTC, AEENDTC, TRTSDTC);

    format AESTDT yymmdd10.;
run;
```

或者在 PROC SQL 中调用：
```sas
options cmplib = sasuser.func;
proc sql noprint;
    create table ae_fcmp as
        select
            *,
            impute_ae(AESTDTC, AEENDTC, TRTSDTC) as AESTDT format = yymmdd10.
        from ae;
quit;
```

### 子程序
这里使用 `OUTARGS` 声明了一个对外部变量 `AESTDT` 的引用，使得子程序内部可以直接修改外部变量 `AESTDT` 的值：
```sas
/*自定义子程序*/
proc fcmp outlib = sasuser.func.impute;
    subroutine impute_ae_subrt(AESTDTC $, AEENDTC $, TRTSDTC $, AESTDT);
        outargs AESTDT;
        /*拆分年月日*/
        AESTDTC_y = upcase(scan(AESTDTC, 1, "-"));
        AESTDTC_m = upcase(scan(AESTDTC, 2, "-"));
        AESTDTC_d = upcase(scan(AESTDTC, 3, "-"));

        TRTSDTC_y = upcase(scan(TRTSDTC, 1, "-"));
        TRTSDTC_m = upcase(scan(TRTSDTC, 2, "-"));
        TRTSDTC_d = upcase(scan(TRTSDTC, 3, "-"));

        /*进行缺失填补*/
        if AESTDTC_y ^= "UKUK" and AESTDTC_m ^= "UK" and AESTDTC_d = "UK" then do; /*日缺失*/
            if AESTDTC_y = TRTSDTC_y and AESTDTC_m = TRTSDTC_m then do; /*年、月相同*/
                if input(AEENDTC, yymmdd10.) > input(TRTSDTC, yymmdd10.) then do;
                    AESTDT = input(TRTSDTC, yymmdd10.);
                end;
                else do;
                    AESTDT = mdy(input(AESTDTC_m, 8.), 1, input(AESTDTC_y, 8.));
                end;
            end;
            else do;
                AESTDT = mdy(input(AESTDTC_m, 8.), 1, input(AESTDTC_y, 8.));
            end;
        end;
        else if AESTDTC_y ^= "UKUK" and AESTDTC_m = "UK" and AESTDTC_d = "UK" then do; /*月、日缺失*/
            if AESTDTC_y = TRTSDTC_y then do; /*年相同*/
                if input(AEENDTC, yymmdd10.) > input(TRTSDTC, yymmdd10.) then do;
                    AESTDT = input(TRTSDTC, yymmdd10.);
                end;
                else do;
                    AESTDT = mdy(1, 1, input(AESTDTC_y, 8.));
                end;
            end;
            else do;
                AESTDT = mdy(1, 1, input(AESTDTC_y, 8.));
            end;
        end;
        else do; /*年、月、日均缺失*/
            if input(AEENDTC, yymmdd10.) > input(TRTSDTC, yymmdd10.) then do;
                AESTDT = input(TRTSDTC, yymmdd10.);
            end;
        end;
    endsub;
quit;

```

自定义子程序结束后，可直接在 DATA 步中使用 `CALL` 语句调用。
```sas
options cmplib = sasuser.func;
data ae_fcmp_subrt;
    set ae;
    AESTDT = .;
    format AESTDT yymmdd10.;
    call impute_ae_subrt(AESTDTC, AEENDTC, TRTSDTC, AESTDT);
run;
```

注意，这里应当事先初始化 `AESTDT` 变量，以便 `CALL impute_ae_subrt` 子程序将填补结果存储到变量中。

## CMPLIB 系统选项
在调用自定义函数和子程序之前，先当先通过 `CMPLIB` 系统选项指定一个或多个包含已经编译好的函数和子程序的数据集。
例如：
```sas
options cmplib = sasuser.cmpl;
options cmplib = (sasuser.cmpl sasuser.cmplA sasuser.cmpl3);
options cmplib = (sasuser.cmpl1 - sasuser.cmpl6);
```

## 可变参数（伪）
函数和子程序均支持定义可变参数，在不知道实际传入的参数的个数的情况下十分有用。

使用 `VARARGS` 选项，指定函数或子程序支持可变数量的参数，当指定了 `VARARGS` 时，函数或子程序的最后一个参数应当是一个数组。

例如：定义一个求和函数，参数个数未知。
```sas
proc fcmp outlib = sasuser.func.stats;
    function summation(args[*]) varargs;
        total = 0;
        do i = 1 to dim(args);
            total = total + args[i];
        end;
        return(total);
    endsub;
    a = summation(1, 2, 3, 4, 5);
    put a=;
quit;

option cmplib = sasuser.func;
data _null_;
    array num[5] _TEMPORARY_ (1:5);
    a = summation(num);
    put a=;
run;
```

<font color=red><b>注意：</b></font>在以上函数定义中，PROC FCMP 内部调用 `summation` 函数时，可以使用 `summation(1, 2, 3, 4, 5)` 这种不限参数个数的语法，然而，当需要在 DATA 步中进行调用时，必须事先声明并初始化一个含有多个参数值的数组，然后将数组名称 `array` 作为最后一个参数传入函数中，即 `summation(array)`。

## 递归
由于自定义函数也可以在 PROC FCMP 内部使用，因此，我们可以很方便地借助 PROC FCMP 实现递归。

例1：斐波那契数列。

```sas
proc fcmp outlib = sasuser.func.recursive;
    function Fibonacci(n);
        if n = 1 then do;
            return(1);
        end;
        else if n = 2 then do;
            return(1);
        end;
        else do;
            return(Fibonacci(n - 1) + Fibonacci(n - 2));
        end;
    endsub;
quit;

option cmplib = sasuser.func;
data a;
    do i = 1 to 20;
        Fibonacci = Fibonacci(i);
        output;
    end;
run;
```

输出结果：

![img](./img/PROC%20FCMP%20概述/fibonacci-example.png)

例2：经典汉诺塔游戏
```sas
options pagesize = 50;
proc fcmp outlib = sasuser.func.recursive;
    subroutine Hanoi(n, start $, mid $, end $);
        if n = 1 then do;
            put start " -> " end;
        end;
        else do;
            call Hanoi(n - 1, start, end, mid);
            put start " -> " end;
            call Hanoi(n - 1, mid, start, end);
        end;
    endsub;

    call Hanoi(4, "A", "B", "C");
quit;
```

输出结果：

![img](./img/PROC%20FCMP%20概述/hanoi-example-output.png)

## FCMP 的特殊函数和子程序

PROC FCMP 过程提供了一些特殊的函数和子程序，可以在自定义函数或子程序中调用它们，但不能直接在 DATA 步中进行调用。但是，我们可以对这些特殊的函数和子程序封装为自定义函数和子程序，从而间接实现在 DATA 步中进行调用。

这些特殊的函数和子程序包括：

- 数组相关
    - CALL DYNAMIC_ARRAY
    - READ_ARRAY
    - WRITE_ARRAY
- C 语言相关
    - CALL SETNULL
    - CALL STRUCTINDEX
    - ISNULL
- 执行 SAS 代码
    - <font color=red><b>RUN_MACRO</b></font>
    - RUN_SASFILE
- 方程求根
    - SOLVE
- 矩阵操作
    - CALL ADDMATRIX
    - CALL CHOL
    - CALL DET
    - CALL ELEMENT
    - CALL EXPMATRIX
    - CALL FILLMATRIX
    - CALL IDENTITY
    - CALL INV
    - CALL MULT
    - CALL POWER
    - CALL SUBTRACTMATRIX
    - CALL TRANSPOSE
    - CALL ZEROMATRIX
- 统计相关
    - INVCDF
    - LIMMOMENT

### RUN_MACRO
`RUN_MACRO` 函数用于执行预定义的 SAS 宏，相当于执行 `%macro_name`。这个函数可以实现在 DATA 步中执行 DATA 步。

<font color=red><b>语法：</b></font><b>*rc* = **RUN_MACRO**(*'macro_name'* <, *variable_1*, *variable_2*, ...>)</b>

例如：以下定义了一个按照变量值拆分数据集的宏程序，宏程序中使用了 DATA 步和 PROC DATASETS 过程对数据集进行拆分，使用 `RUN_MACRO` 函数对宏程序进行封装。

宏定义：
```sas
%macro split_dataset;
    %let indata = %sysfunc(dequote(&indata));
    %let var = %sysfunc(dequote(&var));
    %if %sysfunc(exist(subdata_&var)) %then %do; /*数据集存在，继续追加*/
        proc datasets;
            append base = subdata_&var data = &indata(firstobs = &_n_ obs = &_n_);
        quit;
    %end;
    %else %do; /*数据集不存在，创建数据集*/
        data subdata_&var;
            set &indata(firstobs = &_n_ obs = &_n_);
        run;
    %end;
    %let is_split_success = 1;
%mend;
```

PROC FCMP 函数定义，其中变量 `is_split_success` 指示拆分是否成功：
```sas
proc fcmp outlib = sasuser.func.split;
    function split(indata $, var $, _n_) $;
        is_split_success = 0;
        rc = run_macro('split_dataset', indata, var, _n_, is_split_success);
        if rc = 0 and is_split_success = 1 then do;
            return("Success");
        end;
        else do;
            return("Failed");
        end;
    endsub;
quit;
```

调用 `split` 函数：
```sas
data dm;
    input SUBJID $ SITEID $ SEX $ AGE AGRGR $;
cards;
S01001 01 Male 14 <18
S01002 01 Male 33 18~60
S01003 01 Male 76 >60
S01004 01 Female 45 18~60
S01005 01 Female 23 18~60
S02001 02 Male 56 18~60
S02002 02 Female 77 >60
S02003 02 Female 12 <60
S02004 02 Male 33 18~60
S03001 03 Female 44 18~60
S03002 03 Female 62 >60
S04001 04 Female 22 18~60
;
run;


/*调用 SPLIT 函数对数据集进行拆分*/
options cmplib = sasuser.func;
data dm_test;
    set dm;
    length flag $10;
    flag = split("dm", siteid, _n_);
run;
```

在这一个例子中，`split` 函数按照变量 `siteid` 的具体值，将原数据集 `dm` 拆分为 `subdata_01, subdata_02, subdata_03, subdata04` 数据集，分别包含 01~04 中心的受试者信息，`dm_test` 数据集的变量 `flag` 指示当前观测是否被成功拆分到相应的数据集中。

![img](./img/PROC%20FCMP%20概述/run-macro-example.png)

<font color=red><b>注意事项：</b></font>
1. 函数 `RUN_MACRO` 的返回值仅仅代表宏程序被成功提交了，但并不意味着宏程序按照预期执行完成了，建议在宏程序内部声明一个宏变量，用于指示宏程序是否按照预期被执行
2. 参数 `macro_name` 指定需要执行的宏程序名称，应当使用引号包围
3. `variable_1, variable_2, ...` 指定的变量具有以下特征：
    - 在执行 RUN_MACRO 指定的宏程序前，与 PROC FCMP 内变量具有相同名称的宏变量会被定义，并且会使用 PROC FCMP 内相同名称的变量的值进行初始化
    - 在执行 RUN_MACRO 指定的宏程序后，会将宏程序内宏变量的值复制回 PROC FCMP 内具有相同名称的变量中


## Microsoft Excel 函数
SAS 预先实现了很多 Microsoft Excel 中的函数，这些函数可以在 `sashelp.slkwxl` 数据集中找到。使用以下语句可以列出所有 Excel 函数：

```sas
proc fcmp inlib = sashelp.slkwxl listall;
quit;
```

Excel 函数列表：[List of Excel functions available in SAS (via SASHELP.SLKWXL)](https://blogs.sas.com/content/sgf/2020/10/15/using-microsoft-excel-functions-in-sas/#lst-excel-sas)


## 组件对象

SAS 提供了 Component Object Interface，用于在 DATA 步和 PROC FCMP 步中操纵预定义的组件对象（Component Object）。

SAS 为 DATA 步提供了以下预定义的组件对象:
- hash and hash iterator objects（哈希和哈希迭代器对象）
- Java object（Java 对象）
- logger and appender objects

SAS 为 PROC FCMP 步提供了以下预定义的组件对象：
- <font color=red><b>dictionary object</b></font>（字典对象）
- <font color=red><b>hash and hash iterator objects</b></font>（哈希和哈希迭代器对象）
- <font color=red><b>Python objects</b></font>（Python 对象）

组件对象由属性、方法、运算符组成：

- 属性：与对象关联的特定的信息
- 方法：对象能进行的操作
- 运算符：为对象提供特殊的功能

通过句点 `.` 来访问对象的属性和方法，例如：`hash.add()`。

### 哈希

PROC FCMP 提供了哈希对象和哈希迭代器对象，基于查找键 (*lookup keys*) 快速存储、搜索、筛选和检索数据。哈希被认为是在大量数据中进行查找的最快方式。

> *Hashing is considered the fastest way to search a large amount of information that is referenced through keys.*

#### 声明
1. 哈希对象的声明：

- **DECLARE HASH** *object-reference*

```sas
declare hash h;
```

2. 哈希迭代器 (*iterator*) 对象的声明：

- **DECLARE HITER** *object-reference*("*hash-reference*")

```sas
declare hiter iter(h);
```

3. 带有构造器 (*constructor*) 的哈希对象的声明：

- **DECLARE** *object object-reference*(<*argument_tag-1*: *value-1*, *argument_tag-2*: *value-2*, ...>)

`argument_tag: value` 用于指定创建哈希对象的实例时用到的信息，取值为以下 4 种：

- **dataset**: '*dataset_name*<(*datasetoption*)>' : 指定加载到哈希对象的 SAS 数据集名称
- **duplicate**: '*option*' : 指定如何处理重复的键，取值如下：
    - 'replace' | 'r' : 存储最后一个重复的键
    - 'error' | 'e' : 当发现重复键时，在日志中报告错误
- **hashexp**: *n* : 指定哈希表的大小为 2<sup>n</sup>，默认值为 2<sup>8</sup> = 256。
    > 哈希表的大小不等于哈希对象能够存储的键值对的数量。可以将哈希表想象为一个桶（buckets）数组，大小为 256 的哈希表表示有 256 个桶，每个桶能容纳无限多的键值对，当需要存储大量键值对到一个哈希对象时，应当适当扩大 `hashexp` 的大小以提高性能。
- **order**: '*option*' : 指定在哈希对象上使用迭代器时，返回的键值对顺序，取值如下：
    - 'ascending' | 'a' : 顺序排列
    - 'descending' | 'd' : 逆序排列
    - 'YES' | 'Y' : 顺序排列
    - 'NO' | 'N' : 未定义的顺序

```sas
declare hash myhash(dataset: "work.table", duplicate: "r");
```

#### 方法
哈希对象的方法：

- **DEFINEDATA** : 定义哈希对象的值变量
- **DEFINEKEY** : 定义哈希对象的键变量
- **DEFINEDONE** : 指示哈希对象的初始化已完成（键、值变量均已定义）
- **NUM_ITEMS** : 获取哈希对象的键值对数量
- **ADD** : 添加键值对 (key-value pair)
- **REMOVE** : 移除哈希表中指定键的键值对
- **REPLACE** : 替换哈希表中指定键的值
- **CLEAR** : 清除哈希对象中的所有键值对
- **DELETE** : 删除哈希对象
- **CHECK** : 检查哈希对象中是否有指定的键
- **FIND** : 检查哈希对象中是否有指定的键，并返回键对应的值

哈希迭代器对象的方法：

- **FIRST** : 获取哈希对象的第一个键值对
- **LAST** : 获取哈希对象的最后一个键值对
- **NEXT** : 获取哈希对象的下一个键值对
- **PREV** : 获取哈希对象的上一个键值对

### 字典
字典是另一种用于存储数据的方法，它与哈希不同的地方在于：哈希对象仅可存储字符串和数值数据，而字典不仅可以存储字符串和数值，还可以存储数组、哈希对象，甚至其他字典对象。字典能够通过值或者引用存储数据。

通过值存储的数据类型有：

- 数值
- 字符串

通过引用存储的数据类型有：

- 数组
- 哈希对象
- 哈希迭代器对象
- ASTORE 对象
- Python 对象
- 字典对象


#### 声明
字典对象使用以下语法进行声明：

**DECLARE DICTIONARY** *object-reference*

`DICTIONARY` 可以使用缩写 `DNARY` 进行替代。

#### 方法

- **NUM_ITEMS** : 获取字典存储的数据个数
- **DESCRIBE** : 获取字典指定位置处存储的数据信息
    > `DESCRIBE()` 方法接受的第一个参数为一个变量 *array-indicator*，这个变量在方法结束后会获得一个值，用于指示数据是否为数组，可能的取值及其含义如下：
    > - 1 : 指定位置存储的数据是一个数组
    > - 0 : 指定位置存储的数据不是一个数组
    > - MISSING : 指定位置没有存储任何数据

    > `DESCRIBE()` 方法的返回值是一个数值 *data-type*，代表数据存储的类型，可能的取值及其含义如下：
    > - 1 : 双精度浮点数
    > - 2 : 字符
    > - 0 : 缺失
    > - -1 : 字典
    > - -2 : 哈希对象
    > - -3 : 哈希迭代器
    > - -4 : 其他对象
    > - -5 : ASTORE 对象
    > - -6 : Python 对象
- **CLONE** : 通过值存储一个数组
    > 出于性能考虑，在默认情况下，数组是通过引用进行存储的，使用 `CLONE()` 方法可以让字典使用数组的值进行存储
- **REF** : 通过引用存储一个数值或字符串
- **REMOVE** : 移除字典中指定的键值对
- **CLEAR** : 清除字典对象的所有键值对
- **FIRST** : 复制字典的第一个数据，并将迭代器指向第一个位置
- **LAST** : 复制字典的最后一个数据，并将迭代器指向最后一个位置
- **NEXT** : 复制字典的下一个数据，并将迭代器指向下一个位置
- **PREV** : 复制字典的上一个数据，并将迭代器指向上一个位置
- **HASNEXT** : 指示字典是否存在下一个数据
- **HASPREV** : 指示字典是否存在上一个数据
- **SKIPNEXT** : 将迭代器指向下一个位置
- **SKIPPREV** : 将迭代器指向上一个位置

#### 例子
以下代码定义了一个函数 `const()`，返回指定的常量。在函数定义中，声明了一个字典对象，该字典对象包含 3 个浮点数和 2 个字典对象。
```sas
proc fcmp outlib = sasuser.func.math;
    function const(name $);
        declare dnary cst;
            cst["E"] = 2.718281828;
            cst["PI"] = 3.1415926536;
            cst["EULAR"] = 0.5772156649;

            declare dnary cst_max;
                cst_max["SHORT"] = 32767;
                cst_max["INT"] = 2147483647;
                cst_max["LONG"] = 9223372036854775807;
            cst["MAX"] = cst_max;

            declare dnary cst_min;
                cst_min["SHORT"] = -32768;
                cst_min["INT"] = -2147483648;
                cst_min["LONG"] = -9223372036854775808;
            cst["MIN"] = cst_min;

        upname = upcase(name);
        result = .;
        if count(upname, ".") = 0 then do; /*一级索引*/
            dtype = cst.describe(isarr, upname);
            if dtype = 1 then do;
                result = cst[upname];
            end;
        end;
        else if count(upname, ".") = 1 then do; /*二级索引*/
            name1 = scan(upname, 1, ".");
            name2 = scan(upname, 2, ".");

            dtype = cst.describe(isarr, name1);
            if dtype = -1 then do;
                if name1 = "MAX" then do;
                    dtype = cst_max.describe(isarr, name2);
                    if dtype = 1 then do;
                        result = cst_max[name2];
                    end;
                end;
                else if name1 = "MIN" then do;
                    dtype = cst_min.describe(isarr, name2);
                    if dtype = 1 then do;
                        result = cst_min[name2];
                    end;
                end;
            end;
        end;
        return(result);
    endsub;
quit;

/*调用函数 const()*/
option cmplib = sasuser.func;
data a;
    do type = "SHORT", "INT", "LONG";
        do direction = "MIN", "MAX";
            value = const(direction||"."||type);
            output;
        end;
    end;
run;
```

输出结果：

![img](./img/PROC%20FCMP%20概述/fcmp-dict-example.png)

### Python
PROC FCMP 提供了 Python 对象，可以将 Python 函数嵌入到 SAS 程序当中，Python 代码并不会转为 SAS 代码，而是直接使用 Python 解释器进行执行，并将执行结果返回给 SAS。 

#### 环境依赖

**软件要求**：

- SAS 9.4M6 或更高版本
- Python 2.7 或更高版本

**环境变量配置**

参考 [Configuring SAS to Run the Python Language](https://go.documentation.sas.com/doc/zh-CN/bicdc/9.4/biasag/n1mquxnfmfu83en1if8icqmx8cdf.htm) 配置环境变量。

1. 设置环境变量 `MAS_M2PATH`，路径指向 `mas2py.py` 文件的绝对路径，例如：`D:\Program Files\SASHome\SASFoundation\9.4\tkmas\sasmisc\mas2py.py`
2. 设置环境变量 `MAS_PYPATH`，路径指向 Python 可执行文件的绝对路径，例如：`D:\Program Files\Python\python.exe`

![img](./img/PROC%20FCMP%20概述/fcmp-python-environment-variable-config.png)

#### Python 函数工作流

在 PROC FCMP 中使用 Python 对象的典型工作流如下所述：

1. 声明一个 Python 对象
```sas
declare object py(python);
```

2. 将 Python 源代码插入到 SAS 程序中，例如：使用 `SUBMIT INTO` 语句：
```sas
submit into py;
    def MyPyFunc(var1, var2):
        "Output: MyOutputKey"
        newvar = var1 * var2
        return newvar,
endsubmit;
```

3. 发布 Python 源代码
```sas
rc = py.publish();
```

4. 调用 Python 源代码
```sas
rc = py.call("MyPyFunc", var1, var2);
```

5. 返回调用结果。
```sas
MyResult = py.results["MyOutputKey"];
```

#### Python 对象的细节

##### 元组和输出结果
Python 函数定义的函数体中，第一行使用一个字符串对返回值的形式进行定义。字符串以 `"Output: "` 开头，后面跟着代表函数返回值的键，多个返回值之间使用逗号隔开。Python 返回值被存储在一个元组 (tuple) 中，使用键可以对指定的返回值进行访问。例如：下面的例子中定义了一个有两个返回值的函数，并分别使用对应的键获得返回值。

```python
def MyFunction(foo):
    "Output: Python_Return_Key1, Python_Return_Key2"
    Tuple_Element1 = foo * 2
    Tuple_Element2 = foo + 2
    return Tuple_Element1, Tuple_Element2
```

```sas
My_Output1 = py.results["Python_Return_Key1"]
My_Output2 = py.results["Python_Return_Key2"]
```
##### 单行长度限制
PROC FCMP 提交的 Python 源代码的单行长度不能超过 255 个字节。若存在超出 255 字节长度的代码，应当使用字符 "\"，并在下一行继续书写代码

```python
def MyPythonFunc(arg1, arg2, arg3):
    "Output: MyOutputKey"
    Result = arg1 + arg2 - arg3 + \
    arg2 * arg1
    return Result
```

##### 类型转换
SAS 会自动将 Python 代码的返回值转换为合适的数据类型，需要注意的是，SAS 数组不支持混合类型，因此这种转换可能会造成信息丢失。

例如：Python 代码运行后返回一个列表 `[1, 2.3, 4.01]`，SAS 以列表中第一个非空元素的数据类型为基准，将剩余所有元素的类型均转换为这个类型，因此，SAS 将获得数组 `[1, 2, 4]`。

为了避免这种问题，可以尝试在 Python 代码中返回列表 `[float(1), 2.3, 4.01]`。

##### 日期时间
1. 当 Python 返回日期时间结果至 SAS 时，SAS 会将日期时间转换为合适的 SAS 日期时间；
2. 当 Python 接受来自 SAS 的日期时间时，Python 无法自动将 SAS 日期时间转换为合适的 Python 日期时间，因此，需要手动进行转换

```python
def get_date(indate):
    "Output: outdate"
    d = datetime.date(1960, 1, 1) + datetime.timedelta(days = indate)
    return d.strftime('%m/%d/%Y')
```

##### 注释
在 PROC FCMP 中的 Python 代码中，只能使用字符 `#` 开头的注释，诸如 `"""This is my comment"""` 之类的文档注释无法使用。

#### 声明

语法：**DECLARE OBJECT** *object-reference*(**PYTHON**<("*module-name*")>)

**"*module-name*"** 指定存储在 Python 对象中的模块名称，非函数名称。

#### 提交

语法：

- **SUBMIT INTO** *object-reference*; ...*python-code*... **ENDSUBMIT**;
- **SUBMIT INTO** *object-reference*("*file-path*")

#### 方法
- **APPEND** : 在运行时向 Python 对象中追加 Python 代码
- **CALL** : 执行 Python 对象中的 Python 函数
- **CLEAR** : 清空 Python 对象中的已提交代码和返回结果
- **INFILE** : 在解析时向 Python 对象读入外部文件的 Python 源代码
- **PUBLISH** : 向 Python 解释器提交 Python 源代码
- **RESULTS** : 获取 Python 函数返回值构成的字典对象
    - *object-reference*.**RESULTS**["*output-key*"]
    - *object-reference*.**RESULTS.CLEAR()**
- **RTINFILE** : 在运行时向 Python 对象读入外部文件的 Python 源代码
## 函数编辑器

SAS 系统提供了一个 FCMP 函数编辑器的图形界面，可以在 `Solution -> Analysis -> FCmp Function Editor` 中找到。

![img](./img/PROC%20FCMP%20概述/fcmp-function-editor-gui.png)

## 加密
使用 PROC FCMP 自定义的函数和子程序存储在 `OUTLIB` 指定的数据集中，可直接打开查看源代码。PROC FCMP 提供了 `ENCRYPT` 选项实现对源代码的加密。
```sas
proc fcmp outlib = sasuser.func.recursive encrypt;
    subroutine Hanoi(n, start $, mid $, end $);
        if n = 1 then do;
            put start " -> " end;
        end;
        else do;
            call Hanoi(n - 1, start, end, mid);
            put start " -> " end;
            call Hanoi(n - 1, mid, start, end);
        end;
    endsub;

    call Hanoi(4, "A", "B", "C");
quit;
```

加密前：

![img](./img/PROC%20FCMP%20概述/fcmp-encrypt-before.png)
加密后：

![img](./img/PROC%20FCMP%20概述/fcmp-encrypt-after.png)