## Задача 2: Получение данных о гноме с навыками и назначениями
Создайте запрос, который возвращает информацию о гноме, включая идентификаторы всех его навыков, текущих назначений, принадлежности к отрядам и используемого снаряжения.

```sql
select
    d.dwarf_id,
    d.name,
    d.age,
    d.profession,
    json_build_object(
        'skill_ids', coalesce((
            select json_agg(ds.skill_id)
            from dwarf_skills ds
            where ds.dwarf_id = d.dwarf_id
        ), '[]'::json),
        'assignment_ids', coalesce((
            select json_agg(wc.workshop_id)
            from workshop_craftsdwarves wc
            where wc.dwarf_id = d.dwarf_id
        ), '[]'::json),
        'squad_ids', coalesce((
            select json_agg(sm.squad_id)
            from squad_members sm
            where sm.dwarf_id = d.dwarf_id
            and (sm.exit_date is null or sm.exit_date > current_date)
        ), '[]'::json),
        'equipment_ids', coalesce((
            select json_agg(de.equipment_id)
            from dwarf_equipment de
            where de.dwarf_id = d.dwarf_id
        ), '[]'::json)
    ) as related_entities
from dwarves d
where 1=1
    and dwarf_id = 2;
```

## Задача 3: Данные о мастерской с назначенными рабочими и проектами
Напишите запрос для получения информации о мастерской, включая идентификаторы назначенных ремесленников, текущих проектов, используемых и производимых ресурсов.

```sql
select
    w.workshop_id,
    w.name,
    w.type,
    w.quality,
    json_build_object(
        'craftsdwarf_ids', coalesce((
            select json_agg(wc.dwarf_id)
            from workshop_craftsdwarves wc
            where wc.workshop_id = w.workshop_id
        ), '[]'::json),
        'project_ids', coalesce((
            select json_agg(p.project_id)
            from projects p
            where p.workshop_id = w.workshop_id
            and (p.end_date is null or p.end_date > current_date)
        ), '[]'::json),
        'input_material_ids', coalesce((
            select json_agg(wm.material_id)
            from workshop_materials wm
            where wm.workshop_id = w.workshop_id
            and wm.is_input = true
        ), '[]'::json),
        'output_product_ids', coalesce((
            select json_agg(wp.product_id)
            from workshop_products wp
            where wp.workshop_id = w.workshop_id
        ), '[]'::json)
    ) as related_entities
from workshops w
where 1=1
    and workshop_id = 1;
```

## Задача 4: Данные о военном отряде с составом и операциями

Разработайте запрос, который возвращает информацию о военном отряде, включая идентификаторы всех членов отряда, используемого снаряжения, прошлых и текущих операций, тренировок.

```sql
select
    ms.squad_id,
    ms.name,
    ms.formation_type,
    ms.leader_id,
    json_build_object(
        'member_ids', coalesce((
            select json_agg(sm.dwarf_id)
            from squad_members sm
            where sm.squad_id = ms.squad_id
            and (sm.exit_date is null or sm.exit_date > current_date)
        ), '[]'::json),
        'equipment_ids', coalesce((
            select json_agg(se.equipment_id)
            from squad_equipment se
            where se.squad_id = ms.squad_id
        ), '[]'::json),
        'operation_ids', coalesce((
            select json_agg(so.operation_id)
            from squad_operations so
            where so.squad_id = ms.squad_id
        ), '[]'::json),
        'training_schedule_ids', coalesce((
            select json_agg(st.training_id)
            from squad_training st
            where st.squad_id = ms.squad_id
        ), '[]'::json),
        'battle_report_ids', coalesce((
            select json_agg(sb.battle_id)
            from squad_battles sb
            where sb.squad_id = ms.squad_id
        ), '[]'::json)
    ) as related_entities
from
    military_squads ms;
```