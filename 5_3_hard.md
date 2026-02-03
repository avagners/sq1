## Задача 4*: Комплексный анализ торговых отношений и их влияния на крепость

Разработайте запрос, анализирующий торговые отношения со всеми цивилизациями, оценивая:
- Баланс торговли с каждой цивилизацией за все время
- Влияние товаров каждого типа на экономику крепости
- Корреляцию между торговлей и дипломатическими отношениями
- Эволюцию торговых отношений во времени
- Зависимость крепости от определенных импортируемых товаров
- Эффективность экспорта продукции мастерских

Возможный вариант выдачи:
```json
{
  "total_trading_partners": 5,
  "all_time_trade_value": 15850000,
  "all_time_trade_balance": 1250000,
  "civilization_data": {
    "civilization_trade_data": [
      {
        "civilization_type": "Human",
        "total_caravans": 42,
        "total_trade_value": 5240000,
        "trade_balance": 840000,
        "trade_relationship": "Favorable",
        "diplomatic_correlation": 0.78,
        "caravan_ids": [1301, 1305, 1308, 1312, 1315]
      },
      {
        "civilization_type": "Elven",
        "total_caravans": 38,
        "total_trade_value": 4620000,
        "trade_balance": -280000,
        "trade_relationship": "Unfavorable",
        "diplomatic_correlation": 0.42,
        "caravan_ids": [1302, 1306, 1309, 1316, 1322]
      }
    ]
  },
  "critical_import_dependencies": {
    "resource_dependency": [
      {
        "material_type": "Exotic Metals",
        "dependency_score": 2850.5,
        "total_imported": 5230,
        "import_diversity": 4,
        "resource_ids": [202, 208, 215]
      },
      {
        "material_type": "Lumber",
        "dependency_score": 1720.3,
        "total_imported": 12450,
        "import_diversity": 3,
        "resource_ids": [203, 209, 216]
      }
    ]
  },
  "export_effectiveness": {
    "export_effectiveness": [
      {
        "workshop_type": "Smithy",
        "product_type": "Weapons",
        "export_ratio": 78.5,
        "avg_markup": 1.85,
        "workshop_ids": [301, 305, 310]
      },
      {
        "workshop_type": "Jewelery",
        "product_type": "Ornaments",
        "export_ratio": 92.3,
        "avg_markup": 2.15,
        "workshop_ids": [304, 309, 315]
      }
    ]
  },
  "trade_timeline": {
    "trade_growth": [
      {
        "year": 205,
        "quarter": 1,
        "quarterly_value": 380000,
        "quarterly_balance": 20000,
        "trade_diversity": 3
      },
      {
        "year": 205,
        "quarter": 2,
        "quarterly_value": 420000,
        "quarterly_balance": 35000,
        "trade_diversity": 4
      }
    ]
  }
}
```

Решение: 
```sql
with civilization_trade_data as (
    select 
        c.civilization_type,
        count(distinct c.caravan_id) as total_caravans,
        coalesce(sum(tt.value), 0) as total_trade_value,
        -- торговый баланс: экспорт (положительный) минус импорт (отрицательный)
        coalesce(sum(
            case 
                when tt.balance_direction = 'export' then tt.value
                when tt.balance_direction = 'import' then -tt.value
                else 0
            end
        ), 0) as trade_balance,
        -- коэффициент корреляции между объемом торговли и изменением дипотношений
        coalesce(round(corr(tt.value, coalesce(de.relationship_change, 0))::numeric, 2), 0) as diplomatic_correlation,
        array_agg(distinct c.caravan_id order by c.caravan_id) as caravan_ids
    from caravans c
    left join trade_transactions tt on c.caravan_id = tt.caravan_id
    left join diplomatic_events de on c.caravan_id = de.caravan_id
    group by c.civilization_type
),
critical_import_dependencies as (
    select 
        cg.material_type,
        -- высокий score = критическая зависимость от импорта этого типа
        round(sum(cg.value * cg.quantity * cg.price_fluctuation) / nullif(count(distinct cg.caravan_id), 0), 1) as dependency_score,
        sum(cg.quantity) as total_imported,
        count(distinct cg.goods_id) as import_diversity,
        array_agg(distinct r.resource_id order by r.resource_id) filter (where r.resource_id is not null) as resource_ids
    from caravan_goods cg
    join caravans ca on cg.caravan_id = ca.caravan_id
    left join resources r on lower(r.type) = lower(cg.material_type)
    where cg.type = 'import'
    group by cg.material_type
    having sum(cg.quantity) > 0
),
export_effectiveness as (
    select 
        w.type as workshop_type,
        p.type as product_type,
        -- процент продукции, проданной на экспорт
        round(100.0 * count(distinct cg.goods_id) / nullif(count(distinct p.product_id), 0), 1) as export_ratio,
        round(avg(cg.price_fluctuation), 2) as avg_markup,
        array_agg(distinct w.workshop_id order by w.workshop_id) as workshop_ids
    from caravan_goods cg
    join products p on cg.original_product_id = p.product_id
    join workshops w on p.workshop_id = w.workshop_id
    where cg.type = 'export'
    group by w.type, p.type
),
trade_timeline as (
    select 
        extract(year from tt.date)::integer as year,
        extract(quarter from tt.date)::integer as quarter,
        sum(tt.value) as quarterly_value,
        sum(case 
            when tt.balance_direction = 'export' then tt.value
            when tt.balance_direction = 'import' then -tt.value
            else 0
        end) as quarterly_balance,
        count(distinct ca.civilization_type) as trade_diversity
    from trade_transactions tt
    join caravans ca on tt.caravan_id = ca.caravan_id
    group by extract(year from tt.date), extract(quarter from tt.date)
),
total_stats as (
    select 
        count(distinct civilization_type) as total_trading_partners,
        coalesce(sum(total_trade_value), 0) as all_time_trade_value,
        coalesce(sum(trade_balance), 0) as all_time_trade_balance
    from civilization_trade_data
)
select json_build_object(
    'total_trading_partners', ts.total_trading_partners,
    'all_time_trade_value', ts.all_time_trade_value,
    'all_time_trade_balance', ts.all_time_trade_balance,
    'civilization_data', json_build_object(
        'civilization_trade_data', coalesce((
            select json_agg(json_build_object(
                'civilization_type', ctd.civilization_type,
                'total_caravans', ctd.total_caravans,
                'total_trade_value', ctd.total_trade_value,
                'trade_balance', ctd.trade_balance,
                -- Оценка отношений на основе баланса:
                -- Положительный баланс = выгодные отношения для крепости
                'trade_relationship', case 
                    when ctd.trade_balance > 0 then 'favorable'
                    when ctd.trade_balance < 0 then 'unfavorable'
                    else 'neutral'
                end,
                'diplomatic_correlation', ctd.diplomatic_correlation,
                'caravan_ids', ctd.caravan_ids
            ) order by ctd.total_trade_value desc)
            from civilization_trade_data ctd
        ), '[]'::json)
    ),
    'critical_import_dependencies', json_build_object(
        'resource_dependency', coalesce((
            select json_agg(json_build_object(
                'material_type', cid.material_type,
                'dependency_score', cid.dependency_score,
                'total_imported', cid.total_imported,
                'import_diversity', cid.import_diversity,
                'resource_ids', cid.resource_ids
            ) order by cid.dependency_score desc)
            from critical_import_dependencies cid
        ), '[]'::json)
    ),
    'export_effectiveness', json_build_object(
        'export_effectiveness', coalesce((
            select json_agg(json_build_object(
                'workshop_type', ee.workshop_type,
                'product_type', ee.product_type,
                'export_ratio', ee.export_ratio,
                'avg_markup', ee.avg_markup,
                'workshop_ids', ee.workshop_ids
            ) order by ee.export_ratio desc)
            from export_effectiveness ee
        ), '[]'::json)
    ),
    'trade_timeline', json_build_object(
        'trade_growth', coalesce((
            select json_agg(json_build_object(
                'year', tl.year,
                'quarter', tl.quarter,
                'quarterly_value', tl.quarterly_value,
                'quarterly_balance', tl.quarterly_balance,
                'trade_diversity', tl.trade_diversity
            ) order by tl.year, tl.quarter)
            from trade_timeline tl
        ), '[]'::json)
    )
) as result
from total_stats ts;
```