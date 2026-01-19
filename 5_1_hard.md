## Задача 2*: Комплексный анализ эффективности производства

Разработайте запрос, который анализирует эффективность каждой мастерской, учитывая:
- Производительность каждого ремесленника (соотношение созданных продуктов к затраченному времени)
- Эффективность использования ресурсов (соотношение потребляемых ресурсов к производимым товарам)
- Качество производимых товаров (средневзвешенное по ценности)
- Время простоя мастерской
- Влияние навыков ремесленников на качество товаров

Возможный вариант выдачи:
```json
[
  {
    "workshop_id": 301,
    "workshop_name": "Royal Forge",
    "workshop_type": "Smithy",
    "num_craftsdwarves": 4,
    "total_quantity_produced": 256,
    "total_production_value": 187500,
    
    "daily_production_rate": 3.41,
    "value_per_material_unit": 7.82,
    "workshop_utilization_percent": 85.33,
    
    "material_conversion_ratio": 1.56,
    
    "average_craftsdwarf_skill": 7.25,
    
    "skill_quality_correlation": 0.83,
    
    "related_entities": {
      "craftsdwarf_ids": [101, 103, 108, 115],
      "product_ids": [801, 802, 803, 804, 805, 806],
      "material_ids": [201, 204, 208, 210],
      "project_ids": [701, 702, 703]
    }
  },
  {
    "workshop_id": 304,
    "workshop_name": "Gemcutter's Studio",
    "workshop_type": "Jewelcrafting",
    "num_craftsdwarves": 2,
    "total_quantity_produced": 128,
    "total_production_value": 205000,
    
    "daily_production_rate": 2.56,
    "value_per_material_unit": 13.67,
    "workshop_utilization_percent": 78.95,
    
    "material_conversion_ratio": 0.85,
    
    "average_craftsdwarf_skill": 8.50,
    
    "skill_quality_correlation": 0.92,
    
    "related_entities": {
      "craftsdwarf_ids": [105, 112],
      "product_ids": [820, 821, 822, 823, 824],
      "material_ids": [206, 213, 217, 220],
      "project_ids": [705, 708]
    }
  }
]
```

Решение:
```sql
with 
-- Получение текущих уровней навыков для каждого гнома
current_dwarf_skills as (
    select distinct on (dwarf_id, skill_id)
        dwarf_id, skill_id, level, experience
    from dwarf_skills
    order by dwarf_id, skill_id, date desc
),
-- Получение назначений гномов в цехи с информацией о навыках
workshop_assignments as (
    select 
        wc.workshop_id,
        wc.dwarf_id,
        wc.role,
        ds.name as dwarf_name,
        ds.profession,
        cds.level as skill_level,
        s.name as skill_name
    from workshop_craftsdwarves wc
    join dwarves ds on wc.dwarf_id = ds.dwarf_id
    join current_dwarf_skills cds on wc.dwarf_id = cds.dwarf_id
    join skills s on cds.skill_id = s.skill_id
),
-- Расчет метрик производства
production_metrics as (
    select 
        wp.workshop_id,
        count(distinct wp.product_id) as unique_products,
        sum(wp.quantity) as total_quantity,
        avg(p.quality) as avg_product_quality,
        sum(p.value * wp.quantity) as total_production_value,
        min(wp.production_date) as first_production_date,
        max(wp.production_date) as last_production_date,
        count(distinct wp.production_date) as production_days
    from workshop_products wp
    join products p on wp.product_id = p.product_id
    group by wp.workshop_id
),
-- Расчет использования материалов
material_usage as (
    select 
        wm.workshop_id,
        sum(wm.quantity) as total_material_quantity,
        sum(m.quality * wm.quantity) as total_material_quality_score,
        count(distinct wm.material_id) as unique_materials
    from workshop_materials wm
    join materials m on wm.material_id = m.material_id
    where wm.is_input = true
    group by wm.workshop_id
), 
-- Расчет скорости производства
production_rate as (
    select 
        pm.workshop_id,
        case 
            when pm.production_days > 0 
            then pm.total_quantity::decimal / pm.production_days
            else 0 
        end as daily_production_rate
    from production_metrics pm
),
-- Расчет коэффициента конверсии материалов
material_conversion as (
    select 
        pm.workshop_id,
        case 
            when mu.total_material_quality_score > 0 
            then pm.total_production_value::decimal / mu.total_material_quality_score
            else 0 
        end as material_conversion_ratio
    from production_metrics pm
    left join material_usage mu on pm.workshop_id = mu.workshop_id
),
-- Расчет стоимости на единицу материала
value_per_material as (
    select 
        pm.workshop_id,
        case 
            when mu.total_material_quantity > 0 
            then pm.total_production_value::decimal / mu.total_material_quantity
            else 0 
        end as value_per_material_unit
    from production_metrics pm
    left join material_usage mu on pm.workshop_id = mu.workshop_id
),
-- Расчет корреляции навыков и качества
skill_quality_correlation as (
    select 
        wa.workshop_id,
        case 
            when count(wa.dwarf_id) > 0 and count(pm.avg_product_quality) > 0 
            then corr(wa.skill_level, pm.avg_product_quality)
            else 0 
        end as skill_quality_correlation
    from workshop_assignments wa
    join production_metrics pm on wa.workshop_id = pm.workshop_id
    group by wa.workshop_id
),
-- Расчет использования цеха
workshop_utilization as (
    select 
        pm.workshop_id,
        case 
            when pm.first_production_date is not null 
            then pm.production_days::decimal / 
                 ((pm.last_production_date - pm.first_production_date) + 1) * 100
            else 0 
        end as workshop_utilization_percent
    from production_metrics pm
),
-- Расчет среднего уровня навыков
average_skill as (
    select 
        wa.workshop_id,
        avg(wa.skill_level) as average_craftsdwarf_skill
    from workshop_assignments wa
    group by wa.workshop_id
),
-- Получение деталей цехов
workshop_details as (
    select 
        w.workshop_id,
        w.name as workshop_name,
        w.type as workshop_type
    from workshops w
),
-- Все метрики для каждого цеха
workshop_analysis as (
    select 
        wd.workshop_id,
        wd.workshop_name,
        wd.workshop_type,
        count(distinct wa.dwarf_id) as num_craftsdwarves,
        coalesce(pm.total_quantity, 0) as total_quantity_produced,
        coalesce(pm.total_production_value, 0) as total_production_value,
        coalesce(pr.daily_production_rate, 0) as daily_production_rate,
        coalesce(vpm.value_per_material_unit, 0) as value_per_material_unit,
        coalesce(wu.workshop_utilization_percent, 0) as workshop_utilization_percent,
        coalesce(mc.material_conversion_ratio, 0) as material_conversion_ratio,
        coalesce(ascs.average_craftsdwarf_skill, 0) as average_craftsdwarf_skill,
        coalesce(sqc.skill_quality_correlation, 0) as skill_quality_correlation
    from workshop_details wd
    left join workshop_assignments wa on wd.workshop_id = wa.workshop_id
    left join production_metrics pm on wd.workshop_id = pm.workshop_id
    left join production_rate pr ON wd.workshop_id = pr.workshop_id
    left join material_conversion mc on wd.workshop_id = mc.workshop_id
    left join value_per_material vpm on wd.workshop_id = vpm.workshop_id
    left join workshop_utilization wu on wd.workshop_id = wu.workshop_id
    left join average_skill ascs on wd.workshop_id = ascs.workshop_id
    left join skill_quality_correlation sqc on wd.workshop_id = sqc.workshop_id
    group by 
        wd.workshop_id, wd.workshop_name, wd.workshop_type,
        pm.total_quantity, pm.total_production_value,
        pr.daily_production_rate, vpm.value_per_material_unit,
        wu.workshop_utilization_percent, mc.material_conversion_ratio,
        ascs.average_craftsdwarf_skill, sqc.skill_quality_correlation
),
-- Получение связанных сущностей
related_entities as (
    select 
        wa.workshop_id,
        array_agg(distinct wa.dwarf_id) as craftsdwarf_ids,
        array_agg(distinct wp.product_id) as product_ids,
        array_agg(distinct wm.material_id) as material_ids,
        array_agg(distinct pw.project_id) as project_ids
    from workshop_assignments wa
    left join workshop_products wp on wa.workshop_id = wp.workshop_id
    left join workshop_materials wm on wa.workshop_id = wm.workshop_id
    left join project_workers pw on wa.dwarf_id = pw.dwarf_id
    group by wa.workshop_id
)
-- Финальный запрос с JSON-выводом
select
    json_agg(
        json_build_object(
            'workshop_id', wa.workshop_id,
            'workshop_name', wa.workshop_name,
            'workshop_type', wa.workshop_type,
            'num_craftsdwarves', wa.num_craftsdwarves,
            'total_quantity_produced', wa.total_quantity_produced,
            'total_production_value', wa.total_production_value,
            'daily_production_rate', wa.daily_production_rate,
            'value_per_material_unit', wa.value_per_material_unit,
            'workshop_utilization_percent', wa.workshop_utilization_percent,
            'material_conversion_ratio', wa.material_conversion_ratio,
            'average_craftsdwarf_skill', wa.average_craftsdwarf_skill,
            'skill_quality_correlation', wa.skill_quality_correlation,
            'related_entities', json_build_object(
                'craftsdwarf_ids', re.craftsdwarf_ids,
                'product_ids', re.product_ids,
                'material_ids', re.material_ids,
                'project_ids', re.project_ids
            )
        )
        order by wa.workshop_id
    )
from workshop_analysis wa
left join related_entities re on wa.workshop_id = re.workshop_id;
```