# Native Immich(guide to migrate from docker stack)

This repository provides instructions and helper scripts to install [Immich](https://github.com/immich-app/immich) without Docker, natively in proxmox lxc, and migrate from Immich installed with docker stack.
(This guide is fork of [Immich Native](https://github.com/arter97/immich-native). I added some instructions to migrate from preexisting Immich and did some tweaks)

### Notes

 * This is tested on Debian 12.02 on x86 as the host distro, but it will be similar on other distros. 

 * This guide installs Immich to `/var/lib/immich`. To change it, replace it to the directory you want in this README and `install.sh`'s `$IMMICH_PATH`.

 * This guid will make 2 lxc conainers, 1st one for postgresql and 2nd one for Immich service, redis and machine learning service.
 * 
 * The [install.sh](install.sh) script currently is using Immich v1.108.0. It should be noted that due to the fast-evolving nature of Immich, the install script may get broken if you replace the `$TAG` to something more recent.

 * `mimalloc` is deliberately disabled as this is a native install and sharing system library makes more sense.

 * Original Native Immich used `pgvector` instead of `pgvecto.rs(used by official Immich) to remove additional Rust build dependency. But I found it hard to change from pgvecto.rs to pgvector with pre-exiting postgresql DB. So This guide will use original pgvecto.rs

 * Microservice and machine-learning's host is opened to 0.0.0.0 in the default configuration. This behavior is changed to only accept 127.0.0.1 during installation. Only the main Immich service's port, 3001, is opened to 0.0.0.0.

 * Only the basic CPU configuration is used. Hardware-acceleration such as CUDA is unsupported. In my personal experience, importing about 10K photos on a x86 processor doesn't take an unreasonable amount of time (less than 30 minutes).

 * JPEG XL support may differ official Immich due to base-image's dependency differences.


## 1. Make backups

 * Make backup of old Immich databse, following [Offician Immich guide](https://immich.app/docs/administration/backup-and-restore)

   ``` bash
   docker exec -t immich_postgres pg_dumpall --clean --if-exists --username=postgres | gzip > "/YOUR/PATH/OF/BACKUP/dump.sql.gz"
   ```

 * Make backup of pre-existing docker stack and Images(in case of woops moment) from following locations

   UPLOAD_LOCATION/library
   UPLOAD_LOCATION/upload
   UPLOAD_LOCATION/profile


## 2. Prepare seperate postgresql lxc for Immich

 * Make new lxc with debian 12.02. give some core and memory for migration(2core and 16G memory was sufficient)
 
 * prepare basic stuff
   ``` bash
   export LANGUAGE=en_US.UTF-8                   
   export LANG=en_US.UTF-8
   locale-gen en_US.UTF-8
   apt update && apt upgrade -y
   apt install sudo
   ```
  
 * Install [postgresql](https://www.postgresql.org/download/linux/debian/)
   ``` bash
   apt install -y postgresql-common
   /usr/share/postgresql-common/pgdg/apt.postgresql.org.sh
   apt -y install postgresql
   ```

 * Install [pgvecto.rs](https://docs.pgvecto.rs/getting-started/installation.html) It's import to install 0.2.1. version, as Immich does not support current version(0.3) as of 16th of July, 2024
    ``` bash
    wget https://github.com/tensorchord/pgvecto.rs/releases/download/v0.2.1/vectors-pg16_0.2.1_amd64.deb 
    sudo apt install ./vectors-pg16_0.2.1_amd64.deb
    ```

 * Edit postgresql conf files
    ``` bash
    sed -i "s|listen_addresses = 'localhost'|listen_addresses = '*' |" /etc/postgresql/16/main/postgresql.conf    
    sed -i "s|port = 5432|port = YOUR_POSTGRES_PORT|" /etc/postgresql/16/main/postgresql.conf
    sed -i '/log_destination/s/^#//g'  /etc/postgresql/16/main/postgresql.conf
    sed -i '/logging_collector/^#//g'  /etc/postgresql/16/main/postgresql.conf
    sed -i '/log_directory/^#//g'  /etc/postgresql/16/main/postgresql.conf
    sed -i '/log_filename/^#//g'  /etc/postgresql/16/main/postgresql.conf
    sed -i '/log_file_mode/^#//g'  /etc/postgresql/16/main/postgresql.conf
    sed -i '/log_rotation_age/^#//g'  /etc/postgresql/16/main/postgresql.conf
    sed -i '/log_rotation_size/^#//g'  /etc/postgresql/16/main/postgresql.conf
    sed -i "s|#shared_preload_libraries = ''|shared_preload_libraries = 'vectors.so'|" /etc/postgresql/16/main/postgresql.conf
    echo host        immich,postgres        postgres        YOUR_IMMICHSERVER_IP/32        scram-sha-256 | tee -a /etc/postgresql/16/main/pg_hba.conf
    systemctl restart postgresql
    ```

 * Restore Immich DB
    ``` bash
     gunzip < "/YOUR/PATH/OF/BACKUP/dump.sql.gz" | sed "s/SELECT pg_catalog.set_config('search_path', '', false);/SELECT pg_catalog.set_config('search_path', 'public, p
g_catalog', true);/g" | sudo -u postgres psql   
    ```

 * Test new DB
   At immich docker stack, add enviorment variable "DBURL" with value of "postgresql://YOUR_DB_ID:YOUR_DB_PASSWD@YOUR_NEW_POSTGRESQL_SERVER_IP:YOUR_POSTGRES_PORT/immich"

   Update stack, and immich should work with new DB(if not, check docker container logs and db logs(at /var/log/postgresql/16/main/log)


## 3. Prepare Immich lxc

 * Make new lxc

 todo from here.
 *  
## 1. Install dependencies

 * [Node.js](https://github.com/nodesource/distributions)

 * [PostgreSQL](https://www.postgresql.org/download/linux)

 * [Redis](https://redis.io/docs/install/install-redis/install-redis-on-linux)

As the time of writing, Node.js v20 LTS, PostgreSQL 16 and Redis 7.2.4 was used.

 * [pgvector](https://github.com/pgvector/pgvector)

pgvector is included in the official PostgreSQL's APT repository:

``` bash
sudo apt install postgresql(-16)-pgvector
```

 * [FFmpeg](https://github.com/FFmpeg/FFmpeg)

Immich uses FFmpeg to process media.

FFmpeg provided by the distro is typically too old.
Either install it from [jellyfin](https://github.com/jellyfin/jellyfin-ffmpeg/releases)
or use [FFmpeg Static Builds](https://johnvansickle.com/ffmpeg) and install it to `/usr/bin`.

### Other APT packages

``` bash
sudo apt install --no-install-recommends \
        python3-venv \
        python3-dev \
        uuid-runtime \
        autoconf \
        build-essential \
        unzip \
        jq \
        perl \
        libnet-ssleay-perl \
        libio-socket-ssl-perl \
        libcapture-tiny-perl \
        libfile-which-perl \
        libfile-chdir-perl \
        libpkgconfig-perl \
        libffi-checklib-perl \
        libtest-warnings-perl \
        libtest-fatal-perl \
        libtest-needs-perl \
        libtest2-suite-perl \
        libsort-versions-perl \
        libpath-tiny-perl \
        libtry-tiny-perl \
        libterm-table-perl \
        libany-uri-escape-perl \
        libmojolicious-perl \
        libfile-slurper-perl \
        liblcms2-2 \
        wget
```

A separate Python's virtualenv will be stored to `/var/lib/immich`.

## 2. Prepare `immich` user

This guide isolates Immich to run on a separate `immich` user.

This provides basic permission isolation and protection.

``` bash
sudo adduser \
  --home /var/lib/immich/home \
  --shell=/sbin/nologin \
  --no-create-home \
  --disabled-password \
  --disabled-login \
  immich
sudo mkdir -p /var/lib/immich
sudo chown immich:immich /var/lib/immich
sudo chmod 700 /var/lib/immich
```

## 3. Prepare PostgreSQL DB

Create a strong random string to be used with PostgreSQL immich database.

You need to save this and write to the `env` file later.

``` bash
sudo -u postgres psql
postgres=# create database immich;
postgres=# create user immich with encrypted password 'YOUR_STRONG_RANDOM_PW';
postgres=# grant all privileges on database immich to immich;
postgrse=# ALTER USER immich WITH SUPERUSER;
postgres=# \q
```

## 4. Prepare `env`

Save the [env](env) file to `/var/lib/immich`, and configure on your own.

You'll only have to set `DB_PASSWORD`.

``` bash
sudo cp env /var/lib/immich
sudo chown immich:immich /var/lib/immich/env
```

## 5. Build and install Immich

Clone this repository to somewhere anyone can access (like /tmp) and run `install.sh` as root.

Anytime Immich is updated, all you have to do is run it again.

In summary, the `install.sh` script does the following:

#### 1. Clones and builds Immich.

#### 2. Installs Immich to `/var/lib/immich` with minor patches.

  * Sets up a dedicated Python venv to `/var/lib/immich/app/machine-learning/venv`.

  * Replaces `/usr/src` to `/var/lib/immich`.

  * Limits listening host from 0.0.0.0 to 127.0.0.1. If you do not want this to happen (make sure you fully understand the security risks!), comment out the `sed` command in `install.sh`'s "Use 127.0.0.1" part.

## 6. Install systemd services

Because the install script switches to the immich user during installation, you must install systemd services manually:

``` bash
sudo cp immich*.service /etc/systemd/system/
sudo systemctl daemon-reload
for i in immich*.service; do
  sudo systemctl enable $i
  sudo systemctl start $i
done
```

## Done!

Your Immich installation should be running at 3001 port, listening from localhost (127.0.0.1).

Immich will additionally use localhost's 3002 and 3003 ports.

Please add firewall rules and apply https proxy and secure your Immich instance.

## Uninstallation

``` bash
# Run as root!

# Remove Immich systemd services
for i in immich*.service; do
  systemctl stop $i
  systemctl disable $i
done
rm /etc/systemd/system/immich*.service
systemctl daemon-reload

# Remove Immich files
rm -rf /var/lib/immich

# Delete immich user
deluser immich

# Remove Immich DB
sudo -u postgres psql
postgres=# drop user immich;
postgres=# drop database immich;
postgres=# \q

# Optionally remove dependencies
# Review /var/log/apt/history.log and remove packages you've installed
```
