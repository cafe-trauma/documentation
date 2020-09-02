# Table of Contents


# Requirements
1. Python (Django)
2. NPM (Angular)
3. Java (or OpenJDK)
4. Tomcat
5. Nginx
6. Rdf4j
7. An account with sudo permissions

# Firewall

You may need to use a proxy for git. In our case, our server was provided by UF Health IT, and they provided a proxy to use. If you are deploying this to your local machine, you will likely not need to do this.

Because I am using the `/opt/` directory, which requirs root access, I also added this proxy for the git config of the root user.

`sudo git config --global <proxy>`

# File organization

When deploying locally, the locations of your downloads do not matter much. On the actual server, I cloned the repositories to `/opt/git/` (I created the `git` directory). You can clone them to a different location, but this will affect the nginx config later on, so be sure to update it accordingly.

# Back end: cafe-server

The back end is handled by the [cafe-server](https://github.com/cafe-trauma/cafe-server) repository.

`git clone https://github.com/cafe-trauma/cafe-server.git`

You will need to make some changes to `cafe/cafe/settings.py`. If you are on the production server, be sure `DEBUG` is set to `False`. The domain of the server should be added to the `ALLOWED_HOSTS` array. The account information for the postgres user is also stored in this file under `DATABASES`. While you can use the default user (cafeuser) and database name (cafe), you should change the password.

The settings file also reads from a config file. The config file should be named email_config.cfg and contain a JSON object with `SMTP_HOST_USER` and `SMTP_HOST_PASSWORD`, which correlate to an account that can access the UF SMTP server. There is an example in the repository that can be used as a template.

Optional: Create virtual environment with `python -m venv venv` and connect with `source venv/bin/activate`. To close the virtual environment, use `deactivate`.

`pip3 install -r requirements.txt`

You will need to create the postgres database before finishing the last two steps.

`python manage.py migrate`

**Do not create more than one superuser until you have loaded the data into the postgres database (instructions below)**. This will cause issues with the key in the postgres table for users.

`./manage createsuperuser`

Make sure to remember the information for the superuser. This is how you can log into the Django admin page.

`./manage runserver`

# Postgres

If you are using macOS, you can look up a guide for installing postgres. One option is using homebrew: `brew install postgresql`. You can then start it with `pg_ctl -D /usr/local/var/postgres start` and then accessed with `psql postgres`.

If you are using Debian, the following steps initialize the database and start (and set to autostart) the postgresql-12 service

`sudo /usr/pgsql-12/bin/postgresql-12-setup initdb`

`sudo systemctl enable --now postgresql-12`

You can check the status of postgres with `systemctl status postgresql-12.service`.

To connect, first switch to the postgres user with `sudo su - postgres` and then connect with `psql`.

You will need to make the user described in the databases section of `cafe-server/cafe/cafe/settings.py`.

You can edit the settings.py file to choose the password for the cafeuser. Make sure to use the same password in the settings file as you do when creating the cafeuser. It is advisable to change the password listed in the settings file rather than using the default.

`CREATE USER cafeuser WITH PASSWORD '<password>' CREATEDB;`

You will need to upload the data later so your user needs the pg_read_server_files role.

`GRANT pg_read_server_files TO cafeuser;`

Next you will need to create the "cafe" database (or a differently named database but it needs to match `name` in the database section of `settings.py`)

`CREATE DATABASE cafe WITH OWNER cafeuser;`

`CREATE TABLE auth_group;`

(If you are setting up the database on macOS, you do not need to worry about the next paragraph)

For cafe-server to be able to log into the database, you will need to edit `pg_hba.conf`. In order to find the file, you can use `sudo find / -name "pg_hba.conf" -print`. I found it at `/var/lib/pgsql/12/data`. As this folder is owned by the postgres user, I needed to switch to that user with `sudo -su postgres`. Next, edit the IPv4 local connections to use `md5` as their method, instead of `ident`. If you will want to log in to the Postgres server locally, you will also need to change the method for local type from `peer` to `md5` as well. If postgres was running, you will need to restart it after making these changes with `sudo systemctl restart postgresql-12.service`.

To connect to the database, use the following command (assuming your user is `cafeuser` and your database is `cafe`):

`psql -U cafeuser -d cafe`

If you are on the Debian server and have not switched to the postgres user, use:

`sudo -u postgres psql -U cafeuser -d cafe`

## Loading data into Postgres

The tables have already been copied from the Heroku site and are located in the BMI share drive in the folder `HOBI/BMI/CAFE/psql_dumps`. If you need to copy the tables yourself, you can use the following command:

`\COPY <table_name> TO <path/to/save_location.csv> WITH (FORMAT csv, DELIMITER ',', HEADER true);`

When uploading the user-related tables auth_user and authtoken_token, you will need to remove the row with the user id (`id` in auth_user and `user_id` in authtoken_token) **1** in these csvs. That is replaced by the initial superuser you created.

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

`SELECT SETVAL ((SELECT pg_get_serial_sequence('<table_name>', 'id')), <start_number>, true);`

Every table uses the name 'id' for its primary key column, so that will never need to be changed. The 'true' flag is for the is_called flag. It means the start_number has already been used, so, if true, the table should start counting at start_number + 1. You will need to find the start number by checking the id_column of the tables. Unfortunately, you can not just do a count of the rows. Some tables have skipped id numbers (presumably from previously deleted rows). My recommendation:

`SELECT id FROM <table_name> ORDER BY id DESC limit 1;`

In my case, I added 1 to the final id number and used the false is_called flag. You can also just use the final id number and set that flag to true.

# Front end: cafe-app

The front end is handled by the [cafe-app](https://github.com/cafe-trauma/cafe-app) respoitory.

`git clone https://github.com/cafe-trauma/cafe-app.git`

# Tomcat

Download [Tomcat core](https://tomcat.apache.org/download-80.cgi) or have it installed on the server by IT. If Tomcat is installed for you by IT, you will not need to do the rest of the setup steps in this section. I followed the instructions [here](https://wolfpaulus.com/tomcat/), which are for macOS Catalina.

Move the unarchived distribution

`sudo mkdir -p /usr/local`

`sudo mv ~/Downloads/apache-tomcat-9.0.30 /usr/local`

Create a symlink. You may need to first remove an old symlink (if you have no old symlink, this command will do nothing so you can run it freely).

`sudo rm -f /Library/Tomcat`

`sudo ln -s /usr/local/apache-tomcat-9.0.30 /Library/Tomcat`

Change ownership of the folder

`sudo chown -R <your_username> /Library/Tomcat`

Make the scripts executable

`sudo chmod +x /Library/Tomcat/bin/*.sh`

## Finding the Tomcat directory

When installing on your local machine, if you followed the above instructions, you will find the Tomcat directory at `/Library/Tomcat`. However, if IT has installed Tomcat, this may not be the case. You will need to find where $CATALINA_HOME points to. To do this, you can use `ps aux | grep catalina`. In my case, it was at `usr/share/tomcat`

## Starting and Ssopping Tomcat

To start and stop Tomcat locally, use `$CATALINA_HOME/bin/startup.sh` and `$CATALINA_HOME/bin/shutdown.sh` respectively.

On an IT-provided server, use `sudo tomcat start` and `sudo tomcat stop`. If this does not work, contact an IT representetive. It is important that Tomcat be a service for configuring on-boot setup later.

# Nginx

When deploying locally, you need not use Nginx. The downside is that images will not work on your local deployment, but the website should otherwise be fully functional.

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
       proxy_pass http://localhost:8000/static/;
   }

   location /graphs {
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

The following is a helpful edition to the nginx config file for the **dev** server (they should be added within the server section, before the final closing bracket `}`):

**Do not add these to the prod server.**

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

# RDF4J

Download [rdf4j sdk](https://rdf4j.org/download/)

On the server, I created the directory `/opt/rdf4j` to place the extracted rdf4j folder. Locally, feel free to place this anywhere.

You will need to move the server and workbench files to the Tomcat webapps directory. The location on our server is `/usr/share/tomcat/webapps`. As mentioned above, this is the `$CATALINA_HOME` directory found by running `ps aux | grep catalina`. The server and workbench are war files that Tomcat will unzip. You do not need to unzip them yourself. If they do not unzip automaticaaly, then they are not in the correct folder.

## Limiting access to the triplestore

On a local instance, this is not necessary. However, when you put RDF4J on a server (prod or dev), you should limit access to it. Before editing the files mentioned, make sure Tomcat is off.

## Connect to RDF4J console

Tomcat must be running to use RDF4J.

Within the RDF4J folder, navigate through the extracted eclipse directory to the `bin` directory, which should contain a `console.bat` and `console.sh`. Either can be used to connect to the console.

`/opt/rdf4j/eclipse-rdf4j-3.0.4/bin/console.sh`

Connect to your server.

`connect http://localhost:8080/rdf4j-server`

You can show the various repositories by typing `show repositories` or `show r`, and you can connect to one with `connect <repository>` or connect to the default one using `connect default`. Use `disconnect` to disconnect.

The repositories used by the server belong to Tomcat. Locally, I found it easier to connect to rdf4j-workbench through my browser at `http://localhost:8080/rdf4j-workbench` and create the repository there. I created a `native` respository.

Note: `8080` is the default port for Tomcat. If you have changed Tomcat's port, then that is what you will need to connect to.

On the server, you will need to find the repository directory belonging to Tomcat. You can find them using `sudo find / -name 'repositories'` to find all possible `repositories` directories and then selecting the one that is a subdirectory of a `tomcat/.RDF4J` directory. I found the folder at `/usr/share/tomcat/.RDF4J/server/`.

You will also need to log in to the console at tomcat in order to have permission to access this directory.

`sudo -u tomcat /opt/rdf4j/eclipse-rdf4j-3.0.4/bin/console.sh`

You may get a message saying `Illegal value for property workdir`. You can ignore this. Next, connect to the correct directory.

`connect /usr/share/tomcat/.RDF4J/server/`

## Create a triple repository

After connecting:

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

If you have connected to the workbench through your browser, the sidebar should have an option called `New repository` under `Repositories` on the left.

## Uploading triples

You can verify a file using `verify path/to/file.rdf`

You can load a file using `load path/to/file.rdf`

You will need to download the owl file of the ontology from [here](https://github.com/OOSTT/OOSTT/blob/master/oostt.owl). You can then use the `load` command to add it to the triplestore. After making changes to the repository via the console, you must restart Tomcat.

I used the api on the heroku site to curl the RDF files for the organizations. These files are saved to the share drive. If you need to pull down more triples, you can modify the endpoint that you are requesting from. The example below goes to the heroku site and is pulling down RDF for organization 34. The token used is the token for a user authorized for the given organization. You can look through the Postgres database to find the desired token.

```
curl --request GET \
  --url https://cafe-trauma.herokuapp.com/api/rdf/34 \
  --header 'authorization: Token <token>'
```

I have placed the rdf files for these approved organizations on the BMI share drive under `/HOBI/BMI/CAFE/org_triples`.

If you have connected to the workbench through your browser, check if you are connected to the correct repository by looking at the top right corner and checking what `Repository:` says. If you are connected to the wrong repository or it says `- none -`, click `Respoitories` in the left sidebar and then click on the repository you want. Then go to the `Add` option under `Modify` and upload the files from there (just choose the correct file and then click `Upload`).

# Creating service files

This section does not apply to a local instance, where you will simply run django and angular in open tabs of your terminal.

If the server is restarted for whatever reason, we would like all the website services to boot automatically rather than having to be manually started. We can use system services for this. The postgres service file was already enabled to start on boot-up. Nginx and Tomcat are also already service files, so they need only be enabled to start on bootup.

`sudo systemctl enable nginx.service`

`sudo systemctl enable tomcat`

For the Django and Angular repositories, you will need to create service files in the directory `/etc/systemd/`.

`cafe-django.service`
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

`cafe-angular.service`
```
[Unit]
Description=Angular front end for cafe questionnaire application
After=network-online.target

[Service]
WorkingDirectory=/opt/git/cafe-app/
ExecStart=/usr/lib/node_modules/@angular/cli/bin/ng serve --proxy-config proxy.dev.json

[Install]
WantedBy=multi-user.target
```

You can start the service with `sudo systemctl start cafe-django` and stop with `sudo systemctl stop cafe-django`. You can check the status with `sudo systemctl status cafe-django`.

You will need to enable both with `sudo systemctl enable cafe-django` (and `sudo systemctl enable cafe-angular`).

In order to check what services are enabled, you can use `systemctl list-unit-files | grep enabled`.

# File Permissions

Ultimately, file permissions are up to your discretion. UF Health IT requires you to go through them to add a user, so I asked for a `cafe` user. I then created a `baristas` group.

`sudo groupadd baristas`

I then added all developer accounts and the `cafe` user to the `baristas` group.

`sudo usermod -a -G baristas <user>`

Next, change the ownership of the desired files. For example, I changed the ownership of the `/opt/git` directory. You will also need to change permissions on the file so that group members can edit.

`sudo chown -R cafe /opt/git && sudo chgrp -R baristas /opt/git`

`sudo chmod 775 /opt/git`