# Setup Django on Un-Managed VPS

After some struggles to setup my django project on an unmanaged vps I put all the instructions that helped me into easy to follow setup.
replace values surounded by <> with your own values

## Server setup

### Ubuntu setup with root or sudo the below if you dont have root access.

`apt-get update`

`apt upgrade`

`apt-get install nginx`

`apt-get install systemd`

`apt-get install ufw`

`ufw allow 'Nginx Full'`

`ufw status`

### Python setup

`apt-get install python3`

`apt-get install python3-pip`

`pip3 install virtualenv`

### Add new user and give user sudo

`adduser <username>`

`usermod -a -G sudo <username>`

### Login with new user and continue.
### Install and update django

`cd ~`

`mkdir django-apps`

`cd django-apps`

`virtualenv env`

`source env/bin/activate`

`pip install django`

`pip install django gunicorn psycopg2`

`cd ~/django-apps`

`django-admin startproject mysite`

you can now run your development server to test your setup for development 

`~/django-apps/<mysite>/python manage.py runserver <yourIP>:8000`

after your site has been set up and you are ready to go into production in the `settings.py` file of your project change the following:

```
DEBUG = False
...
ALLOWED_HOSTS = [<yourIP>, <yourdomain.com>]
...
STATIC_ROOT = os.path.join(BASE_DIR, 'static')
STATIC_URL = '/static/'
...
MEDIA_ROOT = os.path.join(BASE_DIR, 'media_root')
MEDIA_URL = '/media_root/'
```
run

`python manage.py collectstatic`

this command will collect all the static files of your project into the directories specified in the settings.py file. We will later point our nginx files to these directories to serv up these files. Next run

`deactivate` 

to close out the virtualevn in order to continue the rest of the setup.

### Gunicorn setup

in `/etc/systemd/system/` create a new file called `gunicorn.service` and add the below code to that file and save:

```
[Unit]
Description=gunicorn daemon
After=network.target

[Service]
User=<username>
Group=www-data
WorkingDirectory=/home/<username>/django-apps/mysite
ExecStart=/home/<username>/django-apps/env/bin/python3 /home/<username>/django-apps/env/bin/gunicorn --access-logfile - --workers 3 --bind unix:/home/<username>/django-apps/<mysite>.sock mysite.wsgi:application

[Install]
WantedBy=multi-user.target
```
run the below commands to start and enable gunicorn.

`sudo systemctl start gunicorn`

`sudo systemctl enable gunicorn`

check the status of gunicorn with the below command. If there are issues check that the gunicorn.service file was setup correctly.

`sudo systemctl status gunicorn`

if you fixed any issues with gunicorn remember to run the below commands to reload the deamon and gunicorn

`sudo systemctl daemon-reload`

`sudo systemctl restart gunicorn`

### Nginx setup

go to `/etc/nginx/sites-available/` and create `<mysite>` file and add the below setup

```
server {
    listen 80;
    listen [::]:80;
    server_name <servername>;

    location = /favicon.ico { access_log off; log_not_found off; }
    location /static {
        root /home/<username>/django-apps/<mysite>;
    }

    location /media {
        root /home/<username>/django-apps/<mysite>;
    }


    location / {
        include proxy_params;
        proxy_pass http://unix:/home/<username>/django-apps/<mysite>.sock;
    }
}
```

after creating this file run the below command to add a shortcut of the file to the sites-enabled directory  

`sudo ln -s /etc/nginx/sites-available/<mysite> /etc/nginx/sites-enabled`

nginx and gunicorn should now be fully setup and working. Run 

`sudo nginx -t`

to check your nginx file for syntax errors. Then Run

`sudo systemctl restart nginx`

to restart nginx.

If there were any issues run `sudo tail -30 /var/log/nginx/error.log` to check the error logs.

Congrats your django site should be up and running. 


## Bonus

### setup a SSL certificate using certbot

`sudo apt-get update`

`sudo apt-get install software-properties-common`

`sudo add-apt-repository ppa:certbot/certbot`

`sudo apt-get update`

`sudo apt-get install python-certbot-nginx` 

`sudo certbot --nginx`
