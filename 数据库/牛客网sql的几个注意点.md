[toc]
## group by 和 max 的结合 ##
- 统计各个部门最高薪水的员工，联结查询分组之后使用max函数。
- [原题](https://www.nowcoder.com/practice/4a052e3e1df5435880d4353eb18a91c6?tpId=82&tqId=29764&rp=0&ru=%2Fta%2Fsql&qru=%2Fta%2Fsql%2Fquestion-ranking&tPage=1)
```sql
select de.dept_no, de.emp_no, max(s.salary) as salary
from dept_emp de inner join salaries s
on de.emp_no  = s.emp_no
where de.to_date='9999-01-01' and s.to_date='9999-01-01'
group by de.dept_no;
```

## 自联结应用 ##
- 从第一张获取入职时候的薪水，从第二张获取当前薪水，从而得到增长。
- [原题](https://www.nowcoder.com/practice/fc7344ece7294b9e98401826b94c6ea5?tpId=82&tqId=29773&rp=0&ru=%2Fta%2Fsql&qru=%2Fta%2Fsql%2Fquestion-ranking&tPage=2)
```sql
select e.emp_no, (s2.salary - s1.salary) as growth
from employees e 
inner join salaries s1
on e.emp_no = s1.emp_no and s1.from_date = e.hire_date
inner join salaries s2
on e.emp_no = s2.emp_no and s2.to_date = '9999-01-01'
order by growth;
```

- 对薪水进行排序，第二张表用来统计薪水大于等于待排序薪水的记录的数量。
- [原题](https://www.nowcoder.com/practice/b9068bfe5df74276bd015b9729eec4bf?tpId=82&tqId=29775&rp=0&ru=/ta/sql&qru=/ta/sql/question-ranking)
```sql
select s1.emp_no, s1.salary, (count(distinct s2.salary)) as rank
from salaries s1 inner join salaries s2
where s1.to_date='9999-01-01' and s2.to_date='9999-01-01' and s1.salary <= s2.salary
group by s1.emp_no
order by s1.salary desc;
```

- 第一张表示初始薪水，第二张表示结束的薪水。
    - 一段时期的开始时间和另一段时期的结束时间在年份上是相同的。
- [原题](https://www.nowcoder.com/practice/eb9b13e5257744db8265aa73de04fd44?tpId=82&tqId=29779&tPage=2&rp=&ru=/ta/sql&qru=/ta/sql/question-ranking)
```sql
select s2.emp_no, s2.from_date, (s2.salary - s1.salary) as salary_growth
from salaries s1 inner join salaries s2
on s1.emp_no = s2.emp_no
and salary_growth > 5000
and (
    strftime('%Y', s2.to_date) - (strftime('%Y', s1.to_date)) = 1
    or strftime('%Y', s2.from_date) - (strftime('%Y', s1.from_date)) = 1
     )
order by salary_growth desc;
```

- 以第一张表的emp_no分组，在第二张表中通过联结条件计算较小的所有emp_no对应的薪水的和。
- [原题](https://www.nowcoder.com/practice/58824cd644ea47d7b2b670c506a159a6?tpId=82&tqId=29828&rp=0&ru=/ta/sql&qru=/ta/sql/question-ranking)
```sql
select s1.emp_no, s1.salary, sum(s2.salary) as running_total
from salaries s1 inner join salaries s2
on s2.emp_no <= s1.emp_no and s2.to_date = '9999-01-01' and s1.to_date = '9999-01-01'
group by s1.emp_no;
```

## 从查询的结果中查询 ##
- 查找普通员工当前的薪水和经理当前的薪水，并从这个结果中查找比经理薪水高的员工相关信息。
- [原题](https://www.nowcoder.com/practice/f858d74a030e48da8e0f69e21be63bef?tpId=82&tqId=29777&tPage=2&rp=&ru=/ta/sql&qru=/ta/sql/question-ranking)
```sql
select es.emp_no, ms.emp_no as manager_no, es.salary as emp_salary, ms.salary as manager_salary
from 
(select s.emp_no, de.dept_no, s.salary 
 from salaries s inner join dept_emp de
 on s.emp_no = de.emp_no and s.to_date='9999-01-01') as es,
(select s.emp_no, dm.dept_no, s.salary
 from salaries s inner join dept_manager dm
 on s.emp_no = dm.emp_no and s.to_date='9999-01-01') as ms
where es.dept_no = ms.dept_no and es.salary > ms.salary;
```

## 删除重复记录 ##
- 先查找需要保留的id，再将不在范围内的记录删除。
- [原题](https://www.nowcoder.com/practice/3d92551a6f6d4f1ebde272d20872cf05?tpId=82&tqId=29810&rp=0&ru=/ta/sql&qru=/ta/sql/question-ranking)
```sql
delete from titles_test 
where id not in (select min(id) from titles_test group by emp_no);
```