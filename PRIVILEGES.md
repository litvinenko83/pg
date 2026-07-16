## права пользователей в БД

```sql
WITH RECURSIVE
  user_roles (rolname, oid) AS (
    SELECT
      rolname,
      oid
    FROM
      pg_roles
    WHERE
      rolcanlogin
    UNION
    SELECT
      ur.rolname,
      m.roleid
    FROM
      user_roles ur,
      pg_auth_members m
    WHERE
      m.member = ur.oid
  ),
  grants AS (
    SELECT
      (
        ACLEXPLODE(
          COALESCE(c.relacl, ACLDEFAULT('r'::"char", c.relowner))
        )
      ).grantee AS grantee,
      (
        ACLEXPLODE(
          COALESCE(c.relacl, ACLDEFAULT('r'::"char", c.relowner))
        )
      ).privilege_type AS privilege_type
    FROM
      pg_class c
  )
SELECT
  ur.rolname AS USER,
  ARRAY_AGG(DISTINCT ur.oid::regrole::TEXT) AS roles,
  REPLACE(
    ARRAY_AGG(DISTINCT g.privilege_type)::TEXT,
    ',NULL',
    ''
  ) AS PRIVILEGES
FROM
  user_roles ur
  LEFT JOIN grants g ON ur.oid = g.grantee
GROUP BY
  ur.rolname
ORDER BY
  ur.rolname;
```
