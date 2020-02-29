Talkyard production installation
================

For one single server: Ubuntu 18.04 with at least 2 GB RAM.

Docker based installation. Automatic upgrades.
You can configure HTTPS via LetsEncrypt.
One installation can host many sites.

<!-- NO, Swarm is abandonware
If however you already have a Docker-Compose or Docker Swarm installation
with a HTTPS reverse proxy, and want to add Talkyard to it,
then have a look at: https://github.com/debiki/talkyard-prod-swarm.
-->


You should be familiar with Linux, Bash and Git. Otherwise you might run into
problems. For example, there might be Git edit conflicts, if you and we change
the same file — then you need to know how to resolve those edit conflicts.
Also, knowing a bit about Docker can be good.
See https://www.talkyard.io/plans for alternatives to installing yourself.

Ask questions and report problems in **[the forum](http://www.talkyard.io/forum/latest/support)**.
This is beta software; there might be bugs.

If you'd like to test install on your laptop, there's
[a Vagrantfile here](scripts/Vagrantfile) — open it in a text editor, and read,
for details.

Installation overview: You'll rent a virtual private server (VPS) somewhere, then download
and install Talkyard, then sign up for a send-emails service and configure email settings.
Then optionally configure OpenAuth login for Google, Facebook, Twitter, GitHub.
And off-site backups.

Dockerfiles, build scripts and source code are in another repo: https://github.com/debiki/talkyard.
Have a look in `./docker-compose.yml` (in this repo) for details and links.


Get a server
----------------

Provision an Ubuntu 18.04 server with at least 2 GB RAM, for example at [Digital Ocean](https://www.digitalocean.com/).


Installation instructions
----------------

1. Become root: `sudo -i`, then install Git and English: (can be missing, in minimal Ubuntu builds)

       # As root:
       apt-get update
       apt-get -y install git vim locales
       locale-gen en_US.UTF-8                      # installs English
       export LC_ALL=en_US.UTF-8 LANG=en_US.UTF-8  # starts using English (warnings are harmless)

1. Download installation scripts: (you need to install in
   `/opt/talkyard/` for the backup scripts to work)

       cd /opt/
       git clone https://github.com/debiki/talkyard-prod-one.git talkyard
       cd talkyard

1. Prepare Ubuntu: install tools, enable automatic security updates, simplify troubleshooting,
   and make ElasticSearch work:

       ./scripts/prepare-ubuntu.sh 2>&1 | tee -a talkyard-maint.log

   (If you don't want to run this whole script, you at least need to copy the
   sysctl `net.core.somaxconn` and `vm.max_map_count` settings in the script to your
   `/etc/sysctl.conf` config file — otherwise, the full-text-search-engine (ElasticSearch)
   won't work. Afterwards, run `sysctl --system` to reload the system configuration.)

1. Install Docker:

       ./scripts/install-docker-compose.sh 2>&1 | tee -a talkyard-maint.log

1. Install a firewall, namely *ufw*: (and answer Yes to the question you'll get. You can skip this if
   you use Google Cloud Engine; GCE already has a firewall)

       ./scripts/start-firewall.sh 2>&1 | tee -a talkyard-maint.log

1. Edit config values:

   ```
   nano conf/play-framework.conf  # fill in values in the Required Settings section
   nano .env                      # type a database password
   ```

   Note:
   - If you don't edit `play.http.secret.key` in `play-framework.conf`, the server won't start.
   - A PostgreSQL database user, named *talkyard*, gets created automatically,
     by the *rdb* Docker container, with the password you type in the `.env` file.
     You don't need to do anything.
   - If you're using a non-standard port, say 8080 (which you do if you're using **Vagrant**),
     then comment in `talkyard.port=8080` in `play-framework.conf`.

1. Depending on how much RAM your server has (run `free -mh` to find out), choose one of these files:
   mem/1.7g.yml, mem/2g.yml, mem/3.6g.yml, ... and so on,
   and copy it to ./docker-compose.override.yml. For example, for
   a server with 2 GB RAM:

        cp mem/2g.yml docker-compose.override.yml

1. Install and start the latest version. This might take a few minutes
   the first time (to download Docker images).

        # This script also installs, although named "upgrade–...".
        ./scripts/upgrade-if-needed.sh 2>&1 | tee -a talkyard-maint.log

   (This creates a new Docker network — you can choose the IP range; see the
   section *A New Docker Network* below.)

1. Schedule deletion of old log files, daily backups and deletion old backups,
   and automatic upgrades:

        ./scripts/schedule-logrotate.sh 2>&1 | tee -a talkyard-maint.log
        ./scripts/schedule-daily-backups.sh 2>&1 | tee -a talkyard-maint.log
        ./scripts/schedule-automatic-upgrades.sh 2>&1 | tee -a talkyard-maint.log

1. Point a browser to the server address, e.g. <http://your-ip-addresss> or <http://www.example.com>
   or <http://localhost>. Or <http://localhost:8080> if you're testing with Vagrant.

   In the browser, click _Continue_ and create an admin account
   with the email address you specified when you edited `play-framework.conf` earlier (see above).
   Follow the getting-started guide ...

   <i>... Or maybe you'd like to <b>enable HTTPS</b> before you click Continue
   and submit your email address? See the Next Steps just below.</i>

Everything will restart automatically on server reboot.

Next steps:

- Do not enable HTTP2, currently doesn't work with Nginx + the Lua module (apparently [this](https://github.com/openresty/lua-nginx-module/blob/52af63a5b949d6da2289e2de3fb839e2aba4cbfd/src/ngx_http_lua_headers.c#L116) error happens).
- Enable HTTPS, see [docs/setup-https.md](docs/setup-https.md).
- Sign up for a send-email-service — see the section just below.
- Send an email to `hello at talkyard.io` so we get your address, and can
  inform you about security issues and major software
  upgrades that might require you to do something manually.
- Copy backups off-site, regularly. See the Backups section below.
- Configure Gmail, Facebook, Twitter, GitHub login,
    by creating OpenAuth apps over at their sites, and adding API keys and secrets
    to `play-framework.conf`. See below, just after the next section, about email.


Configuring email
----------------

If you don't have a mail server already, then sign up for a transactional email
service, for example Mailgun, Elastic Email, SendGrid, Mailjet or Amazon SES.
(Signing up, and verifying your sender email address and domain, is a bit complicated
— nothing you do in five minutes.)

Then, configure email settings in `/opt/talkyard/conf/play-framework.conf`, that is, fill in these values:

```
talkyard.smtp.host="..."
talkyard.smtp.port="587"
talkyard.smtp.requireStartTls=true
#talkyard.smtp.tlsPort="465"
#talkyard.smtp.connectWithTls=true
talkyard.smtp.checkServerIdentity=true
talkyard.smtp.user="..."
talkyard.smtp.password="..."
talkyard.smtp.fromAddress="support@your-organization.com"
```

(Google Cloud Engine blocks outgoing ports 587 and 465 (at least it did in the past).
Probably you email provider has made other ports available for you to use,
e.g. Amazon SES: ports 2587 and 2465.)


OpenAuth login
----------------

You want login with Facebook, Gmail and maybe Twitter and GitHub to work? Here's how.

However, we haven't written easy to follow instructions for this yet.
Send us an email: `hello at talkyard.io`, mention OpenAuth, and we'll hurry up.

<small>(There are very very brief instructions in this the markdown source but they might be out of date,
or there might be typos,
so they're hidden unless you are a tech person who knows how to view the source.)</small>

<!-- The "hidden" instructons.
You can try to follow the instructions below, and maybe won't be easy.

The login callbacks that you will need to fill in, are
`http(s)://your.website.com/-/login-auth-callback/NAME` where *NAME* is
one of `google`, `twitter`, `facebook`, `github`.

The "copy-paste" instructions below are for `/opt/talkyard/conf/play-framework.conf`,
at the end of the file.

Facebook:

 - Go to https://developers.facebook.com, and sign up or log in
 - Select the **My Apps** menu to the upper right
 - Click **Add New App**
 - Create a *Products | Facebook Login* app. (We should write more about this and
   add screenshots.)
 - Copy-paste the Facebook app id into `#facebook.clientID="..."` and `#facebook.clientSecret="..."`
   (instead of the `...`), and activate ("comment in") each line by removing the `#`.

Gmail:

First, consider visiting https://developers.google.com/people/v1/getting-started#1.-get-a-google-account
  and reading the instructions.

Then let's get started for real:
- Go to Google's People API setup tool: https://console.developers.google.com/start/api?id=people.googleapis.com&credential=client_key
- Select an existing project of yours, or create a new one.
- Click Continue.
- You should see a message "People API has been enabled" in the upper left corner.
- Click "Go to credentials"
- You should see: "Find out what kind of credentials you need".
  (If you get lost, you can go back to here, by clicking the upper left corner
  hamburger menu, then choosing "APIs & Services", then clicking "Credentials",
  then in the "Create credentials" dropdown, selecting "Help me choose". )

- In the "Which API are you using?" dropdown, select "People API".
- In the "Where will you be calling the API from?" dropdown, select "Web server".
- Below "What data will you be accessing?", select "User data".
- Click "What credentials do I need", and proceed with creating credentials if needed.

- Now you need to fill in fields for an OAuth Consent dialog. This dialog is where
  your users see your organization's name, URL and logo, and can read about
  how you handle their data — you need to add a link to a Privacy Policy,
  and Terms of Use. If you don't have your own Privacy Policy and ToU, then,
  you can use these:
    https://YOUR_TALKYARD_SERVER/-/privacy-policy
    https://YOUR_TALKYARD_SERVER/-/terms-of-use

- You'll get to a page "Client ID for Web application".
  There, in the "Authorized redirect URIs" field, type:
    https://YOUR_TALKYARD_SERVER/-/login-auth-callback/google

    (Ignore the "Authorized JavaScript origins" field.)

- (Old? blog post w photos:
    https://medium.com/@pablo127/google-api-authentication-with-oauth-2-on-the-example-of-gmail-a103c897fd98 )

Twitter:
 - Go to https://apps.twitter.com, sign up or log in.
 - Click **Create New App**
 - As callback URL, specify: `https://your.website.com/-/login-auth-callback/twitter`
 - Copy-paste your key and secret into `#twitter.consumerKey="..."` and `#twitter.consumerSecret="..."`,
   and remove the `#`.

GitHub:
 - Log in to GitHub. Click your avatar menu. Then Settings, then Developer Settings, OAuth Apps.
 - Copy-paste your client ID and secret into `#github.clientID="..."` and `#github.clientSecret="..."`,
   and remove the `#`.
-->


Viewing log files
----------------

Change directory to `/opt/talkyard/`.

Then, view the application server logs like so: `./view-logs app`
or `./view-logs -f --tail 30 app`.  
The web server: `tail -f /var/log/nginx/{access,error}.log` (mounted on the Docker host in docker-compose.yml)  
The database: `less /var/log/postgres/LOG_FILE_NAME`  
The search engine: `./view-logs search`.


Upgrading to newer versions
----------------

If you followed the instructions above — that is, if you ran these scripts:
`./scripts/configure-ubuntu.sh` and `./scripts/schedule-automatic-upgrades.sh`
— then your server should keep itself up-to-date, and ought to require no maintenance.

In a few cases you might have to do something manually, when upgrading.
Like, running `git pull` and editing config files, maybe running a shell script.
For us to be able to tell you about this, please send us an email at
`hello at talkyard.io`.

If you didn't run `./scripts/schedule-automatic-upgrades.sh`, you can upgrade
manually like so:

    sudo -i
    cd /opt/talkyard/
    ./scripts/upgrade-if-needed.sh 2>&1 | tee -a talkyard-maint.log



Backups
----------------

### Importing a backup

See [docs/how-restore-backups.md](./docs/how-restore-backup.md).


You can login to Postgres like so:

    sudo docker-compose exec rdb psql postgres postgres  # as user 'postgres'
    sudo docker-compose exec rdb psql talkyard talkyard  # as user 'talkyard'


### Backing up, manually

You should have configured automatic backups already, see the Installation
Instructions section above. In any case, you can backup manually like so:

    sudo -i
    cd /opt/talkyard/
    ./scripts/backup.sh manual 2>&1 | tee -a talkyard-maint.log


### Copy backups elsewhere

You should copy the backups to a safety off-site backup server, regularly.
Otherwise, if your main server suddenly disappears, or someone breaks into it
and ransomware-encrypts everything — you'd lose all data.

There's also a script you can copy-paste to that off-site backup server,
and run daily via Cron, to get notified via email if backups stop working
— but no, not yet implmented `[BADBKPEML]`.

See [docs/copy-backups-elsewhere.md](./docs/copy-backups-elsewhere.md).


A new Docker network
----------------

Talkyard creates its own Docker network, and assigns static IPs to the containers.
Otherwise, if a container restarts, Docker might give it a new IP,
and the other containers then couldn't find it it. —
Unless they're also restarted, so all things that have cached the old stale IP,
picks up the new IP instead. Or unless one starts using something like Traefik.
But static IPs is simpler.

You can choose the network IP range in the `.env` file — there's this variable:

```
INTERNAL_NET_SUBNET=172.26.0.0/25
```



Tips
----------------

If you start running out of disk, one reason can be old patches for automatic operating system security updates.
You can delete them to free up disk:

```
sudo apt autoremove --purge
```


Docker mounted directories
----------------

- `conf/`: Container config files, mounted read-only in the containers. Can add to a Git repo.
- `data/`: Directories mounted read-write in the containers (and sometimes read-only too). Not for Git.



License (MIT)
----------------

```
Copyright (c) 2016-2020 Debiki AB and Kaj Magnus Lindberg.

Licensed under the MIT license, see `LICENSE-MIT.txt` — and this is for the
instructions and scripts in this repository only, not for Talkyard source code
or things in other repositories.
```


<!-- vim: set et ts=2 sw=2 tw=0 fo=r list : -->
