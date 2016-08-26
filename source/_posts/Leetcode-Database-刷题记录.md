title: Leetcode - Database 刷题记录
date: 2016-08-26 15:50:59
categories: 闲言碎语
author: Sergio Chan


tags: [MySQL, Leetcode]
---



偶然被 Cee 酱吸引打开了 Leetcode，这个大一的时候觉得自己一辈子都不会去做的事情……居然。发现 OJ 竟然有 Database 模块了，想当年大学学数据库的时候最想有的就是一个和 OJ 一样的 SQL 在线运行平台，不然数据库上机只会教你安装 SQL Server = = 想想都想哭，时间都浪费在那了。而且做了之后发现好多自己在生产环境都不会遇到的神奇的坑……因此抽着空闲做了大部分，留下此篇笔记。

浮生若梦。

## List

- [175 . Combine Two Tables](#175_-_Combine_Two_Tables)
- [177 . Nth Highest Salary](#177_-_Nth_Highest_Salary)
- [181 . Employees Earning More Than Their Managers](#181_-_Employees_Earning_More_Than_Their_Managers)
- [182 . Duplicate Emails](#182_-_Duplicate_Emails)
- [183 . Customers Who Never Order](#183_-_Customers_Who_Never_Order)
- [185 . Department Top Three Salaries](#185_-_Department_Top_Three_Salaries)
- [196 . Delete Duplicate Emails](#196_-_Delete_Duplicate_Emails)
- [197 . Rising Temperature](#197_-_Rising_Temperature)

### 175 . Combine Two Tables

Table: `Person`

```
+-------------+---------+
| Column Name | Type    |
+-------------+---------+
| PersonId    | int     |
| FirstName   | varchar |
| LastName    | varchar |
+-------------+---------+
PersonId is the primary key column for this table.
```

Table: `Address`

```
+-------------+---------+
| Column Name | Type    |
+-------------+---------+
| AddressId   | int     |
| PersonId    | int     |
| City        | varchar |
| State       | varchar |
+-------------+---------+
AddressId is the primary key column for this table.

```

Write a SQL query for a report that provides the following information for each person in the Person table, regardless if there is an address for each of those people:

```
FirstName, LastName, City, State
```

#### SQL :

```
SELECT Person.FirstName AS FirstName ,Person.LastName AS LastName ,Address.City AS City,Address.State AS State
FROM Person
LEFT JOIN Address ON (Person.PersonId = Address.PersonId)
```

#### Comment :

其实这题看起来很简单，但是要注意一个是用 **LEFT JOIN**，这样即使没有地址的 Person 也会被选出来，另一个是在 Select 的时候要按照它的要求命名选出来的列（由于这题老是遇到 Internal Error，相同的语句运行三遍通过一遍有两遍都是 Internal Error，因此并不是特别明白这个是不是必须的）但是 **LEFT JOIN** 是一定要有的，用 where 连接两个表的话也要特别注意到不存在的情况，因此本题还是用 JOIN 做连接好过用 where 做多表连接

### 177 . Nth Highest Salary

Write a SQL query to get the *n*th highest salary from the `Employee` table.

```
+----+--------+
| Id | Salary |
+----+--------+
| 1  | 100    |
| 2  | 200    |
| 3  | 300    |
+----+--------+

```

For example, given the above Employee table, the *n*th highest salary where *n* = 2 is `200`. If there is no *n*th highest salary, then the query should return `null`.

#### SQL :

```
CREATE FUNCTION getNthHighestSalary(N INT) RETURNS INT
BEGIN
  DECLARE l INT;
  DECLARE r INT;
  DECLARE c INT;
  
  SET l = N-1;
  SELECT COUNT(DISTINCT(Salary)) INTO c FROM Employee;
  
  IF N > c THEN 
    RETURN (null);
  ELSE
    SELECT Salary INTO r FROM Employee GROUP BY Salary ORDER BY Salary DESC LIMIT l, 1;
    RETURN (r);
  END IF;
END
```

#### Comment :

这题简直变态。

![](https://ooo.0o0.ooo/2016/08/26/57bfebf246fe0.png)

有这时间不如用 Python 什么的取出想要的数据在内存里排个序就完事了，非得整个 SQL Function，简直不知道说什么好。之前也写过 Function，但是主要是在 SQL 的 Procedure 编程里面会用到，而且基本不会写到复杂的逻辑 = = 复杂逻辑写个脚本什么的不好么……言归正传，这题考察了 Function 中的多个基本语法，包括 **DECLARE**, **SET**, **SELECT 的结果赋值**，**IF...THEN...ELSE...END IF 逻辑**。（这里其实题目没交代清楚，懒得吐槽了）如果遇到相同 Salary 的数据怎么办，其实题目的意思是如果两个人的 Salary 都为 200，一个人 Salary 为 300，那就只有第一 300 第二 200 而没有第三，他最后需要你返回的也只是 Salary 的值，因此我们需要先去重一遍得到 DISTINCT 的 Salary 值的数量，如果要获取的第 N 位并不存在，则返回 null，如果存在则用 **GROUP BY 套 ORDER BY 套 LIMIT** 来获取分组排序后的第 N - 1 位，记住要获取的第 N 位的 index 是 N - 1，所以这儿才要重新 DECLARE 一个 l 。

###  181 . Employees Earning More Than Their Managers

The `Employee` table holds all employees including their managers. Every employee has an Id, and there is also a column for the manager Id.

```
+----+-------+--------+-----------+
| Id | Name  | Salary | ManagerId |
+----+-------+--------+-----------+
| 1  | Joe   | 70000  | 3         |
| 2  | Henry | 80000  | 4         |
| 3  | Sam   | 60000  | NULL      |
| 4  | Max   | 90000  | NULL      |
+----+-------+--------+-----------+

```

Given the `Employee` table, write a SQL query that finds out employees who earn more than their managers. For the above table, Joe is the only employee who earns more than his manager.

```
+----------+
| Employee |
+----------+
| Joe      |
+----------+

```

#### SQL :

```
SELECT a.Name as Employee 
FROM Employee as a 
WHERE a.ManagerId is not NULL AND a.Salary > 
(SELECT b.Salary 
FROM Employee as b 
WHERE b.Id = a.ManagerId);
```

#### Comment :

这题简单，不多解释。有 JOIN 的解法应该也行。

###  182 . Duplicate Emails

Write a SQL query to find all duplicate emails in a table named `Person`.

```
+----+---------+
| Id | Email   |
+----+---------+
| 1  | a@b.com |
| 2  | c@d.com |
| 3  | a@b.com |
+----+---------+

```

For example, your query should return the following for the above table:

```
+---------+
| Email   |
+---------+
| a@b.com |
+---------+
```

#### SQL :

```
SELECT Person.Email 
FROM Person 
GROUP BY Person.Email 
HAVING COUNT(Person.Id) > 1;
```

#### Comment :

这题简单的不能再简单了，然而还是有一个蛋疼的坑：**COUNT 的用法**。这题如果你在 having COUNT() 括号中写 * 号，是不能通过的，原因就在于 **COUNT(*)** 统计的是表中数据的总条数，而 **COUNT(Person.Id)** 统计的则是除去表中 Id 不等于 NULL 的记录的总条数。因此很可能它有某个 testcase 中有 NULL 的数据来考察对于 COUNT 的理解吧。

### 183 . Customers Who Never Order

Suppose that a website contains two tables, the `Customers` table and the `Orders` table. Write a SQL query to find all customers who never order anything.

Table: `Customers`.

```
+----+-------+
| Id | Name  |
+----+-------+
| 1  | Joe   |
| 2  | Henry |
| 3  | Sam   |
| 4  | Max   |
+----+-------+

```

Table: `Orders`.

```
+----+------------+
| Id | CustomerId |
+----+------------+
| 1  | 3          |
| 2  | 1          |
+----+------------+

```

Using the above tables as example, return the following:

```
+-----------+
| Customers |
+-----------+
| Henry     |
| Max       |
+-----------+
```

#### SQL :

```
SELECT a.Name AS Customers 
FROM Customers AS a 
LEFT JOIN Orders AS b 
ON (a.Id = b.CustomerId) 
WHERE a.Id IS NULL OR b.CustomerId IS NULL;
```

#### Comment :

这题如果不用 Left Join 而用 Not in 的话也是可以的，只是效率方面 Not in 的话运行时间是排在百分之八十多，如果用了上面的 Left Join 则会好一些 （ runtime 每次运行会有微妙的差距，可能几十毫秒就是百分之几十的差距）。

![](https://ooo.0o0.ooo/2016/08/26/57bfeb47b9186.png)

###  185 . Department Top Three Salaries

The `Employee` table holds all employees. Every employee has an Id, and there is also a column for the department Id.

```
+----+-------+--------+--------------+
| Id | Name  | Salary | DepartmentId |
+----+-------+--------+--------------+
| 1  | Joe   | 70000  | 1            |
| 2  | Henry | 80000  | 2            |
| 3  | Sam   | 60000  | 2            |
| 4  | Max   | 90000  | 1            |
| 5  | Janet | 69000  | 1            |
| 6  | Randy | 85000  | 1            |
+----+-------+--------+--------------+

```

The `Department` table holds all departments of the company.

```
+----+----------+
| Id | Name     |
+----+----------+
| 1  | IT       |
| 2  | Sales    |
+----+----------+
```

Write a SQL query to find employees who earn the top three salaries in each of the department. For the above tables, your SQL query should return the following rows.

```
+------------+----------+--------+
| Department | Employee | Salary |
+------------+----------+--------+
| IT         | Max      | 90000  |
| IT         | Randy    | 85000  |
| IT         | Joe      | 70000  |
| Sales      | Henry    | 80000  |
| Sales      | Sam      | 60000  |
+------------+----------+--------+
```

#### SQL :

```
select d.Name as Department, a.Name as Employee, a.Salary as Salary 
from Employee as a,Department as d 
where (select count(b.Salary) 
from (select e.Salary as Salary,e.DepartmentId as DepartmentId 
from Employee as e 
group by e.Salary,e.DepartmentId) as b 
where b.Salary > a.Salary and b.DepartmentId = a.DepartmentId) <= 2 and d.Id = a.DepartmentId 
order by a.DepartmentId,a.Salary desc;
```

#### Comment :

变态。这道题无愧是 Hard。上面这个解法虽然套了三层 Select ，但是它的运行时间都能排在前 15%

![](https://ooo.0o0.ooo/2016/08/26/57bfea0296548.jpeg)

可见这题本来就是计算量很大的一次查询。主要考察了 **group by  的用法**，group by 的用法其实比较特殊，比如 Salary 相同但其他列不同的情况下，如果 group by Salary，那就只能 select salary，否则 salary 相同的项目会无法合并。（当然在你自己的 MySQL 环境下你可以配一个 full group mode 参数来强制允许 MySQL 能够执行这样的语句）因此在最中间我们先是将用户分组排序过后的数据分进一个单独的表 b ，这里的数据由于 group by Salary 和 DepartmentId，因此同部门相同 Salary 的数据已经被合并到一项去了，那么接着就确认当前同部门的工资比自己高的等级是否小于 3 个，如果是的话就取出来。

*PS . 这句 SQL 主要就是长长长，其实也没什么 = =*

###  196 . Delete Duplicate Emails

Write a SQL query to delete all duplicate email entries in a table named `Person`, keeping only unique emails based on its *smallest* **Id**.

```
+----+------------------+
| Id | Email            |
+----+------------------+
| 1  | john@example.com |
| 2  | bob@example.com  |
| 3  | john@example.com |
+----+------------------+
Id is the primary key column for this table.

```

For example, after running your query, the above `Person` table should have the following rows:

```
+----+------------------+
| Id | Email            |
+----+------------------+
| 1  | john@example.com |
| 2  | bob@example.com  |
+----+------------------+
```

#### SQL :

```
delete from Person where Id in (
select * from ( 
select b.Id
from (select * from Person) as b 
where exists (
select d.* 
from (select * from Person) as d 
where d.Id < b.Id and b.Email = d.Email)) as test);
```

#### Comment :

这题考察的是 **delete + where + in** 的套路。delete where in 后面跟着的 select 里面的表不能是 delete 的 target，这个规则 delete 和 update 同理，要在 delete where in 后面加上 target 表的查询，你就需要套一层 select * … as … 就可以了。效率略低，但是实用。

###  197 . Rising Temperature

Given a `Weather` table, write a SQL query to find all dates' Ids with higher temperature compared to its previous (yesterday's) dates.

```
+---------+------------+------------------+
| Id(INT) | Date(DATE) | Temperature(INT) |
+---------+------------+------------------+
|       1 | 2015-01-01 |               10 |
|       2 | 2015-01-02 |               25 |
|       3 | 2015-01-03 |               20 |
|       4 | 2015-01-04 |               30 |
+---------+------------+------------------+

```

For example, return the following Ids for the above Weather table:

```
+----+
| Id |
+----+
|  2 |
|  4 |
+----+
```

#### SQL :

```
SELECT a.Id 
FROM Weather AS a 
WHERE EXISTS (
SELECT b.Temperature 
FROM Weather AS b 
WHERE b.Date = SUBDATE(a.Date,1) and b.Temperature < a.Temperature);
```

#### Comment :

此题又是看起来简单的不能再简单了。然而坑就在 Date 类型的数据操作上，第一，几乎少有人在实际生产环境中用 Date 这种不靠谱的数据格式，我还没见过谁生产环境不用时间戳的 = =，第二，这个坑谁知道啊…… Date 提供了加减的操作，看起来你好像可以直接 update 一个 date + 1 等于第二天的日期，然而碰到月份的边界这个加减就挂了啊（黑人问号.jpg） 碰到 01-01，减 1 之后等于 01-00，因此其实 SQL 是有日期操作**函数**的， SUBDATE(DATE, INTERVAL) 就可以实现。