```sql
create table roles (
role_id serial primary key,
role_name varchar(10) unique not null check (role_name in ('student', 'mentor'))
);
```

```sql
create table users (
user_id serial primary key,
full_name varchar(50) not null check (char_length(full_name) >= 5),
phone_number varchar(9) unique not null
);
```

```sql
create table groups (
group_id serial primary key,
group_name varchar(30) not null check (char_length(group_name) >= 3),
created_at date not null default current_date,
role_id int references roles (role_id) on delete set null,
mentor_user_id int references users (user_id) on delete set null
);
```

```sql
create table students (
student_id serial primary key,
enrolled_at date not null default current_date,
is_active boolean not null default false,
user_id int references users (user_id) on delete cascade,
group_id int references groups (group_id) on delete cascade
);
```

```sql
create view top10 as
select
s.student_id,
u.full_name as student_full_name,
s.enrolled_at as student_enrolled_at,
g.group_name,
m.full_name as mentor_full_name
from
students s
join
users u on s.user_id = u.user_id
left join
groups g on s.group_id = g.group_id
left join
users m on g.mentor_user_id = m.user_id
order by
s.enrolled_at desc
limit 10;
```

```sql
create materialized view top_3_groups as
select
gr.group_id,
gr.group_name,
gr.created_at as group_created_at,
count(st.student_id) as student_count
from
groups as gr
left join
students as st on gr.group_id = st.group_id
group by
gr.group_id, gr.group_name, gr.created_at
order by
student_count desc
limit 3;
```
```sql
create function group_of_mentor(p_mentor_id integer)
returns table (
mentor_id integer,
mentor_full_name varchar,
group_name varchar,
group_created_at date,
student_count bigint
)
language plpgsql
as $$
begin
return query
select
u.user_id as mentor_id,
u.full_name as mentor_full_name,
g.group_name,
g.created_at as group_created_at,
count(s.student_id) as student_count
from
users u
join
groups g on u.user_id = g.mentor_user_id
left join
students s on g.group_id = s.group_id
where
u.user_id = p_mentor_id
group by
u.user_id, u.full_name, g.group_id, g.group_name, g.created_at
order by
g.group_name;
end;
$$;
```
```sql
create or replace procedure student_activator(p_group_id integer)
language plpgsql
as $$
begin
update students
set
is_active = true
where
group_id = p_group_id and is_active = false;
end;
$$;
```

```sql
insert into roles (role_name) values ('student'), ('mentor');
```

```sql
insert into users (full_name, phone_number) values
('ali valiyev', '998901234'),
('nodira karimova', '998915678'),
('javohir ahmedov', '998939876'),
('diana ismoilova', '998943210'),
('farhod rustamov', '998951122'),
('guli halimova', '998901111'),
('sardor sayidov', '998912222');

insert into groups (group_name, mentor_user_id, role_id) values
('web-01', (select user_id from users where full_name = 'ali valiyev'), (select role_id from roles where role_name = 'mentor')),
('mobile-02', (select user_id from users where full_name = 'nodira karimova'), (select role_id from roles where role_name = 'mentor')),
('design-03', (select user_id from users where full_name = 'javohir ahmedov'), (select role_id from roles where role_name = 'mentor')),
('marketing-04', (select user_id from users where full_name = 'ali valiyev'), (select role_id from roles where role_name = 'mentor'));

insert into students (user_id, group_id, is_active) values
((select user_id from users where full_name = 'diana ismoilova'), (select group_id from groups where group_name = 'web-01'), true),
((select user_id from users where full_name = 'farhod rustamov'), (select group_id from groups where group_name = 'web-01'), false),
((select user_id from users where full_name = 'guli halimova'), (select group_id from groups where group_name = 'web-01'), true),
((select user_id from users where full_name = 'sardor sayidov'), (select group_id from groups where group_name = 'mobile-02'), false),
((select user_id from users where full_name = 'ali valiyev'), (select group_id from groups where group_name = 'mobile-02'), true);

select '--- top 10 talaba ma''lumotlari ---' as info;
select * from top10;
```

```sql
select '--- eng ko''p talabaga ega 3 ta guruh ---' as info;
select * from top_3_groups;


select '--- materialized view yangilash ---' as info;
refresh materialized view top_3_groups;


select '--- mentorning guruhlari ---' as info;
select * from group_of_mentor((select user_id from users where full_name = 'ali valiyev'));


select '--- talabalarni faollashtirish protsedurasini chaqirish ---' as info;
call student_activator((select group_id from groups where group_name = 'web-01'));
```
```sql
select
s.student_id,
u.full_name,
s.is_active,
g.group_name
from
students s
join
users u on s.user_id = u.user_id
join
groups g on s.group_id = g.group_id
where
g.group_name = 'web-01';
```

