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

CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    password VARCHAR(255) NOT NULL
);

CREATE TABLE threads (
    id SERIAL PRIMARY KEY,
    title VARCHAR(255) NOT NULL,
    description TEXT NOT NULL,
    created TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    author_id INT REFERENCES users(id)
);

CREATE TABLE responses (
    id SERIAL PRIMARY KEY,
    thread_id INT REFERENCES threads(id),
    comment TEXT NOT NULL,
    created TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    author_id INT REFERENCES users(id)
);

```
  
