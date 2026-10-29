# SQL templates (Supabase / Postgres)

Replace the example tables (`crm_customers`, `crm_products`, `crm_sales`, `erp_locations`) and columns
with the user's sources. Every block is safe to rerun.

## Contents
1. Schemas
2. Bronze tables
3. Loading bronze
4. Silver procedure
5. Gold star schema + load procedure
6. Quality checks
7. Security (read-only role, RLS)
8. Scheduling with pg_cron

---

## 1. Schemas

```sql
create schema if not exists bronze;
create schema if not exists silver;
create schema if not exists gold;
```

## 2. Bronze tables

One table per CSV. Column order = CSV header order, names in lower case.

```sql
-- source: crm/customers.csv
create table if not exists bronze.crm_customers (
    cst_id             integer,
    cst_key            text,
    cst_firstname      text,
    cst_lastname       text,
    cst_marital_status text,
    cst_gndr           text,
    cst_create_date    date
);

-- source: crm/sales.csv  (dates delivered as integers yyyymmdd)
create table if not exists bronze.crm_sales (
    sls_ord_num  text,
    sls_prd_key  text,
    sls_cust_id  integer,
    sls_order_dt integer,
    sls_ship_dt  integer,
    sls_due_dt   integer,
    sls_sales    integer,
    sls_quantity integer,
    sls_price    integer
);
```

## 3. Loading bronze

### psql through the session pooler (PowerShell)

```powershell
# Set in the session only – never store the password in a file.
$env:PGPASSWORD = '<password>'
$env:PGHOST     = 'aws-0-<region>.pooler.supabase.com'   # Supabase → Connect → Session pooler
$env:PGPORT     = '5432'
$env:PGDATABASE = 'postgres'
$env:PGUSER     = 'postgres.<project-ref>'               # pooler needs user.project-ref

psql -c "truncate bronze.crm_customers"
psql -c "\copy bronze.crm_customers from 'C:/data/crm/customers.csv' with (format csv, header true)"
psql -tAc "select count(*) from bronze.crm_customers"
```

### Table Editor
Lower-case the header line first (columns are matched by exact name). Example in PowerShell:

```powershell
$src = 'C:\data\erp\LOC_A101.csv'; $dst = 'C:\data\import\erp_locations.csv'
$lines = Get-Content $src
@($lines[0].ToLower()) + $lines[1..($lines.Count-1)] | Set-Content $dst -Encoding utf8
```

### Row-count check

```sql
select table_name,
       (xpath('/row/c/text()', query_to_xml(
           format('select count(*) as c from %I.%I', table_schema, table_name), false, true, '')))[1]::text::int as rows
from information_schema.tables
where table_schema in ('bronze', 'silver', 'gold')
order by table_schema, table_name;
```

## 4. Silver procedure

Create the silver tables with clean types first (same columns, `date` instead of integer dates, plus
an optional `dwh_create_date timestamptz default now()`). Then:

```sql
create or replace procedure silver.load_silver()
language plpgsql
set search_path = ''
as $$
declare
    v_rows bigint;
begin
    -- Customers: trim, standardise codes, keep the latest row per customer
    truncate table silver.crm_customers;
    insert into silver.crm_customers (cst_id, cst_key, cst_firstname, cst_lastname,
                                      cst_marital_status, cst_gndr, cst_create_date)
    select cst_id,
           trim(cst_key),
           trim(cst_firstname),
           trim(cst_lastname),
           case upper(trim(cst_marital_status)) when 'S' then 'Single' when 'M' then 'Married' else 'n/a' end,
           case upper(trim(cst_gndr))           when 'F' then 'Female' when 'M' then 'Male'    else 'n/a' end,
           cst_create_date
    from (
        select *, row_number() over (partition by cst_id order by cst_create_date desc) as rn
        from bronze.crm_customers
        where cst_id is not null
    ) t
    where rn = 1;
    get diagnostics v_rows = row_count;
    raise notice 'silver.crm_customers: % rows', v_rows;

    -- Sales: parse yyyymmdd integers, repair inconsistent sales/price
    truncate table silver.crm_sales;
    insert into silver.crm_sales (sls_ord_num, sls_prd_key, sls_cust_id, sls_order_dt, sls_ship_dt,
                                  sls_due_dt, sls_sales, sls_quantity, sls_price)
    select sls_ord_num,
           sls_prd_key,
           sls_cust_id,
           case when sls_order_dt <= 0 or length(sls_order_dt::text) <> 8 then null
                else to_date(sls_order_dt::text, 'YYYYMMDD') end,
           case when sls_ship_dt  <= 0 or length(sls_ship_dt::text)  <> 8 then null
                else to_date(sls_ship_dt::text,  'YYYYMMDD') end,
           case when sls_due_dt   <= 0 or length(sls_due_dt::text)   <> 8 then null
                else to_date(sls_due_dt::text,   'YYYYMMDD') end,
           case when sls_sales is null or sls_sales <= 0 or sls_sales <> sls_quantity * abs(sls_price)
                then sls_quantity * abs(sls_price) else sls_sales end,
           sls_quantity,
           case when sls_price is null or sls_price <= 0
                then sls_sales / nullif(sls_quantity, 0) else sls_price end
    from bronze.crm_sales;

    -- Locations: normalise country variants and key format
    truncate table silver.erp_locations;
    insert into silver.erp_locations (cid, cntry)
    select replace(cid, '-', ''),
           case when trim(cntry) = 'DE' then 'Germany'
                when trim(cntry) in ('US', 'USA') then 'United States'
                when trim(cntry) = '' or cntry is null then 'n/a'
                else trim(cntry) end
    from bronze.erp_locations;

    -- Versioned dimension: end date = next start date - 1 day
    -- prd_end_dt = lead(prd_start_dt) over (partition by prd_key order by prd_start_dt) - interval '1 day'
exception
    when others then
        raise notice 'load_silver failed: %', sqlerrm;
        raise;   -- re-raise so the caller sees the failure
end;
$$;
```

Run with `call silver.load_silver();`.

## 5. Gold star schema + load procedure

```sql
create table if not exists gold.dim_customers (
    customer_key    integer primary key,
    customer_id     integer not null,
    customer_number text,
    first_name      text,
    last_name       text,
    country         text,
    marital_status  text,
    gender          text,
    birthdate       date,
    create_date     date
);

create table if not exists gold.dim_products (
    product_key    integer primary key,
    product_id     integer not null,
    product_number text,
    product_name   text,
    category       text,
    subcategory    text,
    cost           integer,       -- unit cost
    product_line   text,
    start_date     date
);

create table if not exists gold.fact_sales (
    order_number  text,
    product_key   integer references gold.dim_products (product_key),
    customer_key  integer references gold.dim_customers (customer_key),
    order_date    date,
    shipping_date date,
    due_date      date,
    sales_amount  integer,
    quantity      integer,
    price         integer
);

create index if not exists fact_sales_product_key_idx  on gold.fact_sales (product_key);
create index if not exists fact_sales_customer_key_idx on gold.fact_sales (customer_key);
create index if not exists fact_sales_order_date_idx   on gold.fact_sales (order_date);

create or replace procedure gold.load_gold()
language plpgsql
set search_path = ''
as $$
begin
    truncate table gold.fact_sales, gold.dim_customers, gold.dim_products;

    insert into gold.dim_customers
    select row_number() over (order by c.cst_id),
           c.cst_id, c.cst_key, c.cst_firstname, c.cst_lastname,
           coalesce(l.cntry, 'n/a'), c.cst_marital_status, c.cst_gndr,
           null::date, c.cst_create_date
    from silver.crm_customers c
    left join silver.erp_locations l on l.cid = c.cst_key;

    insert into gold.dim_products
    select row_number() over (order by p.prd_start_dt, p.prd_key),
           p.prd_id, p.prd_key, p.prd_nm, x.cat, x.subcat, p.prd_cost, p.prd_line, p.prd_start_dt
    from silver.crm_products p
    left join silver.erp_categories x on x.id = p.cat_id
    where p.prd_end_dt is null;            -- current version only

    insert into gold.fact_sales
    select s.sls_ord_num, p.product_key, c.customer_key,
           s.sls_order_dt, s.sls_ship_dt, s.sls_due_dt,
           s.sls_sales, s.sls_quantity, s.sls_price
    from silver.crm_sales s
    left join gold.dim_products  p on p.product_number = s.sls_prd_key
    left join gold.dim_customers c on c.customer_id    = s.sls_cust_id;
end;
$$;
```

Run with `call gold.load_gold();`. If a gold table is read by a role that gets RLS, see section 7.

## 6. Quality checks (expect 0 everywhere)

```sql
select 'dim_customers: duplicate key' as check_name, count(*) as errors
from (select customer_key from gold.dim_customers group by 1 having count(*) > 1) x
union all
select 'dim_products: duplicate key', count(*)
from (select product_key from gold.dim_products group by 1 having count(*) > 1) x
union all
select 'fact_sales: orphan customer_key', count(*)
from gold.fact_sales f left join gold.dim_customers c using (customer_key) where c.customer_key is null
union all
select 'fact_sales: orphan product_key', count(*)
from gold.fact_sales f left join gold.dim_products p using (product_key) where p.product_key is null
union all
select 'silver sales: sales <> quantity * price', count(*)
from silver.crm_sales where sls_sales <> sls_quantity * sls_price
union all
select 'silver sales: order after ship/due', count(*)
from silver.crm_sales where sls_order_dt > sls_ship_dt or sls_order_dt > sls_due_dt
union all
select 'info: fact rows without order_date', count(*)
from gold.fact_sales where order_date is null;

-- Distinct values of standardised columns
select 'gender' as col, string_agg(distinct gender, ', ') from gold.dim_customers
union all select 'country', string_agg(distinct country, ', ') from gold.dim_customers;
```

## 7. Security (read-only role, RLS)

```sql
do $$
begin
    if not exists (select 1 from pg_roles where rolname = 'powerbi_reader') then
        create role powerbi_reader login nosuperuser nocreatedb nocreaterole noinherit connection limit 10;
    end if;
end
$$;

alter role powerbi_reader set default_transaction_read_only = on;

grant usage  on schema gold to powerbi_reader;
grant select on all tables in schema gold to powerbi_reader;
alter default privileges for role postgres in schema gold
    grant select on tables to powerbi_reader;

-- RLS: only needed when RLS is enabled on the table (check pg_class.relrowsecurity).
-- Without a policy, RLS is default-deny and Power BI sees 0 rows.
drop policy if exists powerbi_reader_select on gold.fact_sales;
create policy powerbi_reader_select on gold.fact_sales for select to powerbi_reader using (true);
-- repeat for every gold table

-- The user sets the password themselves:
--   alter role powerbi_reader with password '<password>';
-- Power BI user name via the pooler: powerbi_reader.<project-ref>
```

Check which gold tables have RLS:

```sql
select c.relname, c.relrowsecurity
from pg_class c join pg_namespace n on n.oid = c.relnamespace
where n.nspname = 'gold' and c.relkind = 'r';
```

## 8. Scheduling with pg_cron (optional)

```sql
create extension if not exists pg_cron;
select cron.schedule('nightly-medallion', '0 2 * * *',
                     $$call silver.load_silver(); call gold.load_gold();$$);
```
