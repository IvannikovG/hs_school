1. Unfinished
CREATE OR REPLACE FUNCTION public.jsonb_keys_recursive(_value jsonb)
 RETURNS TABLE(key text)
 LANGUAGE sql
AS $function$
WITH RECURSIVE _tree (key, value) AS (
  SELECT
    NULL   AS key,
    _value AS value
  UNION ALL
  (WITH typed_values AS (SELECT jsonb_typeof(value) as typeof, value FROM _tree)
   SELECT v.*
     FROM typed_values, LATERAL jsonb_each(value) v
     WHERE typeof = 'object'
   UNION ALL
   SELECT NULL, element
     FROM typed_values, LATERAL jsonb_array_elements(value) element
     WHERE typeof = 'array'
  )
)
SELECT key
  FROM _tree
  WHERE key IS NOT NULL
$function$;
----
select jsonb_keys_recursive(resource) from test_patient


2. Database traversal stats
```
with another_layer as (
with general_stats as(
with all_tables_stats as (
select
    schemaname,
    relname as table_name,
    pg_size_pretty(pg_relation_size(schemaname ||'.'||relname)) as table_size
from
(
  select
      schemaname,
      relname
  from pg_stat_user_tables order by relname
) t ),
table_columns as (
select
table_name, jsonb_agg(column_name) as columns
from information_schema.columns
group by table_name
),
indexes as (
select tablename as table_name, jsonb_agg(indexname) as indexes
from pg_indexes
group by table_name
),
index_name_types as (
SELECT
  amname as index_type,
  pgClass.relname   AS table_name,
  t.relname as index_name,
  pgClass.reltuples AS rows_estimate,
  pg_relation_size(t.relname::text) as index_size
FROM
  pg_class pgClass
JOIN
  pg_namespace pgNamespace ON (pgNamespace.oid = pgClass.relnamespace)
  join (select relname, indrelid, amname
    FROM pg_index idx
     JOIN pg_class cls ON cls.oid=idx.indexrelid
     JOIN pg_am am ON am.oid=cls.relam
     order by indrelid) t on relfilenode = indrelid
WHERE
  pgNamespace.nspname NOT IN ('pg_catalog', 'information_schema') AND
  pgClass.relkind='r'
)
select all_tables_stats.table_name,
schemaname as schema,
table_size,
indexes,
rows_estimate,
index_name,
index_size,
index_type,
pg_size_pretty(pg_indexes_size(all_tables_stats.table_name::text)) as total_indexes_size
from all_tables_stats
join table_columns on all_tables_stats.table_name = table_columns.table_name
join indexes on all_tables_stats.table_name = indexes.table_name
join index_name_types on all_tables_stats.table_name = index_name_types.table_name)
select schema, table_name,
jsonb_build_object('rows', jsonb_agg(rows_estimate),
                   'index_size', jsonb_agg(total_indexes_size),
                   index_name, jsonb_build_object('size', jsonb_agg(index_size),
                                               'index_type', jsonb_agg(index_type))) as index

from general_stats
group by schema, table_name, index_name)
select
jsonb_build_object(schema, jsonb_build_object(table_name, index))
from another_layer;
```
3. Unfinished
```
DROP FUNCTION knife_extract2(jsonb,jsonb,jsonb);
----
create or replace function knife_extract2(
jsonbinina jsonb,
jsonbpath jsonb,
target_value jsonb
) returns jsonb as
$$
begin
       return
       (select * from
                 jsonb_path_query(jsonbinina, concat('$.', jsonbpath)::jsonpath, '{}'::jsonb));
end
$$ language plpgsql;
----

select * from
jsonb_path_query('{"a":{"b": [1,2,3,4,5]}}', '$.a.b ? (@ >= $min && @ <= $max)', '{"min":2,"max":4}');
----
select knife_extract2('{"a": 2}'::jsonb, '{"a": 2}'::jsonb, '{"a": 2}'::jsonb)
```
