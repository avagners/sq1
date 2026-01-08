# 1. Найдите все отряды, у которых нет лидера.

```sql
select 
    squad_id,
    name
from Squads
where 1=1
    and leader_id is NULL;
```

# 2. Получите список всех гномов старше 150 лет, у которых профессия "Warrior".

```sql
select 
    dwarf_id,
    name,
    age,
    profession
from Dwarves
where 1=1
    and age > 150
    and profession = 'Warrior';
```

# 3. Найдите гномов, у которых есть хотя бы один предмет типа "weapon".

```sql
select distinct
    d.dwarf_id,
    d.name,
    d.profession
from
    Dwarves d
join Items i 
    on d.dwarf_id = i.owner_id
where
    i.type = 'weapon';
```

# 4. Получите количество задач для каждого гнома, сгруппировав их по статусу.

```sql
select
    d.dwarf_id,
    d.name,
    t.status,
    count(t.task_id) as task_count
from Dwarves d
left join Tasks t 
    on d.dwarf_id = t.assigned_to
group by d.dwarf_id, d.name, t.status
```

# 5. Найдите все задачи, которые были назначены гномам из отряда с именем "Guardians".

```sql
select
    t.task_id,
    t.description,
    t.status,
    d.name as assigned_to
from Tasks t
join Dwarves d 
    on t.assigned_to = d.dwarf_id
join Squads s 
    on d.squad_id = s.squad_id
where 1=1
    and s.name = 'Guardians'
```

# 6. Выведите всех гномов и их ближайших родственников, указав тип родственных отношений.

```sql
select
    d1.name as dwarf_name,
    d1.profession,
    r.relationship,
    d2.name as relative_name,
    d2.profession as relative_profession
from Dwarves d1
join Relationships r 
    on d1.dwarf_id = r.dwarf_id
join Dwarves d2 
    on r.related_to = d2.dwarf_id;    
```
