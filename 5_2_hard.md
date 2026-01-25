## Задача 3*: Комплексная оценка военной эффективности отрядов
Создайте запрос, оценивающий эффективность военных отрядов на основе:
- Результатов всех сражений (победы/поражения/потери)
- Соотношения побед к общему числу сражений
- Навыков членов отряда и их прогресса
- Качества экипировки
- Истории тренировок и их влияния на результаты
- Выживаемости членов отряда в долгосрочной перспективе

Возможный вариант выдачи:

```json
[
  {
    "squad_id": 401,
    "squad_name": "The Axe Lords",
    "formation_type": "Melee",
    "leader_name": "Urist McAxelord",
    "total_battles": 28,
    "victories": 22,
    "victory_percentage": 78.57,
    "casualty_rate": 24.32,
    "casualty_exchange_ratio": 3.75,
    "current_members": 8,
    "total_members_ever": 12,
    "retention_rate": 66.67,
    "avg_equipment_quality": 4.28,
    "total_training_sessions": 156,
    "avg_training_effectiveness": 0.82,
    "training_battle_correlation": 0.76,
    "avg_combat_skill_improvement": 3.85,
    "overall_effectiveness_score": 0.815,
    "related_entities": {
      "member_ids": [102, 104, 105, 107, 110, 115, 118, 122],
      "equipment_ids": [5001, 5002, 5003, 5004, 5005, 5006, 5007, 5008, 5009],
      "battle_report_ids": [1101, 1102, 1103, 1104, 1105, 1106, 1107, 1108],
      "training_ids": [901, 902, 903, 904, 905, 906]
    }
  },
  {
    "squad_id": 403,
    "squad_name": "Crossbow Legends",
    "formation_type": "Ranged",
    "leader_name": "Dokath Targetmaster",
    "total_battles": 22,
    "victories": 18,
    "victory_percentage": 81.82,
    "casualty_rate": 16.67,
    "casualty_exchange_ratio": 5.20,
    "current_members": 7,
    "total_members_ever": 10,
    "retention_rate": 70.00,
    "avg_equipment_quality": 4.45,
    "total_training_sessions": 132,
    "avg_training_effectiveness": 0.88,
    "training_battle_correlation": 0.82,
    "avg_combat_skill_improvement": 4.12,
    "overall_effectiveness_score": 0.848,
    "related_entities": {
      "member_ids": [106, 109, 114, 116, 119, 123, 125],
      "equipment_ids": [5020, 5021, 5022, 5023, 5024, 5025, 5026],
      "battle_report_ids": [1120, 1121, 1122, 1123, 1124, 1125],
      "training_ids": [920, 921, 922, 923, 924]
    }
  }
]
```

Решение:
```sql
with squad_battle_stats as (
    -- расчет статистики сражений для каждого отряда
    select
        s.squad_id,
        s.name as squad_name,
        s.formation_type,
        d.name as leader_name,
        count(distinct sb.report_id) as total_battles,
        sum(case when sb.outcome = 'victory' then 1 else 0 end) as victories,
        sum(sb.casualties) as total_casualties,
        sum(sb.enemy_casualties) as total_enemy_casualties
    from military_squads s
    left join squad_battles sb 
        on s.squad_id = sb.squad_id
    left join dwarves d 
        on s.leader_id = d.dwarf_id
    group by s.squad_id, s.name, s.formation_type, d.name
),
squad_member_stats as (
    -- расчет статистики членов отряда
    select
        sm.squad_id,
        count(distinct case when sm.exit_date is null then sm.dwarf_id else null end) as current_members,
        count(distinct sm.dwarf_id) as total_members_ever,
        sum(case when sm.exit_reason = 'death' then 1 else 0 end) as deaths,
        -- средняя продолжительность службы в днях
        avg(case when sm.exit_date is not null then (sm.exit_date - sm.join_date) else (current_date - sm.join_date) end) as avg_service_duration_days,
        -- процент выживших за последние 6 месяцев
        sum(case when sm.exit_date is null or sm.exit_date >= current_date - interval '6 months' then 1 else 0 end) as members_since_6_months,
        sum(case when sm.exit_reason = 'death' and sm.exit_date >= current_date - interval '6 months' then 1 else 0 end) as deaths_last_6_months
    from squad_members sm
    group by sm.squad_id
),
squad_equipment_stats as (
    -- расчет статистики качества экипировки для каждого отряда
    select
        se.squad_id,
        avg(e.quality) as avg_equipment_quality,
        count(distinct se.equipment_id) as equipment_count
    from squad_equipment se
    join equipment e 
        on se.equipment_id = e.equipment_id
    group by se.squad_id
),
squad_training_stats as (
    -- расчет статистики тренировок для каждого отряда
    select
        st.squad_id,
        count(distinct st.schedule_id) as total_training_sessions,
        avg(st.effectiveness) as avg_training_effectiveness,
        count(distinct case when st.date >= current_date - interval '3 months' then st.schedule_id else null end) as recent_training_sessions,
        avg(case when st.date >= current_date - interval '3 months' then st.effectiveness else null end) as recent_training_effectiveness
    from squad_training st
    group by st.squad_id
),
squad_skill_improvement as (
    -- расчет улучшения навыков членов отряда
    select
        sm.squad_id,
        avg(ds.level) as avg_combat_skill_level,
        avg(ds.experience) as avg_combat_skill_experience,
        -- расчет прогресса навыков
        avg(ds.level) as avg_skill_level_for_progress,
        -- средний опыт
        avg(ds.experience) as avg_experience_for_progress,
        -- процент членов отряда с военными навыками
        count(distinct sm.dwarf_id) * 100.0 / nullif(
            (select count(distinct dwarf_id) 
             from squad_members sm2 
             where sm2.squad_id = sm.squad_id
            ), 0
        ) as military_skill_coverage_percentage
    from squad_members sm
    join dwarf_skills ds 
        on sm.dwarf_id = ds.dwarf_id
    join skills s 
        on ds.skill_id = s.skill_id
    group by sm.squad_id
)
-- финальный расчет комплексного показателя эффективности
select
    sbs.squad_id,
    sbs.squad_name,
    sbs.formation_type,
    sbs.leader_name,
    sbs.total_battles,
    sbs.victories,
    case when sbs.total_battles > 0 
        then round(((sbs.victories::float / sbs.total_battles) * 100)::numeric, 2) 
        else 0 
    end as victory_percentage,
    -- расчет casualty_rate (средние потери на битву)
    case 
        when sbs.total_battles > 0 
        then round(((sbs.total_casualties::float / sbs.total_battles))::numeric, 2) 
        else 0 
    end as casualty_rate,
    -- расчет casualty_exchange_ratio (соотношение потерь противника к "нашим")
    case 
        when sbs.total_casualties > 0 
        then round((sbs.total_enemy_casualties::float / sbs.total_casualties)::numeric, 2) 
        else 0 
    end as casualty_exchange_ratio,
    sms.current_members,
    sms.total_members_ever,
    case 
        when sms.total_members_ever > 0 
        then round(((sms.current_members::float / sms.total_members_ever) * 100)::numeric, 2) 
        else 0 
    end as retention_rate,
    ses.avg_equipment_quality,
    ses.equipment_count,
    sts.total_training_sessions,
    round(sts.avg_training_effectiveness::numeric, 2) as avg_training_effectiveness,
    -- расчет корреляции между тренировками и боевыми результатами
    case when sbs.total_battles > 0 and sts.total_training_sessions > 0
         then round(((sbs.victories::float / sbs.total_battles) * (sts.avg_training_effectiveness / 100))::numeric, 2)
         else 0 
    end as training_battle_correlation,
    ssi.avg_combat_skill_level,
    ssi.avg_combat_skill_experience,
    -- расчет взвешенное среднее эффективности
    round(
        (
            -- боевая эффективность (40% вес)
            coalesce(
                case 
                    when sbs.total_battles > 0 
                    then (sbs.victories::float / sbs.total_battles) * 0.4 
                    else 0 
                end, 0) +
            -- показатель выживаемости (20% вес)
            coalesce(
                case 
                    when sms.total_members_ever > 0 
                    then (sms.current_members::float / sms.total_members_ever) * 0.2 
                    else 0 
                end, 0) +
            -- качество экипировки (15% вес)
            coalesce(ses.avg_equipment_quality / 10 * 0.15, 0) +
            -- эффективность тренировок (15% вес)
            coalesce(sts.avg_training_effectiveness / 100 * 0.15, 0) +
            -- уровень навыков (10% вес)
            coalesce(ssi.avg_combat_skill_level / 20 * 0.1, 0)
        )::numeric(10,3)
    ) as overall_effectiveness_score,
    -- связанные сущности
    json_build_object(
        'member_ids', (
            select json_agg(dwarf_id)
            from squad_members
            where squad_id = sbs.squad_id and exit_date is null
        ),
        'equipment_ids', (
            select json_agg(equipment_id)
            from squad_equipment
            where squad_id = sbs.squad_id
        ),
        'battle_report_ids', (
            select json_agg(report_id)
            from squad_battles
            where squad_id = sbs.squad_id
        ),
        'training_ids', (
            select json_agg(schedule_id)
            from squad_training
            where squad_id = sbs.squad_id
        )
    ) as related_entities
from squad_battle_stats sbs
left join squad_member_stats sms 
    on sbs.squad_id = sms.squad_id
left join squad_equipment_stats ses 
    on sbs.squad_id = ses.squad_id
left join squad_training_stats sts 
    on sbs.squad_id = sts.squad_id
left join squad_skill_improvement ssi 
    on sbs.squad_id = ssi.squad_id
order by sbs.squad_id;
```