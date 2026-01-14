## Задача 1*: Анализ эффективности экспедиций

Напишите запрос, который определит наиболее и наименее успешные экспедиции, учитывая:
- Соотношение выживших участников к общему числу
- Ценность найденных артефактов
- Количество обнаруженных новых мест
- Успешность встреч с существами (отношение благоприятных исходов к неблагоприятным)
- Опыт, полученный участниками (сравнение навыков до и после)

```sql
with expedition_stats as (
    select
        e.expedition_id,
        e.destination,
        e.status,
        e.departure_date,
        e.return_date,
        -- Соотношение выживших участников к общему числу
        case
            when (select count(*) from expedition_members em where em.expedition_id = e.expedition_id) > 0
            then round(
                (select count(*)::numeric from expedition_members em
                 where em.expedition_id = e.expedition_id and em.survived = true) /
                (select count(*)::numeric from expedition_members em
                 where em.expedition_id = e.expedition_id) * 100, 2
            )
            else null
        end as survival_rate,
        -- Ценность найденных артефактов
        coalesce((select sum(ea.value) from expedition_artifacts ea
                  where ea.expedition_id = e.expedition_id), 0) as artifacts_value,
        -- Количество обнаруженных новых мест
        (select count(*) from expedition_sites es
         where es.expedition_id = e.expedition_id) as discovered_sites,
        -- Успешность встреч с существами (отношение благоприятных исходов к неблагоприятным)
        case
            when (select count(*) from expedition_creatures ec where ec.expedition_id = e.expedition_id) > 0
            then round(
                (select count(*)::numeric from expedition_creatures ec
                 where ec.expedition_id = e.expedition_id and ec.outcome in ('victory', 'stalemate', 'avoided')) /
                (select count(*)::numeric from expedition_creatures ec
                 where ec.expedition_id = e.expedition_id) * 100, 2
            )
            else null
        end as encounter_success_rate,
        -- продолжительность экспедиции в днях
        case
            when e.return_date is not null then e.return_date - e.departure_date
            else current_date - e.departure_date
        end as expedition_duration,
        -- количество участников (для расчета улучшения навыков)
        (select count(*) from expedition_members em where em.expedition_id = e.expedition_id) as participant_count,
        -- Опыт, полученный участниками (сравнение навыков до и после)
        round(
            coalesce(
                (select avg(post.level - pre.level)::numeric
                 from expedition_members em
                 join dwarf_skills pre on em.dwarf_id = pre.dwarf_id and pre.date < e.departure_date
                 join dwarf_skills post on em.dwarf_id = post.dwarf_id and post.date > e.return_date
                 where em.expedition_id = e.expedition_id
                 and pre.skill_id = post.skill_id
                 and post.level > pre.level), 0
            ), 2
        ) as skill_improvement,
        -- связанные сущности
        json_build_object(
            'member_ids', coalesce((
                select json_agg(em.dwarf_id)
                from expedition_members em
                where em.expedition_id = e.expedition_id
            ), '[]'::json),
            'artifact_ids', coalesce((
                select json_agg(ea.artifact_id)
                from expedition_artifacts ea
                where ea.expedition_id = e.expedition_id
            ), '[]'::json),
            'site_ids', coalesce((
                select json_agg(es.site_id)
                from expedition_sites es
                where es.expedition_id = e.expedition_id
            ), '[]'::json)
        ) as related_entities
    from
        expeditions e
    where
        e.status = 'completed'
        and (select count(*) from expedition_members em where em.expedition_id = e.expedition_id) > 0
),
success_scores as (
    select
        expedition_id,
        destination,
        status,
        survival_rate,
        artifacts_value,
        discovered_sites,
        encounter_success_rate,
        skill_improvement,
        expedition_duration,
        related_entities,
        -- общий показатель успеха (по нему определяем успешность экспедиции)
        round(
            (
                -- вес процента выживших: 40%
                (coalesce(survival_rate, 0) * 0.4) +
                -- вес стоимости артефактов: 20% (нормализовано к шкале 0-100)
                (least(artifacts_value / 1000, 100) * 0.2) +
                -- вес обнаруженных локаций: 15%
                (least(discovered_sites * 25, 100) * 0.15) +
                -- вес успешности встреч: 25%
                (coalesce(encounter_success_rate, 0) * 0.25)
            ) / 100, 2
        ) as overall_success_score
    from
        expedition_stats
)
select
    expedition_id,
    destination,
    status,
    survival_rate,
    artifacts_value,
    discovered_sites,
    encounter_success_rate,
    skill_improvement,
    expedition_duration,
    overall_success_score,
    related_entities
from
    success_scores
order by
    overall_success_score desc;
```