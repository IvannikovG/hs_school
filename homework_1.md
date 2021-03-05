# Homework 1
## Sql queries
### Themes

- Table creation
- Table inserion
- Hierarchical table update

##### Task 1. Generate table codes 
------------------

| id | code | desc |
| ---- | ---- | ---- |
| 1 | A | Some value |
| 2 | A.1 | Some sub value |
| 3 | A.1.01 | Some sub sub value 1 |
| 4 | A.1.02 | Some sub sub value 2 |
| 5 | A.1.02 | Some sub sub value 2.1 |
>
```
create table if not exists codes (
id int,
code text,
"desc" text
);

insert into codes (id, code, "desc")
values
(1, 'A', 'Some value'),
(2, 'A.1', 'Some sub value'),
(3, 'A.1.01', 'Some sub sub value 1'),
(4, 'A.1.02', 'Some sub sub value 2'),
(5, 'A.1.02.1', 'Some sub sub value 2.1');
```

##### Task 2 and 3. Generate table items
--------------
| id | code | display | path |
| ---- | ---- | ---- | ---- |
| 1 | A.1.02.1 | Some static item num 'somestr' | |
| 2 | A.1.02.1 | Some static item num 'somestr' | |
| 3 | A.1.02.1 | Some static item num 'somestr' | |
| . | A.1.02.1 | Some static item num 'somestr' | |
| n | A.1.02.1| Some static item num 'somestr'  | |

```
create table if not exists items (
id serial,
code text,
display text,
"path" jsonb
);

insert into items (code, display)
select
'A.1.02.1',
'static element id ' || left(md5(i::text), 3)
from generate_series(1, 1000000) s(i);
```

##### Task 4. Write a query that updates path by putting all parents in the hierarchy for the code in item.
--------------

```
with i_path as (
select i.id,
jsonb_agg(
       jsonb_build_object
       ('code', c.code,
        'desc', c.desc))
as path from items i
join codes c
on i.code ~ c.code and i.code != c.code
group by i.id)
update items as it
set path = ip.path
from i_path ip
where it.id = ip.id;
update items set path = path;

```
#### Test
```
select * from items limit 1;
```
| id | code | display | path |
| ---- | ---- | ---- | ---- |
| 1 | A.1.02.1 | Some static item num a87 | [{"code": "A", "desc": "Some value"}, {"code": "A.1", "desc": "Some sub value"}, {"code": "A.1.02", "desc": "Some sub sub value 2"}]|

