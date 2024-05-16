exam-database

```
$ cd /course_data/labs/exam

sudo apt install dos2unix -y
dos2unix jumpbox_shell.sh
dos2unix database-2_shell.sh

./jumpbox_shell.sh
psql "postgresql://admin:Ct=Snackul4@database-1.int.agency.test/forum_data"
CREATE USER "db_app_user" WITH PASSWORD '0205Sopwith.Camel';
GRANT ALL PRIVILEGES ON DATABASE forum_data TO "db_app_user";
GRANT CONNECT ON DATABASE "forum_data" TO "db_app_user";
```
  
