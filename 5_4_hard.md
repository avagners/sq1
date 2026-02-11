## Задача 5*: Многофакторный анализ угроз и безопасности крепости

Разработайте запрос, который комплексно анализирует безопасность крепости, учитывая:
- Историю всех атак существ и их исходов
- Эффективность защитных сооружений
- Соотношение между типами существ и результативностью обороны
- Оценку уязвимых зон на основе архитектуры крепости
- Корреляцию между сезонными факторами и частотой нападений
- Готовность военных отрядов и их расположение
- Эволюцию защитных способностей крепости со временем

Возможный вариант выдачи:

```json
{
  "total_recorded_attacks": 183,
  "unique_attackers": 42,
  "overall_defense_success_rate": 76.50,
  "security_analysis": {
    "threat_assessment": {
      "current_threat_level": "Moderate",
      "active_threats": [
        {
          "creature_type": "Goblin",
          "threat_level": 3,
          "last_sighting_date": "0205-08-12",
          "territory_proximity": 1.2,
          "estimated_numbers": 35,
          "creature_ids": [124, 126, 128, 132, 136]
        },
        {
          "creature_type": "Forgotten Beast",
          "threat_level": 5,
          "last_sighting_date": "0205-07-28",
          "territory_proximity": 3.5,
          "estimated_numbers": 1,
          "creature_ids": [158]
        }
      ]
    },
    "vulnerability_analysis": [
      {
        "zone_id": 15,
        "zone_name": "Eastern Gate",
        "vulnerability_score": 0.68,
        "historical_breaches": 8,
        "fortification_level": 2,
        "military_response_time": 48,
        "defense_coverage": {
          "structure_ids": [182, 183, 184],
          "squad_ids": [401, 405]
        }
      }
    ],
    "defense_effectiveness": [
      {
        "defense_type": "Drawbridge",
        "effectiveness_rate": 95.12,
        "avg_enemy_casualties": 12.4,
        "structure_ids": [185, 186, 187, 188]
      },
      {
        "defense_type": "Trap Corridor",
        "effectiveness_rate": 88.75,
        "avg_enemy_casualties": 8.2,
        "structure_ids": [201, 202, 203, 204]
      }
    ],
    "military_readiness_assessment": [
      {
        "squad_id": 403,
        "squad_name": "Crossbow Legends",
        "readiness_score": 0.92,
        "active_members": 7,
        "avg_combat_skill": 8.6,
        "combat_effectiveness": 0.85,
        "response_coverage": [
          {
            "zone_id": 12,
            "response_time": 0
          },
          {
            "zone_id": 15,
            "response_time": 36
          }
        ]
      }
    ],
    "security_evolution": [
      {
        "year": 203,
        "defense_success_rate": 68.42,
        "total_attacks": 38,
        "casualties": 42,
        "year_over_year_improvement": 3.20
      },
      {
        "year": 204,
        "defense_success_rate": 72.50,
        "total_attacks": 40,
        "casualties": 36,
        "year_over_year_improvement": 4.08
      }
    ]
  }
}
```

```sql
with attack_stats as (
    select
        count(*) as total_attacks,
        count(distinct creature_id) as unique_attackers,
        avg(case when outcome = 'отражена' then 1 else 0 end) * 100 as defense_success_rate
    from creature_attacks
),
threat_assessment as (
    select
        c.type as creature_type,
        c.threat_level,
        max(cs.date) as last_sighting_date,
        min(ct.distance_to_fortress) as territory_proximity,
        sum(c.estimated_population) as estimated_numbers,
        array_agg(c.creature_id) as creature_ids
    from creatures c
    left join creature_sightings cs 
        on c.creature_id = cs.creature_id
    left join creature_territories ct 
        on c.creature_id = ct.creature_id
    where c.active = true
    group by c.creature_id, c.type, c.threat_level
    order by c.threat_level desc
),
vulnerability_analysis as (
    select
        l.location_id as zone_id,
        l.name as zone_name,
        (l.fortification_level * 0.3 + l.wall_integrity * 0.2 + l.trap_density * 0.2 + l.choke_points * 0.1 + l.access_points * 0.2) / 10 as vulnerability_score,
        count(ca.attack_id) as historical_breaches,
        l.fortification_level,
        avg(ca.military_response_time_minutes) as avg_response_time,
        array_agg(distinct ds.structure_id) as structure_ids,
        array_agg(distinct ms.squad_id) as squad_ids
    from locations l
    left join creature_attacks ca 
        on l.location_id = ca.location_id
    left join defense_structures ds 
        on l.location_id = ds.location_id
    left join military_stations mst 
        on l.location_id = mst.location_id
    left join military_squads ms 
        on mst.squad_id = ms.squad_id
    group by l.location_id, l.name, l.fortification_level, l.wall_integrity, l.trap_density, l.choke_points, l.access_points
    having count(ca.attack_id) > 0 or l.fortification_level < 5
    order by vulnerability_score desc
),
defense_effectiveness as (
    select
        ds.type as defense_type,
        count(ca.attack_id) as total_attacks_vs_type,
        avg(case when ca.outcome = 'отражена' then 1 else 0 end) * 100 as effectiveness_rate,
        avg(ca.enemy_casualties) as avg_enemy_casualties,
        array_agg(distinct ds.structure_id) as structure_ids
    from defense_structures ds
    left join creature_attacks ca 
        on ds.location_id = ca.location_id
    where ds.status = 'operational'
    group by ds.type
    having count(ca.attack_id) > 0
    order by effectiveness_rate desc
),
military_readiness_assessment as (
    select
        ms.squad_id,
        ms.name as squad_name,
        case
            when count(sm.dwarf_id) = 0 then 0.5
            else least((count(sm.dwarf_id) * 1.0 / greatest(count(sm.dwarf_id), 5)) * 0.6 +
                  (avg(ds.effectiveness) * 0.4 / 10), 1.0)
        end as readiness_score,
        count(sm.dwarf_id) as active_members,
        avg(ds.effectiveness) as avg_combat_skill,
        case
            when count(sb.report_id) = 0 then 0.7
            else avg(case when sb.outcome = 'победа' then 1.0 else 0.5 end)
        end as combat_effectiveness
    from military_squads ms
    left join squad_members sm 
        on ms.squad_id = sm.squad_id and sm.exit_date is null
    left join defense_structures ds 
        on ms.fortress_id = (select fortress_id from locations where location_id = ds.location_id limit 1)
    left join squad_battles sb 
        on ms.squad_id = sb.squad_id
    group by ms.squad_id, ms.name
),
squad_coverage as (
    select
        ms.squad_id,
        json_agg(
            json_build_object(
                'zone_id', l.location_id,
                'response_time', coalesce(mca.avg_response_time, 0)
            )
        ) as response_coverage
    from military_squads ms
    left join military_stations mst 
        on ms.squad_id = mst.squad_id
    left join locations l 
        on mst.location_id = l.location_id
    left join (
        select
            ca.location_id,
            avg(ca.military_response_time_minutes) as avg_response_time
        from creature_attacks ca
        group by ca.location_id
    ) mca on l.location_id = mca.location_id
    group by ms.squad_id
),
security_evolution as (
    select
        extract(year from ca.date) as year,
        count(ca.attack_id) as total_attacks,
        sum(ca.casualties) as casualties,
        avg(case when ca.outcome = 'отражена' then 1 else 0 end) * 100 as defense_success_rate,
        lag(avg(case when ca.outcome = 'отражена' then 1 else 0 end) * 100, 1, 0)
            over (order by extract(year from ca.date)) as prev_year_rate,
        avg(case when ca.outcome = 'отражена' then 1 else 0 end) * 100 -
            lag(avg(case when ca.outcome = 'отражена' then 1 else 0 end) * 100, 1, 0)
            over (order by extract(year from ca.date)) as year_over_year_improvement
    from creature_attacks ca
    group by extract(year from ca.date)
    order by year
)
select 
    json_build_object(
        'total_recorded_attacks', (select total_attacks from attack_stats),
        'unique_attackers', (select unique_attackers from attack_stats),
        'overall_defense_success_rate', round((select defense_success_rate from attack_stats), 2),
        'security_analysis', json_build_object(
            'threat_assessment', json_build_object(
                'current_threat_level', 
                    case 
                        when (select max(threat_level) from threat_assessment) >= 8 then 'high'
                        when (select max(threat_level) from threat_assessment) >= 5 then 'moderate'
                        else 'low'
                    end,
                'active_threats', (
                    select json_agg(
                        json_build_object(
                            'creature_type', creature_type,
                            'threat_level', threat_level,
                            'last_sighting_date', last_sighting_date,
                            'territory_proximity', territory_proximity,
                            'estimated_numbers', estimated_numbers,
                            'creature_ids', creature_ids
                        )
                    )
                    from threat_assessment
                )
            ),
            'vulnerability_analysis', (
                select json_agg(
                    json_build_object(
                        'zone_id', zone_id,
                        'zone_name', zone_name,
                        'vulnerability_score', round(vulnerability_score, 2),
                        'historical_breaches', historical_breaches,
                        'fortification_level', fortification_level,
                        'military_response_time', round(avg_response_time, 0),
                        'defense_coverage', json_build_object(
                            'structure_ids', structure_ids,
                            'squad_ids', squad_ids
                        )
                    )
                )
                from vulnerability_analysis
            ),
            'defense_effectiveness', (
                select json_agg(
                    json_build_object(
                        'defense_type', defense_type,
                        'effectiveness_rate', round(effectiveness_rate, 2),
                        'avg_enemy_casualties', round(avg_enemy_casualties, 1),
                        'structure_ids', structure_ids
                    )
                )
                from defense_effectiveness
            ),
            'military_readiness_assessment', (
                select json_agg(
                    json_build_object(
                        'squad_id', mra.squad_id,
                        'squad_name', mra.squad_name,
                        'readiness_score', round(mra.readiness_score, 2),
                        'active_members', mra.active_members,
                        'avg_combat_skill', round(mra.avg_combat_skill, 1),
                        'combat_effectiveness', round(mra.combat_effectiveness, 2),
                        'response_coverage', coalesce(sc.response_coverage, '[]'::json)
                    )
                )
                from military_readiness_assessment mra
                left join squad_coverage sc on mra.squad_id = sc.squad_id
            ),
            'security_evolution', (
                select json_agg(
                    json_build_object(
                        'year', year,
                        'defense_success_rate', round(defense_success_rate, 2),
                        'total_attacks', total_attacks,
                        'casualties', casualties,
                        'year_over_year_improvement', round(year_over_year_improvement, 2)
                    )
                )
                from security_evolution
            )
        )
    ) as security_analysis_result;
```