Postgres role hell
==================
tl;dr Postgres roles and permissions aren't that easy to work with.

For a production system you'll want to roll over your database credentials ever so often.

The usual way to do this with Postgres -- as far as I have been able to figure out -- is to
create two users:

- One persistent user, e.g. "dbappowner" -- this will never be deleted and also doesn't have a password.
  This user should own all the tables, and have the approriate permissions to your tables and database objects.
  ```CREATE ROLE dbappowner NOLOGIN;```
- One temporary user, e.g. "dbappuser-2021-03-04abcd"
  ```CREATE ROLE "dbappuser-2021-03-04abcd" PASSWORD 'zyxwvutsr' LOGIN;
  GRANT ROLE dbappowner TO "dbappuser-2021-03-04abcd";
  ```

This ensures that when rotate credentials -- deprecating "dbappuser-2021-03-04abcd" and creating a new
user "dbappuser-2021-05-19defg" with a different password. You can simply `GRANT ROLE dbappowner TO "dbappuser-2021-05-19defg"`
and you'll be able to do the things you need.

However in order to make this actually work with your app, for example when creating new tables or migrating
you have to ensure that you do `SET ROLE "dbappowner";` so that the temporary user doesn't end up owning the tables that created.

For Django, it means you either have to use something like the [django-postgresql-setrole]-package, or switch
to `dbappowner` when migrating.


An example from the shell
-------------------------

    eduflow=> DROP ROLE eduflow_owner;
    ERROR:  permission denied to drop role -- fair enough!
    
    eduflow=> REVOKE eduflow_owner FROM "eduflowuser-2020-11-04-5gkqdg";
    REVOKE ROLE -- Okay, seems like we succeeded in revoking the role

Let's log in as "eduflowuser-2020-11-04-5gkqdg" and check

    > psql -h 'eduflow-postgres-staging.cwfrbet8cw0x.eu-west-1.rds.amazonaws.com' -U eduflowuser-2020-11-04-5gkqdg eduflow
    Password for user eduflowuser-2020-11-04-5gkqdg:
    psql (11.10, server 10.13)
    SSL connection (protocol: TLSv1.2, cipher: ECDHE-RSA-AES256-GCM-SHA384, bits: 256, compression: off)
    Type "help" for help.
    
    eduflow=> SELECT * FROM user;
         user
    ---------------
     eduflow_owner

WAT??? Didn't we just revoke this role from "eduflowuser-2020-11-04-5gkqdg"?

     eduflow=>  SELECT oid, rolname FROM pg_roles WHERE pg_has_role( 'eduflowuser-2020-11-04-5gkqdg', oid, 'member');
      oid  |            rolname
    -------+-------------------------------
      3373 | pg_monitor
      3374 | pg_read_all_settings
      3375 | pg_read_all_stats
      3377 | pg_stat_scan_tables
      4200 | pg_signal_backend
     16386 | rds_superuser
     16387 | rds_replication
     16389 | rds_password
     16393 | eduflowstaging
     21137 | eduflowuser-2020-11-04-5gkqdg
     21132 | eduflow_owner
    (11 rows)


    eduflow=> SELECT * FROM "user";
    ERROR:  permission denied for relation user


    eduflow=> SET ROLE eduflowstaging;
    SET

[django-postgresql-setrole]: https://github.com/jdelic/django-postgresql-setrole
