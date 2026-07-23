### Поиск по тексту хранимых процедур и функций
```sql
SELECT 
    n.nspname AS schema_name, 
    pp.proname AS function_name, 
    pp.prosrc AS function_source 
FROM 
    pg_proc pp 
JOIN 
    pg_namespace n ON n.oid = pp.pronamespace 
WHERE 
    pp.prosrc ILIKE '%shkfor%'; 
```


### кто владелец ХП

```sql
SELECT
    r.rolname AS owner_name,
    p.prosecdef AS is_security_definer
FROM pg_proc p
JOIN pg_roles r ON p.proowner = r.oid
WHERE p.proname = 'create_daily_partitions';
```


### Update JSON
```sql
update the_table 
   set attr['is_default'] = to_jsonb(false); 
```
*https://dba.stackexchange.com/questions/295298/how-to-update-a-property-value-of-a-jsonb-field*
