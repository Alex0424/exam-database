exam-database
alexander och tan

jag följde stegen från readme.md
här är kommandon jag gjorde:

```
cd /course_data/labs/exam

sudo apt install dos2unix -y
dos2unix jumpbox_shell.sh
dos2unix database-2_shell.sh

./jumpbox_shell.sh
psql "postgresql://admin:Ct=Snackul4@database-1.int.agency.test/forum_data"
CREATE USER "db_app_user" WITH PASSWORD '0205Sopwith.Camel';
GRANT ALL PRIVILEGES ON DATABASE forum_data TO "db_app_user";
GRANT CONNECT ON DATABASE "forum_data" TO "db_app_user";
```

Enable DEBUG for more info in logs
```
vim /course_data/labs/exam/forum_app/configuration.yml
- log_level: "INFO"
+ log_level: "DEBUG"
```
## PostgreSQL Tables:
```
CREATE TABLE threads (
    id SERIAL PRIMARY KEY,
    title TEXT UNIQUE NOT NULL,
    topic_description TEXT NOT NULL,
    author TEXT NOT NULL,
    author_ip INET,
    created TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE responses (
    id SERIAL PRIMARY KEY,
    thread_id INTEGER REFERENCES threads(id) ON DELETE CASCADE,
    comment TEXT NOT NULL,
    author TEXT NOT NULL,
    author_ip INET,
    responded TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE deleted_items (
    id SERIAL PRIMARY KEY,
    item_type TEXT NOT NULL CHECK (item_type IN ('thread', 'response')),
    original_id INTEGER NOT NULL, 
    thread_id INTEGER, -- Added to store the thread ID
    title TEXT, -- For threads
    content TEXT, -- For responses or thread descriptions
    author TEXT NOT NULL,
    author_ip INET,
    deleted_at TIMESTAMP NOT NULL DEFAULT NOW()
);    
```

## python.py code:

```
# SPDX-FileCopyrightText: © 2024 Menacit AB <foss@menacit.se>
# SPDX-License-Identifier: CC-BY-SA-4.0
# X-Context: Database course - Examination lab - Source code for "forum" web application

'''
forum - Web application for meaningful and productivity-boosting discussions.
'''

try:
    import logging as _log
    import time
    
    from flask import Flask, request, jsonify
    from flask_httpauth import HTTPBasicAuth
    from psycopg import connect as postgresql
    import yaml
    
except Exception as error_message:
    raise Exception(f'Failed to import required Python dependencies: {error_message}')

# -------------------------------------------------------------------------------------------------
# Load configuration from YAML file and validate that required keys/options are present
configuration_path = '/etc/app_configuration.yml'

try:
    with open(configuration_path, 'r') as file_handle:
        raw_configuration = yaml.safe_load(file_handle)
        
except Exception as error_message:
    raise Exception(f'Failed to load configuration from "{configuration_path}": {error_message}')

if not isinstance(raw_configuration, dict):
    raise Exception(f'The file "{configuration_path}" is not parsable as a dictionary')

for required_key in ['log_level', 'app_users', 'database_settings']:
    if not required_key in raw_configuration.keys():
        raise Exception(f'Missing option/key "{required_key}": ' + repr(raw_configuration))  
    
if not isinstance(raw_configuration['app_users'], dict):
    raise Exception('Option "app_users" must be a dictionary (key=username, value=password)')

if not isinstance(raw_configuration['database_settings'], dict):
    raise Exception(f'Option "database_settings" must be parseable as a dictionary')
    
for required_key in ['remote_hosts', 'user', 'password', 'database_name']:
    if not required_key in raw_configuration['database_settings'].keys():
        raise Exception(
            f'Missing "database_settings" option/key "{required_key}": ' + repr(raw_configuration))

if not isinstance(raw_configuration['database_settings']['remote_hosts'], list):
    raise Exception(
        f'"remote_hosts" option in "database_settings" must be a list: ' + repr(raw_configuration))

match raw_configuration['log_level']:
    case 'INFO':
        logger_level = _log.INFO

    case 'DEBUG':
        logger_level = _log.DEBUG
        
    case _:
        raise Exception('Option "log_level" is invalid (must be "INFO" or "DEBUG")')

_log.basicConfig(format='%(levelname)s: %(message)s', level=logger_level)
_log.debug('Loaded app configuration: ' + repr(raw_configuration))

app_users = raw_configuration['app_users']
database_settings = raw_configuration['database_settings']

# -------------------------------------------------------------------------------------------------
_log.info('Initializing PostgreSQL client with specified settings')

postgresql_hosts = ','.join(database_settings['remote_hosts'])
s = database_settings
target_uri = f'postgresql://{s['user']}:{s['password']}@{postgresql_hosts}/{s['database_name']}'
_log.debug(f'PostgreSQL connection URI: {target_uri}')

try:
    postgresql_client = postgresql(target_uri)
    _log.debug('Connected to database: ' + postgresql_client.info.dsn)
    
except Exception as error_message:
    raise Exception(f'Failed to connect to PostgreSQL database: "{error_message}"')

# -------------------------------------------------------------------------------------------------
_log.debug('Setting up Flask application and authentication extension')

app = Flask('forum')
authentication = HTTPBasicAuth()


# -------------------------------------------------------------------------------------------------
@authentication.verify_password
def verify_credentials(app_user, password):
    _log.info(f'Checking login credentials for app user "{app_user}"')
    _log.debug(f'Password associated with app user "{app_user}" in login attempt: {password}')

    if not app_user in app_users.keys():
        _log.warning(f'Non-existing app user "{app_user}" tried to authenticate')
        return

    if not password == app_users[app_user]:
        _log.warning(f'App user "{app_user}" tried to authenticate with invalid password')
        return

    _log.debug(f'Successfully authenticated app user "{app_user}"')
    return app_user


# -------------------------------------------------------------------------------------------------
@app.route('/is_the_server_up_and_running', methods=['GET', 'HEAD'])
def return_health_status():
    return f'Yes, the web application does indeed seem to be up and running!'


# -------------------------------------------------------------------------------------------------
@app.get('/api/threads')
@authentication.login_required
def list_threads():
    # ---------------------------------------------------------------------------------------------
    app_user = authentication.current_user()
    _log.info(f'Fetching list of threads for user "{app_user}"')
   
    response = []
    try:
        with postgresql_client.cursor() as cursor:
            cursor.execute("""
                SELECT t.id, t.title, t.author, COALESCE(
                    (SELECT MAX(responded) FROM responses WHERE thread_id = t.id),
                    t.created
                ) AS updated
                FROM threads t
                ORDER BY updated DESC;
            """)
            threads = cursor.fetchall()

            response = [
                {
                    'id': thread[0],
                    'title': thread[1],
                    'author': thread[2],
                    'updated': thread[3].strftime("%Y-%m-%d %H:%M:%S") if thread[3] is not None else None
                }
                for thread in threads
            ]

    except Exception as error_message:
        postgresql_client.rollback()
        raise Exception(
            f'Failed fetching thread list for user "{app_user}" from database: "{error_message}"')

    _log.debug('Generated response data for thread listing: ' + repr(response))

    return jsonify(response)


# -------------------------------------------------------------------------------------------------
@app.post('/api/threads')
@authentication.login_required
def create_thread():
    # ---------------------------------------------------------------------------------------------
    app_user = authentication.current_user()
    _log.info(f'Creating new thread for user "{app_user}"')

    _log.debug('Trying to parse request body data as JSON')
    request_data = request.json
    _log.debug('Parsed request data: ' + repr(request_data))

    log_suffix = f' from user "{app_user}": ' + repr(request_data)

    if not isinstance(request_data, dict):
        _log.warning('Could not parse request data as dictionary' + log_suffix)
        return 'Invalid format of request body', 400

    for required_key in ['title', 'topic_description']:
        if not required_key in request_data.keys():
            _log.warning(f'Could not find key "{required_key}" in request data' + log_suffix)
            return 'Missing key in request body', 400
       
        if not isinstance(request_data[required_key], str):
            _log.warning(f'Key "{required_key}" in request data must be a string' + log_suffix)
            return 'Invalid key type in request body', 400

        if not request_data[required_key]:
            _log.warning(f'Key "{required_key}" is an empty string in request data' + log_suffix)
            return 'Invalid value for key in request body', 400

    title = request_data['title']
    topic_description = request_data['topic_description']
    _log.info(f'Creating thread "{title}" for "{app_user}" with description "{topic_description}"')
   
    # ---------------------------------------------------------------------------------------------
    author_ip = request.remote_addr
    try:
        with postgresql_client.cursor() as cursor:
            cursor.execute("""
                INSERT INTO threads (title, topic_description, author, created, author_ip)
                VALUES (%s, %s, %s, NOW(), %s)
                RETURNING id;
            """, (title, topic_description, app_user, author_ip))

            new_thread_id = cursor.fetchone()[0]
            postgresql_client.commit()

        response = new_thread_id
   
    except Exception as error_message:
        postgresql_client.rollback()
        raise Exception(
            f'Failed creating thread for user "{app_user}": "{error_message}"')

    _log.debug('Generated response data for thread creation: ' + repr(response))

    return jsonify(response)


# -------------------------------------------------------------------------------------------------
@app.get('/api/threads/<int:thread_id>')
@authentication.login_required
def get_thread(thread_id):
    # ---------------------------------------------------------------------------------------------
    app_user = authentication.current_user()
    _log.info(f'Fetching content of thread ID {thread_id} for user "{app_user}"')

    try:
        with postgresql_client.cursor() as cursor:
            cursor.execute("""
                SELECT title, topic_description, created, author
                FROM threads
                WHERE id = %s;
            """, (thread_id,))
            thread_data = cursor.fetchone()

            if not thread_data:
                return jsonify({"error": "Thread not found"}), 404

            cursor.execute("""
                SELECT comment, author, responded
                FROM responses
                WHERE thread_id = %s
                ORDER BY responded ASC;
            """, (thread_id,))
            responses_data = cursor.fetchall()

            
            responses = [{
                "comment": r[0],
                "author": r[1],
                "responded": r[2].strftime("%Y-%m-%d %H:%M:%S")
            } for r in responses_data]

            formatted_created = thread_data[2].strftime("%Y-%m-%d %H:%M:%S")
            response = {
                'title': thread_data[0],
                'topic_description': thread_data[1],
                'created': formatted_created,
                'author': thread_data[3],
                'responses': responses
            }

   
    except Exception as error_message:
        postgresql_client.rollback()
        raise Exception(f'Failed to fetch thread content for user "{app_user}": "{error_message}"')

    _log.debug('Generated response data for thread content: ' + repr(response))

    return jsonify(response)


# -------------------------------------------------------------------------------------------------
@app.put('/api/threads/<int:thread_id>')
@authentication.login_required
def answer_thread(thread_id):
    # ---------------------------------------------------------------------------------------------
    app_user = authentication.current_user()
    _log.info(f'Responding to thread ID {thread_id} for user "{app_user}"')

    _log.debug('Trying to parse request body data as JSON')
    request_data = request.json
    _log.debug('Parsed request data: ' + repr(request_data))

    log_suffix = f' from user "{app_user}": ' + repr(request_data)

    if not isinstance(request_data, str):
        _log.warning('Could not parse request data as a string' + log_suffix)
        return 'Invalid format of request body', 400

    if not request_data:
        _log.warning('Request body contains an empty string' + log_suffix)
        return 'Invalid value for request body', 400

    comment = request_data
    _log.info(f'Responding to thread ID {thread_id} for "{app_user}" with comment "{comment}"')
   
    # ---------------------------------------------------------------------------------------------
    author_ip = request.remote_addr
    try:
        with postgresql_client.cursor() as cursor:
            cursor.execute("""
                INSERT INTO responses (thread_id, comment, author, responded, author_ip)
                VALUES (%s, %s, %s, NOW(), %s);
            """, (thread_id, comment, app_user, author_ip))
            postgresql_client.commit()

    except Exception as error_message:
        postgresql_client.rollback()
        raise Exception(f'Failed to respond to thread for user "{app_user}": "{error_message}"')

    return jsonify('OK')


# -------------------------------------------------------------------------------------------------
@app.delete('/api/threads/<int:thread_id>')
@authentication.login_required
def delete_thread(thread_id):
    app_user = authentication.current_user()
    author_ip = request.remote_addr
    _log.info(f'Deleting thread ID {thread_id} for user "{app_user}"')

    try:
        with postgresql_client.cursor() as cursor:
            
            cursor.execute("SELECT author, author_ip FROM threads WHERE id = %s", (thread_id,))
            thread_data = cursor.fetchone()
            if thread_data is None:
                raise Exception(f"Thread with ID {thread_id} not found")
            thread_author, stored_author_ip = thread_data
            if thread_author != app_user or str(stored_author_ip) != author_ip:
                raise Exception("Unauthorized: You cannot delete another user's thread or a thread created from another IP")

            
            cursor.execute("""
                INSERT INTO deleted_items (item_type, original_id, thread_id, title, content, author, author_ip, deleted_at)
                SELECT 'thread', id, id, title, topic_description, author, author_ip, NOW() 
                FROM threads
                WHERE id = %s
            """, (thread_id,))
            
            
            cursor.execute("""
                INSERT INTO deleted_items (item_type, original_id, thread_id, content, author, author_ip, deleted_at)
                SELECT 'response', id, %s, comment, author, author_ip, NOW() -- Include thread_id
                FROM responses
                WHERE thread_id = %s
            """, (thread_id, thread_id))

            
            cursor.execute("DELETE FROM threads WHERE id = %s;", (thread_id,))

            postgresql_client.commit()
    except Exception as error_message:
        postgresql_client.rollback()
        raise Exception(f'Failed to delete thread ID {thread_id} for user "{app_user}": "{error_message}"')

    return jsonify('OK')


# -------------------------------------------------------------------------------------------------
@app.get('/api/top_posters')
@authentication.login_required
def list_top_posters():
    # ---------------------------------------------------------------------------------------------
    app_user = authentication.current_user()
    _log.info(f'Requesting list of most active forum members for user "{app_user}"')

    try:
        with postgresql_client.cursor() as cursor:
            cursor.execute("""
                SELECT author, COUNT(*) AS total_posts
                FROM (
                    SELECT author FROM threads
                    UNION ALL
                    SELECT author FROM responses
                ) AS all_posts
                GROUP BY author
                ORDER BY total_posts DESC
                LIMIT 3;
            """)
            top_posters = cursor.fetchall()

            response = [poster[0] for poster in top_posters]
   
    except Exception as error_message:
        postgresql_client.rollback()
        raise Exception(f'Failed to fetch top posters for user "{app_user}": "{error_message}"')
   
    _log.debug('Generated response data for top forum posters: ' + repr(response))

    return jsonify(response)


# -------------------------------------------------------------------------------------------------
@app.get('/thread/<int:thread_id>')
@authentication.login_required
def get_thread_html(thread_id):
    _log.info(f'Return thread HTML page for user "{authentication.current_user()}" ({thread_id})')
    return app.send_static_file('thread.html')

# -------------------------------------------------------------------------------------------------
@app.get('/')
@authentication.login_required
def get_thread_list_html():
    _log.info(f'Return thread list HTML page for user "{authentication.current_user()}"')
    return app.send_static_file('thread_list.html')
```
