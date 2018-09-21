# Installation Guide for taiga.io on uberspace.de #

## Install and upgrade PostgreSQL ##

Install PostgreSQL:

    $ uberspace-setup-postgresql

< reconnect ssh >

Disable PostgreSQL:

    $ svc -d ~/service/postgresql

Install new PostgreSQL:

    $ toast arm https://ftp.postgresql.org/pub/source/v9.6.2/postgresql-9.6.2.tar.gz

Edit run-script:

    $ vi ~/service/postgresql/run

Need to replace **rama** with actual user.
File `run` content:

    #!/bin/sh -e
    export USER=rama
    export HOME=/home/rama
    . $HOME/etc/postgresversion
    #export PATH=/package/host/localhost/postgresql-${POSTGRESVERSION}/bin:$PATH
    #export LD_LIBRARY_PATH=/package/host/localhost/postgresql-${POSTGRESVERSION}/lib/:$LD_LIBRARY_PATH
    export PATH=$HOME/.toast/armed/bin/:$PATH
    export LD_LIBRARY_PATH=$HOME/.toast/armed/lib/:$LD_LIBRARY_PATH
    exec postgres -D $HOME/postgresql 2>&1 

Change version in file:

    $ echo "POSTGRESVERSION=9.6" > ~/etc/postgresversion

Fix one parameter in file (string №69):

    $ vi ~/postgresql/postgresql.conf
    
old parameter:

    unix_socket_directory =
    
fixed parameter:

    unix_socket_directories =
  
Backup config:

    $ cp ~/postgresql/postgresql.conf ~/postgresql.conf.bak
  
Clean old files:

    $ rm -rf ~/postgresql
    $ cat ~/.pgpass | tail -1 | sed -E 's/.*:(.*)/\1/' > ~/pg_password.tmp
    $ initdb --pwfile="${HOME}/pg_password.tmp" --auth=md5 -E UTF8 -D ~/postgresql
    $ rm ~/pg_password.tmp
  
Restore config:
    
    $ mv ~/postgresql.conf.bak ~/postgresql/postgresql.conf

Enable PostgreSQL:

    $ svc -u ~/service/postgresql

Configure Python Environment:
  
    $ pip3.4 install virtualenvwrapper --user
  
Edit bash config:
    
    $ vi ~/.bash_profile
    
Old string:

    PATH=$PATH:$HOME/bin

New string:

    PATH=$PATH:$HOME/bin:$HOME/.local/bin/
  
Create and activate virtual environment for Python:
  
    $ source ~/.bash_profile    
    $ source ~/.local/bin/virtualenvwrapper_lazy.sh
    $ mkvirtualenv -p /usr/local/bin/python3.4 taiga
    $ source ~/.virtualenvs/taiga/bin/activate

## Backend installation and configuration ##

Clone repo with source code:

    $ cd ~
    $ git clone https://github.com/taigaio/taiga-back.git taiga-back
    $ cd taiga-back
    $ git checkout stable

Install Python-dependencies:

    $ pip3.4 install -r requirements.txt
    $ export PYTHONPATH=$HOME/.virtualenvs/taiga/lib/python3.4/site-packages

Create user and database:

    $ createuser taiga
    $ createdb taiga -O taiga

Data migration:

    $ python manage.py migrate --noinput
    $ python manage.py loaddata initial_user
    $ python manage.py loaddata initial_project_templates
    $ python manage.py compilemessages
    $ python manage.py collectstatic --noinput

Edit backend-configuration:

    $ vi ~/taiga-back/settings/local.py

Need to replace **rama.elnath** with actual string.
File `local.py` content:

    from .common import *

    MEDIA_URL = "https://rama.elnath.uberspace.de/media/"
    STATIC_URL = "https://rama.elnath.uberspace.de/static/"
    SITES["front"]["scheme"] = "https"
    SITES["front"]["domain"] = "rama.elnath.uberspace.de"

    SECRET_KEY = "theveryultratopsecretkey"

    DEBUG = False
    PUBLIC_REGISTER_ENABLED = True

    DEFAULT_FROM_EMAIL = "no-reply@example.com"
    SERVER_EMAIL = DEFAULT_FROM_EMAIL

    #CELERY_ENABLED = True

    EVENTS_PUSH_BACKEND = "taiga.events.backends.rabbitmq.EventsPushBackend"
    EVENTS_PUSH_BACKEND_OPTIONS = {"url": "amqp://taiga:PASSWORD_FOR_EVENTS@localhost:5672/taiga"}

    # Uncomment and populate with proper connection parameters
    # for enable email sending. EMAIL_HOST_USER should end by @domain.tld
    #EMAIL_BACKEND = "django.core.mail.backends.smtp.EmailBackend"
    #EMAIL_USE_TLS = False
    #EMAIL_HOST = "localhost"
    #EMAIL_HOST_USER = ""
    #EMAIL_HOST_PASSWORD = ""
    #EMAIL_PORT = 25

    # Uncomment and populate with proper connection parameters
    # for enable github login/singin.
    #GITHUB_API_CLIENT_ID = "yourgithubclientid"
    #GITHUB_API_CLIENT_SECRET = "yourgithubclientsecret"
  
Search free port for backend: 
  
    TAIGAPORT=$(( $RANDOM % 4535 + 61000)); netstat -tulpen | grep $TAIGAPORT && echo "versuch's nochmal"

Show actual port for backend. Remember this **TAIGAPORT**:

    $ echo $TAIGAPORT

Test backend:
    
    $ python manage.py runserver $TAIGAPORT

## Frontend installation and configuration ##

Download source code from repo:

    $ cd ~
    $ git clone https://github.com/taigaio/taiga-front-dist.git taiga-front-dist
    $ cd taiga-front-dist
    $ git checkout stable

Copy example config to production config:

    $ cp ~/taiga-front-dist/dist/conf.example.json ~/taiga-front-dist/dist/conf.json
  
Edit config:

    $ vi ~/taiga-front-dist/dist/conf.json

Replace **rama.elnath** with actual string.
File `conf.json` content:

    {
        "api": "https://rama.elnath.uberspace.de/api/v1/",
        "eventsUrl": null,
        "eventsMaxMissedHeartbeats": 5,
        "eventsHeartbeatIntervalTime": 60000,
        "eventsReconnectTryInterval": 10000,
        "debug": true,
        "debugInfo": false,
        "defaultLanguage": "en",
        "themes": ["taiga"],
        "defaultTheme": "taiga",
        "publicRegisterEnabled": true,
        "feedbackEnabled": true,
        "supportUrl": "https://tree.taiga.io/support",
        "privacyPolicyUrl": null,
        "termsOfServiceUrl": null,
        "GDPRUrl": null,
        "maxUploadFileSize": null,
        "contribPlugins": [],
        "tribeHost": null,
        "importers": [],
        "gravatar": true
    }

Copy front-ends source to html-directory:

    $ cp -r ~/taiga-front-dist/dist/* ~/html/

Edit .htaccess:

    $ vi ~/html/.htaccess

Need to replace **TAIGAPORT** with actual port from early step.   
Need to replace **rama.elnath** with actual string.
File `.htaccess` content:
  
    RewriteEngine On

    # redirect to HTTPS
    RewriteCond %{SERVER_PORT} 80
    RewriteRule ^(.*)$ https://rama.elnath.uberspace.de/$1 [R,L]

    # api requests
    RewriteCond %{REQUEST_URI} ^/api
    RewriteRule ^(.*)$ http://127.0.0.1:TAIGAPORT/$1 [P]
    RequestHeader set X-Forwarded-Proto https env=HTTPS

    # deliver files, if not a file deliver index.html
    RewriteCond %{REQUEST_FILENAME} !-f
    RewriteCond %{REQUEST_URI} !^/static/
    RewriteCond %{REQUEST_URI} !^/media/
    RewriteRule ^ index.html [L]

Create symlinks for back-end folders.
Need to replace user **rama** with actual user.

    $ ln -s /home/rama/taiga-back/static/ /home/rama/html/static
    $ ln -s /home/rama/taiga-back/media/ /home/rama/html/media

    $ chmod o+x ~/ ~/taiga-back/ ~/taiga-back/media/ ~/taiga-back/static ~/html/media ~/html/static

    $ chmod 777 -R ~/html/media/
    $ chmod 777 -R ~/html/static/

    $ test -d ~/service || uberspace-setup-svscan
    
Create service for backend. 
Need to replace **TAIGAPORT** with actual port from early step.  
    
    $ uberspace-setup-service taiga gunicorn -w 3 -t 60 --chdir /home/$USER/taiga-back --error-logfile - --pythonpath=. -b 0.0.0.0:TAIGAPORT taiga.wsgi

Enable backend service: 

    $ svc -u ~/service/taiga
