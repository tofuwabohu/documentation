Title: Installing in Production
Date: 2021-04-12
Order: 1

This project is still young and isn't, at the moment, very stable, so please proceed with caution when running in production.

## Server setup
- Get a domain name and set up DNS for your server
- Set your server up with appropriate firewalls for running a web application (this instruction set is tested against Ubuntu 20.04)
- Set up an email service (such as mailgun) and the appropriate SMTP/DNS settings
- Install Docker and docker-compose

## Install and configure BookWyrm

The `production` branch of BookWyrm contains a number of tools not on the `main` branch that are suited for running in production, such as `docker-compose` changes to update the default commands or configuration of containers, and individual changes to container config to enable things like SSL or regular backups.

Instructions for running BookWyrm in production:

- Get the application code:
    `git clone git@github.com:mouse-reeve/bookwyrm.git`
- Switch to the `production` branch:
    `git checkout production`
- Create your environment variables file, `cp .env.prod.example .env`, and update the following:
    - `SECRET_KEY` | A difficult to guess, secret string of characers
    - `DOMAIN` | Your web domain
    - `EMAIL` | Email address to be used for certbot domain verification
    - `POSTGRES_PASSWORD` | Set a secure password for the database
    - `REDIS_ACTIVITY_PASSWORD` | Set a secure password for Redis Activity subsystem
    - `REDIS_BROKER_PASSWORD` | Set a secure password for Redis queue broker subsystem
    - `FLOWER_USER` | Your own username for accessing Flower queue monitor
    - `FLOWER_PASSWORD` | Your own secure password for accessing Flower queue monitor
- Update your nginx configuration in `nginx/default.conf`
    - Replace `your-domain.com` with your domain name
- Configure nginx
    - Make a copy of the production template config and set it for use in nginx `cp nginx/production nginx/default.conf`
    - Update `nginx/default.conf`:
        - Replace `your-domain.com` with your domain name
        - If you aren't using the `www` subdomain, remove the www.your-domain.com version of the domain from the `server_name` in the first server block in `nginx/default.conf` and remove the `-d www.${DOMAIN}` flag at the end of the `certbot` command in `docker-compose.yml`.
        - If you are running another web-server on your host machine, you will need to follow the [reverse-proxy instructions](#running-bookwyrm-behind-a-reverse-proxy)
- If you need to initialize your certbot for your domain, set `CERTBOT_INIT=true` in your `.env` file
- Run the application (this should also set up a Certbot ssl cert for your domain) with
    `docker-compose up --build`, and make sure all the images build successfully
    - If you are running other services on your host machine, you may run into errors where services fail when attempting to bind to a port.
    See the [troubleshooting guide](#port-conflicts) for advice on resolving this.
- When docker has built successfully, stop the process with `CTRL-C`
- If you set `CERTBOT_INIT=true` earlier, set it now as `CERTBOT_INIT=false` so that certbot runs in renew mode
- Run docker-compose in the background with: `docker-compose up -d`
- Initialize the database with: `./bw-dev initdb`
- Set up schedule backups with cron that runs that `docker-compose exec db pg_dump -U <databasename>` and saves the backup to a safe location

Congrats! You did it, go to your domain and enjoy the fruits of your labors.

## Configure your instance
- Register a user account in the application UI
- Make your account a superuser (warning: do *not* use django's `createsuperuser` command)
    - On your server, open the django shell
    `./bw-dev shell`
    - Load your user and make it a superuser:
```
:::python linenums=False
from bookwyrm import models
user = models.User.objects.get(id=1)
user.is_staff = True
user.is_superuser = True
user.save()
```

- Go to the site settings (`/settings/site-settings` on your domain) and configure your instance name, description, code of conduct, and toggle whether registration is open on your instance

## Backups

BookWyrm's db service dumps a backup copy of its database to its `/backups` directory daily at midnight UTC.
Backups are named `backup__%Y-%m-%d.sql`.

The db service has an optional script for periodically pruning the backups directory so that all recent daily backups are kept, but for older backups, only weekly or monthly backups are kept.
To enable this script:

- Uncomment the final line in `postgres-docker/cronfile`
- rebuild your instance `docker-compose up --build`

You can copy backups from the backups volume to your host machine with `docker cp`:

- Run `docker-compose ps` to confirm the db service's full name (it's probably `bookwyrm_db_1`.
- Run `docker cp <container_name>:/backups <host machine path>`

## Port Conflicts

BookWyrm has multiple services that run on their default ports.
This means that, depending on what else you are running on your host machine, you may run into errors when building or running BookWyrm when attempts to bind to those ports fail.

If this occurs, you will need to change your configuration to run services on different ports.
This may require one or more changes the following files:

- `docker-compose.yml`
- `nginx/default.conf`
- `.env` (You create this file yourself during setup)

E.g., If you need Redis to run on a different port:

- In `docker-compose.yml`:
    - In `services` -> `redis` -> `command`, add `--port YOUR_PORT` to the command
    - In `services` -> `redis` -> `ports`, change `6379:6379` to your port
- In `.env`, update `REDIS_PORT`

If you are already running a web-server on your machine, you will need to set up a reverse-proxy.
