---
layout: post
title:  Mysql having用法 
date:   2018-12-26 22:02:00 +0800
categories: 技术
tag: Mysql
---


顺序：where -> group by -> min -> order by -> limit
在select语句中使用having 子句来指定一组行或聚合的过滤条件

having 子句通常与 group by子句一起使用，以根据指定的条件过滤分组。如果省略group by子句，则having 子句的行为与where 子句类似

 

MySQL HAVING子句示例
---
1、查询重复的行

select id,name from student where name in (select name from student group by name having (count(*) > 1)) order by name;

查询student表中重名的学生，结果包含id和name，按name,id升序

2、查询分组中特定要求的行

select sid,avg(score) as avg_score from student_course group by sid having (avg_score < 60);

在student_course表中查询平均分不及格的学生，列出学生id和平均分

3、显示每个地区的总人口数和总面积．仅显示那些面积超过1000000的地区。

select region,sum(population),sum(area) from china group by region having (sum(area) > 1000000);

 

HAVING与WHERE区别
---
having字句可以让我们筛选成组后的各种数据，where字句在聚合前先筛选记录，也就是说作用在group by和having字句前。而 having子句在聚合后对组记录进行筛选
