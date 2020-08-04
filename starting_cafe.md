# Table of Contents

- [Requirements](#requirements)
- [Firewall exceptions](#firewall-exceptions)
- [Back-end](#back-end)
- [Front-end](#front-end)
- [Postgres](#postgres)
  * [Setup](#setup)
  * [Loading data to Postgres](#loading-data-to-postgres)
- [Tomcat setup](#tomcat-setup)
  * [Installing locally](#installing-locally)
  * [Finding the Tomcat directory](#finding-the-tomcat-directory)
  * [To add users](#to-add-users)
- [Nginx Set-up](#nginx-set-up)
- [Rdf4j Set-up](#rdf4j-set-up)
  * [Rdf4j console](#rdf4j-console)
  * [Create a triple repository](#create-a-triple-repository)
  * [Uploading triples to the repository](#uplaoding-triples-to-the-repository)
- [File Permissions](#file-permissions)

<small><i><a href='http://ecotrust-canada.github.io/markdown-toc/'>Table of contents generated with markdown-toc</a></i></small>

# Requirements

1. Python (Django)
2. NPM (Angular)
3. Java (or OpenJDK)
4. Tomcat
5. Nginx
6. Rdf4j

# Firewall exceptions

<!-- If you are using UFHealth IT, you will need to make some exceptions for GitHub and for the package managers.

GitHub: github.com, github.io

pip: pypi.io, pypi.org, python.pypi.org, files.pythonhosted.org

npm: registry.npmjs.org -->

If you need to use a proxy for git, you will have to edit the git config. I edited it for the root user since I needed to sudo to clone the repositories in the `/opt/` directory.

`sudo git config --global <proxy>`

# Back-end

Create a folder and clone [cafe-server](https://github.com/cafe-trauma/cafe-server) repository.

You will need to make some changes to `cafe/cafe/settings.py`. If you are on the production server, be sure `DEBUG` is set to `False`. The domain of the server should be added to the `ALLOWED_HOSTS` array. The account information for the postgres user is also stored in this file under `DATABASES`. While you can use the default user (cafeuser) and database name (cafe), you should change the password.

The settings file also reads from a config file. The config file should be named email_config.cfg and contain a JSON objet with `SMTP_HOST_USER` and `SMTP_HOST_PASSWORD`, which correlate to an account that can access the UF SMTP server.

Optional: Create virtual environment

`pip3 install -r requirements.txt`

You will need to create the postgres database before finishing the last two steps.

`python manage.py migrate`

`./manage createsuperuser`

Make sure to remember the information for the superuser. This is how you can log into the Django admin page.

**Do not create more than one superuser until you have loaded the data (instructions below)**. This will cause issues with the key on the users.

`./manage runserver`

# Front-end

Create a folder and clone [cafe-app](https://github.com/cafe-trauma/cafe-app) respoitory.

<!-- You will need to install angular yourself apart from other npm packages.

`npm install -g @angular/cli`

If you have permission issues (you probably will), you will need to run as root, and you may run into an error where the install freezes indefiitely at rollbackFailedOptional. In that case, try the following:

`sudo npm install --proxy <proxy> install @angular/cli -g` -->

`npm install`

You will see warnings that you need to update packages but you must resist. Running `npm audit fix` will break the application. Some of the listed vulnerabilities are misleading. For instance, js-yaml was listed as a vulnerable package. There are two libraries listed in the package-lock.json file that have js-yaml as a dependency. In one, the required version is 3.13.1. In the other, the required version is 3.7.0. The listed 3.7.0 is raising flags, but the actual installed version is the newer 3.13.1 version. To confirm your package version, you can use `npm info <package> version`.

`ng build --prod`

`ng s --proxy-config proxy.dev.json` or `ng s --proxy-config proxy.prod.json` for development or production, respectively

# Postgres 

## Setup

The following steps initialize the database and start (and set to autostart) the postgresql-12 service

`sudo /usr/pgsql-12/bin/postgresql-12-setup initdb`

`sudo systemctl enable --now postgresql-12`

You can check the status of postgres with `systemctl status postgresql-12.service`.

You will need to make the user described in the databases section of `cafe-server/cafe/cafe/settings.py` (the link to this repo can be found in the back-end section of this document). To connect, first switch to the postgres user with `sudo su - postgres` and then connect with `psql`.

You can edit the settings.py file to choose the password for the cafeuser. Make sure to use the same password in the settings file as you do when creating the cafeuser. It is advisable to change the password listed in the settings file rather than using the default.

`CREATE USER cafeuser WITH PASSWORD '<password>' CREATEDB;`

You will need to upload the data later so your user needs the pg_read_server_files role.

`GRANT pg_read_server_files TO cafeuser;`

Next you will need to create the "cafe" database (or a differently named database but it needs to match `name` in the database section of `settings.py`)

`CREATE DATABASE cafe WITH OWNER cafeuser;`

`CREATE TABLE auth_group;`

For cafe-server to be able to log into the database, you will need to edit `pg_hba.conf`. In order to find the file, you can use `sudo find / -name "pg_hba.conf" -print`. I found it at `/var/lib/pgsql/12/data`. As this folder is owned by the postgres user, I needed to switch to that user with `sudo -su postgres`. Next, edit the IPv4 local connections to use md5 as their method, instead of ident. If you will want to log in to the Postgres server locally, you will also need to change the method for local type from peer to md5 as well. If postgres was running, you will need to restart it after making these changes with `sudo systemctl restart postgresql-12.service`.

To connect to the database, use the following command (assuming your user is `cafeuser` and your database is `cafe`):

`psql -U cafeuser -d cafe`

## Loading data to Postgres

<!-- If you have not already done so, open the Django admin page and open the from_question-to_question section and click on the 'Add from-question-to_question relationship' button in the top right, but you do not need to actually add anything. This should fill in the django_content_type table, which should have 16 rows. -->

The tables have already been copied from the Heroku site and are located in the chare drive in the folder `psql_dumps`. If you need to copy the tables yourself, you can use the following command:

`\COPY <table_name> TO <path/to/save_location.csv> WITH (FORMAT csv, DELIMITER ',', HEADER true);`

When uploading these user-related tables, you will need to remove the row with user_id (id in auth_user and user_id in authtoken_token) **1** in these csvs. That is replaced by the initial superuser you created.

Use the following command to upload the CSVs to the tables:

`COPY <table_name> FROM '<path/to/file.csv>' WITH (FORMAT csv, [HEADER true]);`

If you copied the headers when creating the CSVs, you will need to indicate `HEADER true`. Otherwise, you can leave this part out.

When uploading the data to the Postgres database, you will need to upload in a way that referenced foreign keys already exist when a table references them. The following order should avoid conflicts:

1. auth_user
2. authtoken_token
3. questionnaire_organization
4. questionnaire_activeorganization 
5. questionnaire_organization_users
6. questionnaire_category
7. questionnaire_rdfprefix
8. questionnaire_question
9. questionnaire_question_depends_on
10. questionnaire_option 
11. questionnaire_question_options
12. questionnaire_answer
13. questionnaire_answer_options
14. questionnaire_statement

This method is quick for uploading the data, but Django doesn't seem to register the taken primary keys when this method is used so you will need to set the start value for all non-empty tables (which are all the tables above), **except the table authtoken_token**. The format for this is:

`SELECT SETVAL ((SELECT pg_get_serial_sequence('<table_name>', 'id')), <start_number>, false);`

Every table uses the name 'id' for its primary key column, so that will never need to be changed. The 'false' flag is for the is_called flag. If it were set to true, the incrementation would start at start_number + 1. You will need to find the start number by checking the id_column of the tables. Unfortunately, you can not just do a count of the rows. Some tables have skipped id numbers (presumably from previously deleted rows). My recommendation:

`SELECT id FROM <table_name> ORDER BY id DESC limit 1;`

# Tomcat setup

## Installing locally

Download [Tomcat core](https://tomcat.apache.org/download-80.cgi) or have it installed on the server by IT. If Tomcat is installed for you, you will not need to do the rest of the setup steps in this section. It will likely be at `/etc/tomcat/`.

Move the unarchived distribution

`sudo mkdir -p /usr/local`

`sudo mv ~/Downloads/apache-tomcat-9.0.30 /usr/local`

Creat a symlink

`sudo rm -f /Library/Tomcat`

`sudo ln -s /usr/local/apache-tomcat-9.0.30 /Library/Tomcat`

Change ownership of the folder

`sudo chown -R <your_username> /Library/Tomcat`

Make the scripts executable

`sudo chmod +x /Library/Tomcat/bin/*.sh`

## Finding the Tomcat directory

When installing on your local machine, you will usually find the Tomcat directory at `/Library/Tomcat`. However, if IT has installed Tomcat, this may not be the case. You will need to find where $CATALINA_HOME points to. To do this, you can use `ps aux | grep catalina`. In my case, it was at `usr/share/tomcat`

To start and stop Tomcat, use `$CATALINA_HOME/bin/startup.sh` and `$CATALINA_HOME/bin/shutdown.sh` respectively. On an IT-provided server, you may need to use `sudo tomcat start` and `sudo tomcat stop` as well.

## To add users

Edit `$CATALINA_HOME/conf/tomcat-users.xml`

# Nginx Set-up

To install Nginx on a Red Hat system, first create the file `/etc/yum.repos.d/nginx.repo` and fill it in as below:

```
[nginx]
name=nginx repo
baseurl=https://nginx.org/packages/mainline/rhel/7/$basearch/
gpgcheck=0
enabled=1
```

Then run the following commands:

`sudo yum update`

`sudo yum install nginx`

<!-- Nginx on Redhat does not have the sites-available and sites-enabled directories by default so you will need to make them in `/etc/nginx`. Then edit `/etc/nginx/nginx.conf` and add the line `include /etc/nginx/sites-enabled/*;`. -->

You will need a config file with the relevant aliases and proxy_passes. This file should be saved in the `conf.d` directory of the nginx folder. For me, this file is saved as `/etc/nginx/conf.d/nginx_cafe.conf`. Please note that the /images/ section of the config file is dependent on where you saved the cafe-app repository.

```
server {
   listen       80 default_server;
   listen       [::]:80 default_server;
   server_name  _;
   root         /usr/share/nginx/html;

   # Load configuration files for the default server block.
   include /etc/nginx/default.d/*.conf;

   location / {
        proxy_pass http://localhost:4200/;
   }

   location /static {
       # alias /opt/git/cafe-server/cafe/static;
       proxy_pass http://localhost:8000/static/;
   }

   location /graphs {
       # alias /opt/git/cafe-server/cafe/graphs;
       proxy_pass http://localhost:8000/graphs/;
   }

   location /api/ {
       proxy_pass http://localhost:8000/;
   }

   location /admin/ {
       proxy_pass http://localhost:8000/admin/;
   }

   location /rdf {
      proxy_pass http://localhost:8080/rdf4j-server/repositories/cafe;
   }

   location /images {
      alias /opt/git/cafe-app/src/images;
   }

   error_page 404 /404.html;
      location = /40x.html {
   }

   error_page 500 502 503 504 /50x.html;
      location = /50x.html {
   }

}
```

The following is a helpful edition to the nginx config file for the **dev** server:

```
   location /rdf4j-server {
      proxy_pass http://localhost:8080/rdf4j-server;
   }

   location /rdf4j-workbench {
      proxy_pass http://localhost:8080/rdf4j-workbench;
   }
```

Nginx is started with `sudo systemctl start nginx.service`.

Nginx is stopped with `sudo systemctl stop nginx.service`.

# Rdf4j Set-up

Download [rdf4j sdk](https://rdf4j.org/download/)

I created the directory `/opt/rdf4j` to place the extracted rdf4j folder.

Move the server and workbench files to the webapps directory. The location on our server is `/usr/share/tomcat/webapps`. To find the webapps folder, you can use `ps aux | grep catalina`. Alternatively, you can move them to a different folder as long as you update `Tomcat/server.xml` with the location of that folder. The server and workbench are war files that Tomcat will unzip. You do not need to extract them yourself.

The following steps are run after Tomcat has been started.

## Rdf4j console

Within the rdf4j folder, navigate to `bin/`, where there is a `console.bat` and `console.sh`. Either can be used to connect to the console.

Connect to your server.

`connect http://localhost:8080/rdf4j-server`

You can show the various repositories by typing `show repositories` or `show r` and connect to one with `connect <respoitory>` or connect to the default one using `connect default`. Use `disconnect` to disconnect.

The repositories used by the server belong to Tomcat. I found them at `/usr/share/tomcat/.RDF4J/server/`, which then contains the appropriate `respoitories` directory. You can use `sudo find / -name 'repositories'` to find all possible `respositories` directories. You will need to log in to the console as tomcat with `sudo -u tomcat /opt/rdf4j/eclipse-rdf4j-3.0.4/bin/console.sh`. To connect to the correct directory, you may need to use `connect /usr/share/tomcat/.RDF4J/server/`.

## Create a triple repository

```
> create native
Please specify values for the following variables:
Repository ID [native]: cafe
Repository title [Native store]: Cafe Triples Repository
Query Iteration Cache size [10000]:
Triple indexes [spoc,posc]:
EvaluationStrategyFactory [org.eclipse.rdf4j.query.algebra.evaluation.impl.StrictEvaluationStrategyFactory]:
Repository created
```

To connect to your respoitory, use `open <repo id>`. In our case, we will be using `open cafe`.

## Uploading triples to the repository

You can verify a file using `verify path/to/file.rdf`

You can load a file using `load path/to/file.rdf`

You will need to download the owl file of the ontology from [here](https://github.com/OOSTT/OOSTT/blob/master/oostt.owl). You can then use the `load` command to add it to the triplestore. After making changes to the repository via the console, you must restart Tomcat.

I used the api on the heroku site to curl the RDF files for the organizations. These files are saved to the share drive. If you need to pull down more triples, you can modify the endpoint that you are requesting from. The example below goes to the heroku site and is pulling down RDF for organization 34. The token used is the token for a user authorized for the given organization. You can look through the Postgres database to find the desired token.

```
curl --request GET \
  --url https://cafe-trauma.herokuapp.com/api/rdf/34 \
  --header 'authorization: Token <token>'
```

# File Permissions

Ultimately, file permissions are up to your discretion. UF Health IT requires you to go through them to add a user, so I asked for a `cafe` user. I then created a `baristas` group.

`sudo groupadd baristas`

I then added all developer accounts and the `cafe` user to the `baristas` group.

`sudo usermod -a -G baristas $USER`

Next, change the ownership of the desired files. For example, I changed the ownership of the `/opt/git` directory. You will also need to change permissions on the file so that group members can edit.

`sudo chown -R cafe /opt/git && sudo chgrp -R baristas /opt/git`

`sudo chmod 775 /opt/git`

# Gunicorn etc. (this section is a WIP)

Create a systemd file at `etc/systemd/system/cafe-django.service`. WorkingDirectory and ExecStart will depend on where you put the cafe-server repo.

```
[Unit]
Description=Django server for cafe questionnaire application
After=network.target
 
[Service]
WorkingDirectory=/opt/git/cafe-server/cafe/
ExecStart=/opt/git/cafe-server/venv/bin/gunicorn cafe.wsgi --name cafe-server
 
[Install]
WantedBy=multi-user.target
```

`sudo systemctl start cafe-django`
`sudo systemctl enable cafe-django`

Logs can be seen using `sudo journalctl -u cafe-django`.

```
[Unit]
Description=Angular front end for cafe questionnaire application
After=network-online.target

[Service]
WorkingDirectory=/opt/git/cafe-app/
ExecStart=/usr/lib/node_modules/@angular/cli/bin/ng serve --proxy-config proxy.dev.json

[Install]
WantedBy=multi-user.targe
```
