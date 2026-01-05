## 1. Получить информацию о всех гномах, которые входят в какой-либо отряд, вместе с информацией об их отрядах.

```sql
select *
from Dwarves d
join Squads s
    on s.squad_id == d.squad_id;
```

## 2. Найти всех гномов с профессией "miner", которые не состоят ни в одном отряде.

```sql
select *
from Dwarves d
where 
    d.profession == 'miner'
    and d.squad_id is Null;
```

## 3. Получить все задачи с наивысшим приоритетом, которые находятся в статусе "pending".

```sql
select *
from Tasks t
where 
    t.priority == 0
    and t.status == 'pending';
```

## 4. Для каждого гнома, который владеет хотя бы одним предметом, получить количество предметов, которыми он владеет.

```sql
select
    d.dwarf_id,
    d.name,
    d.profession,
    COUNT(i.item_id) as item_count
from Dwarves d
join Items i 
    on d.dwarf_id = i.owner_id
group by d.dwarf_id, d.name, d.profession
order by item_count desc;
```

## 5. Получить список всех отрядов и количество гномов в каждом отряде. Также включите в выдачу отряды без гномов.

```sql
SELECT
    s.squad_id,
    s.name as squad_name,
    s.mission,
    COUNT(d.dwarf_id) as dwarf_count
from Squads s
left join  Dwarves d 
    on s.squad_id = d.squad_id
group by s.squad_id, s.name, s.mission
order by dwarf_count desc;
```

## 6. Получить список профессий с наибольшим количеством незавершённых задач ("pending" и "in_progress") у гномов этих профессий.

```sql
select
    d.profession,
    count(t.task_id) as unfinished_task_count
from Dwarves d
join Tasks t 
    on d.dwarf_id = t.assigned_to
where 
    t.status in ('pending', 'in_progress')
group by d.profession
order by unfinished_task_count desc;
```

## 7. Для каждого типа предметов узнать средний возраст гномов, владеющих этими предметами.

```sql
select
    i.type as item_type,
    round(avg(d.age), 2) as average_age
from Items i
join Dwarves d 
    on i.owner_id = d.dwarf_id
group by i.type
order by average_age desc;
```

## 8. Найти всех гномов старше среднего возраста (по всем гномам в базе), которые не владеют никакими предметами.

```sql
select
    d.dwarf_id,
    d.name,
    d.age,
    (select avg(age) from Dwarves) as average_age
from Dwarves d
left join Squads s 
    on d.squad_id = s.squad_id
where
    d.age > (
        select avg(age) 
        from Dwarves
    )  -- средний возраст гномов
    and d.dwarf_id not in (
        select distinct owner_id 
        from Items 
        where owner_id is not null
    )  -- гномы, которые не владеют никакими предметами
order by d.age desc;
```