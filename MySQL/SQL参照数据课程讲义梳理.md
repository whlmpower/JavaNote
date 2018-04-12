### 建立与删除索引
数据库扫描数据的方式：   
表扫描   
索引查找    

### 创建索引的优势  
创建索引可以大大提高系统的性能   
加快数据的检索速度   
在使用order by 和 group by 进行数据检索时，可以显著减少查询中分组和排序的时间   
通过使用索引，可以在查询中，使用优化隐藏器，提高系统查询性能   

### 建立索引的缺点
创建索引和维护索引需要消耗时间   
索引占据物理空间   
对表数据进行增删改查，索引也需要动态维护  

### 聚集索引
数据表的物理顺序和索引顺序相同的索引   
聚簇索引的顺序就是数据的物理存储顺序   
每个表只允许建立一个聚簇索引   
建立聚簇索引时要改变表中的数据行的物理顺序  
默认情况下是Primary Key 创建的是聚簇索引（无主键， 找唯一列） 

### 建立与删除索引 
**适宜建立聚集索引的场景**  
在尽可能少的列上定义一个聚集索引  
经常指定的列来排序，该列是聚集索引最佳候选列   
列具有相对较少的相异值   
如果表中的值都是按照顺序进行访问的  
搜索用的列是聚簇索引的最佳候选列   

**不适宜建立聚集索引**   
进行大量数据变动的表，因为DBMS将不得不在表中维护行的次序   

### 建立foreign key
````sql
    create table order(
    O_id int not null,
    OrderNo int not null,
    Id_P, int, 
    primary key (O_id),
    foreign key (Id_P) references Persons(Id_P)
    )
````
## SQL 查询
distinct(作用范围是所有目标列)
涉及空值的查询 IS NULL 不能用 = NULL 来代替   
当排序列为空值的元组 ASC 最后显示， DESC 最先显示   
使用集函数 distinct count sum avg max min    
### group by 子句   
细化集函数的作用对象    
未对查询结果分组，集函数将作用于整个查询结果    
对查询结果分组后，集函数将分别作用于每个组   
````sql
    select Cno, COUNT(Sno) from SC Group by Cno
````
所有相同的Cno值的元组为一组，然后对每一组作用使用集函数count   
使用 Group by 子句后，SELECT子句的列名列表中只能出现**分组属性**和**集函数**   
分组后还要求按一定的条件对这些组进行筛选，可以使用HAVING   
```sql
    select sno from sc group by sno having count(*) > 3
```
查询有3门以上课程是90分以上的学生学号及90分以上的课程数
```sql
    select sno, count(*) from sc where grade >= 90 group by sno having count(*) >= 3
```
having 与 where子句的区别：   
where子句是基于表或者是视图；having短语是作用于组。   
having短语跟在group by 后面，条件中必须包含库函数，否则可以直接放到where中。   
where 一定是在having之前执行   
执行顺序：   
1. 先选择合适的where条件的元组
2. 对选出的元组进行分组 
3. 使用having对选出的元组进行过滤   

### 连接查询
1. 内连接：默认的连接类型，只输出满足连接条件的元组（等值连接是一种特殊情况，把目标列中重复的属性列去掉）   
2. 外连接：以指定表为连接主体，将主体表中**不满足连接**条件的元组一并输出  
3. 交叉连接：两个关系的笛卡尔积   

***连接查询（复合连接）***   
两个表之间无连接条件，引入第三表   
```sql
    select student.sno, sname, cname from student, course, sc where student.sno = sc.sno and sc.cno = course.cno
```
图书（++总编号++，分类号，书名，作者，出版单位，单价）  
读者（++借书证号++，姓名，性别，单位，职称，地址）  
借阅（++借书证号， 总编号++，借阅日期，备注）   
1. 分别找出各个单位当前借阅图书的读者人次
```sql
select 单位, '借书人数: ', count(借书证号) from 借阅, 读者 where 读者.借书证号 = 借阅.借书证号 group by 单位
```
2. 分别找出借书人数超过10个人的单位及人数   
```sql
    SELECT 单位,’借书人数:’ , COUNT(DISTINCT 借书证号)
    FROM 借阅,读者
    WHERE 读者.借书证号=借阅.借书证号
    GROUP BY 单位
    HAVING COUNT（DISTINCT 借书证号） >10
```
3. 找出当前至少借阅了5本图书的读者及所在单位   
```sql
    SELECT 姓名，单位
    FROM 读者，借阅
    WHERE 读者.借书证号=借阅.借书证号
    GROUP BY 借书证号
    HAVING COUNT（*） >=5
```
上述写法是错误的，因为使用了group by 子句后，SELECT语句中只允许出现**分组属性**或者**集函数**   
正确写法：   
```sql
    select 姓名，单位 from 读者 where 借书证号 in (select 借书证号 from 借阅 group by 借书证号 having count(*) > 5)
```
***连接查询（自身连接）***
1. 查询同时选修了c1和c2两门课的学生学号。
```sql
    SELECT sc1.Sno
    FROM SC sc1, SC sc2
    WHERE sc1.Cno=‘c1’ AND sc2.Cno=‘c2’
    AND sc1.Sno=sc2.Sno
```
2. 查询每一门课的间接先修课（即先修课的先修课）把两个表想象成完全一样的表   
```sql
    SELECT FIRST.Cno， SECOND.Cpno
    FROM Course FIRST， Course SECOND
    WHERE FIRST.Cpno = SECOND.Cno
```
***连接查询（外链接）***   
关于外链接的展开   
外连接操作以指定表为连接主体，将主体表中不满足连接条件的元组一并输出   
左外连接： 内连接 + 左边关系中失配的元组（缺少的右边关系属性值用null表示）   
右外连接： 内连接 + 右边关系中失配的元组（缺少 的左边关系属性值用null表示）     
全外连接： 内连接 + 左边关系中失配的元组（缺少 的右边关系属性值用null表示） + 右边关系中失配的 元组（缺少的左边关系属性值用null表示）   

左外连接(LEFT OUTER JOIN或LEFT JOIN)   
右外连接(RIGHT OUTER JOIN或RIGHT JOIN)    
全外连接(FULL OUTER JOIN或FULL JOIN)    
（在表名后面加外连接操作符(*)或(+)指定非主体表）   

***连接查询（交叉连接）***    
其结果集合中的数据行数等于第一个表中符合查询条件的数据行数乘以第二个表中符合查询条件的数据行数    

### 嵌套查询
子查询的限制：不能使用order by 子句   
层层嵌套方式反应了SQL语言的结构化   
有些嵌套查询可以使用连接运算替代   
**嵌套查询的分类**   
不相关子查询：子查询条件不依赖与父查询   
相关子查询：子查询的查询条件依赖于父查询   

引出子查询的谓词： IN、比较运算符、带有Any或者ALL、Exists   

1. 查询选修了课程名为信息系统的学生学号和姓名。  
```sql
SELECT Sno， Sname
    FROM Student
    WHERE Sno IN
        (SELECT Sno
        FROM SC
        WHERE Cno IN
            (SELECT Cno
            FROM Course
            WHERE Cname= ‘信息系统’ ））；

```
可以使用连接查询来替换：（略）  
2. 查询其他系中比IS系任一学生年龄小的学生名单。    
```sql
SELECT Sname ,Sage
FROM Student
WHERE Sage<ANY
      (SELECT Sage
       FROM Student
       WHERE Sdept=‘IS’)
    AND Sdept <> ‘IS’
ORDER BY Sage DESC;
```
别忘了不等于条件** And sdept <> ‘IS’** 
3. 查询其他系中比IS所有学生年龄都小的学生名单    
使用集函数实现   
```sql
SELECT Sname ,Sage
FROM Student
WHERE Sage<
    ( SELECT MIN(sage)
        FROM Student
        WHERE Sdept =’IS’)
        AND Sdept <> ‘IS’
ORDER BY Sage DESC;
```
使用集函数实现子查询通常比直接使用ANY 或者 ALL 查询效率更高   

带有exists 谓词的子查询是相关子查询   
4. 查询所有选修了1号课程的学生姓名  
```sql
SELECT Sname from Student , SC WHERE student.Sno = SC.Sno And SC.Cno = '1'
```
使用相关子查询
```sql
select sname from student where exists (select * from sc where sno= student.sno and cno = '1')
```
代码中 from student 与student.sno 是相互依赖关系    

相关子查询的内层查询与外层查询有关， 因此必须反复求值。    
先关子查询的一般处理过程：   
首先取外层查询中Student表的第一个元组， 根据它与内层查询相关的属性值（即Sno值） 处理内层查询，若WHERE子句返回值为真（即内层查询结果非空），则取此元组放入结果表；然后再检查Student表的下一个元组；   
5. 查询所有未修1号课程的学生姓名   
```sql
SELECT Sname
FROM Student,SC
WHERE Student.Sno=SC.Sno AND Cno <>'1'
```
上述写法是错误的, 正确写法如下：  

```sql
SELECT Sname
FROM Student
WHERE NOT EXISTS
(SELECT *
FROM SC
WHERE Sno = Student.Sno AND Cno=‘1’);
```
另外写法：使用in 进行连接操作   
```sql
SELECT Sname
FROM Student
WHERE sno NOT IN
(SELECT Sno
FROM SC
WHERE Cno==’1’)
```
***注意使用exists 和 in 不相同的地方***   **exists是相关子查询，in是连接操作**
6. 列出选修了001号和002号课程的学生的学号
使用自身连接操作   
```sql
    SELECT sc1.Sno 
FROM SC sc1, SC sc2
WHERE sc1.Cno=‘001’ AND sc2.Cno=‘002’
AND sc1.Sno=sc2.Sno
```
使用相关子查询
```sql
select SNO
from SC SC1
where SC1.CNO = ‘001’
and exists
（select SNO
from SC SC2
where SC2. CNO =‘ 002
and SC2.SNO = SC1.SNO）
```
注意：相关子查询中from SC1 与 SC2.sno = sc1.sno 是相互依赖关系    
7. 找出选课的学生姓名和所在系，他们的1号课的成绩与“王小波”的1号课的成绩相等   
```sql
SELECT Sname,Sdept,Grade
FROM Stuedent,SC
WHERE Student.Sno=SC.Sno AND Cno=‘1’ AND
Grade IN
(SELECT Grade
FROM Stuedent,SC
WHERE Student.Sno=SC.Sno AND
AND Cno=‘1’ Sname=‘王小波’ )
```
### 集合查询 
多个SELECT语句的结果合并为一个结果， 可用集合操作来完成。
集合操作主要包括：  
并操作 Union   
交操作 Intersect
差操作 Minus（大部分可以转换为查询条件的or and not）   

系统会自动去掉结果重复的元组。   
需要注意的是，参加UNION操作的各数据项数目必须相同；对应项的数据类型也必须相同。   
查选选修课程1的学生集合与选修课程2的
学生集合的交集
```sql
SELECT Sno
FROM SC
WHERE Cno=‘1’ AND
Sno IN
(SELECT Sno
FROM SC
WHERE Cno=‘2’)
```


## 关于drop truncate delete 区别
1. delete语句执行删除的过程是每次从表中删除一行，并且把同时把该行的删除操作作为事务记录在日志中保存，以便进行回滚   
2. truncate 则一次性地从表中删除所有数据，通过释放存储表数据所用的数据页来删除数据，并且只在事务日志中记录页的释放。TRUNCATE TABLE 删除表中的所有行，但表结构及其列、约束、索引等保持不变。  
最直观的是 truncate 之后自增字段是从头开始计数，而delete仍保留着原来的最大数值   
3. drop 是将表占用的空间全部释放，包括数据和表的定义结构

TRUNCATE TABLE 在功能上与不带 WHERE 子句的 DELETE 语句相同：二者均删除表中的全部行。但 TRUNCATE TABLE 比 DELETE 速度快，且使用的系统和事务日志资源少。
对于外键（foreignkey ）约束引用的表，不能使用 truncate table，而应使用不带 where 子句的 delete 语句。


