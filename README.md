# WB Apollo
This repository contains the configs and scripts used to run the WB Apollo server.

## New instance setup
New instances are current set up manually. To setup a new instance (for OS upgrades, machine replacement etc.):
 1. Start a new machine (any linux OS)
 2. Install docker (if not pre-installed)
 3. Copy the [docker-compose.yml](./docker-compose.yml) and the [apollo-service-commons.yml](./apollo-service-commons.yml) files onto the machine.
 4. Migrate the data if required (see [below](#apollo-data-migration)).
    If no migration was required, continue below steps.
 5. Set all environment variables used and required by docker compose (if not already done so during data migration).
    Required variables are coded as `${VAR_NAME?}` in the [apollo-service-commons.yml](./apollo-service-commons.yml) file.
 6. Start the appolo server daemon (if not already done so during data migration):
    ```bash
    docker compose up -d wb-apollo-server
    ```

## Apollo data migration
To migrate/restore all data from another Apollo instance to a new instance,
to be used by the Apollo service defined in this repository, do the following steps:

 1. Create a snapshot of the data volume attached to the old instance (on which the jbrowse data files are stored).
 2. If the (postgres) DBs on the old instance are not stored on the data volume, make DB dumps of the appollo DB and the chado DB on it:
    ```bash
    $ sudo -u postgres pg_dump --format=c -d apollo-release-production > apollo-release-production_dump.pg_dump
    $ sudo -u postgres pg_dump --format=c -d apollo-production-chado > apollo-production-chado_dump.pg_dump
    ```
 3. Transfer the dump files to the new machine if generated, store them in the `./apollo_backups` directory (create the directory if needed).
 4. Once the snapshot from step 1 completed, restore it as a new volume and attach it to the new instance.
 5. Mount the new volume (assumed attachment on `/dev/sdg` and OS Ubuntu):
    ```bash
    $ export APOLLO_DATA_VOLUME=/data
    $ mount /dev/xvdg ${APOLLO_DATA_VOLUME}
    ```
 6. If the root of this volume contains the JBrowse data files (subfolders for each species, rather than JBrowse/Postgres/Apollo data separation),
    reorganise the data on it (on the new instance):
     * All JBrowse data folders go in a folder `${APOLLO_DATA_VOLUME}/jbrowse_data`
       ```bash
       $ cd ${APOLLO_DATA_VOLUME}
       $ mkdir jbrowse_data/
       $ mv * jbrowse_data/
       ```
     * Make an empty folder `./temp_apollo/data` in the Apollo data volume, for temporary apollo intermediate files.
     * Make an empty folder `./temp_apollo/io` in the Apollo data volume, for file transfer between container and host system.
     * Make an empty folder `./postgres_data`, in the Apollo data volume, for permanent postgres DB storage.
 4. Export all required environment variables with the appropiate DB credentials
    (see the [appollo-service-commons.yml](./appollo-service-commons.yml) file for a list of variables used).
    Use an extra space before each export command so the secrets don't get stored to bash history (` export VAR_NAME=secret`).
 5. On the new instance, start the apollo container:
    ```bash
    $ docker compose up wb-apollo-server
    ```
    Let it complete startup (to create the necessary DB roles), then stop the process (ctrl+C or cmd+.)
 6. Open a terminal into an appollo service container, with all data volumes mounted:
    ```bash
    $ docker compose run --rm apollo-import-export
    ```
 7. In this container, start the postgres server.
    ```bash
    /usr/lib/postgresql/9.6/bin/pg_ctl -D /var/lib/postgresql/9.6/main -w start
    ```
 8. Drop and restore the main postgres database.
    ```bash
    dropdb $WEBAPOLLO_DB_NAME
    createdb -E UTF-8 -O $WEBAPOLLO_DB_USERNAME $WEBAPOLLO_DB_NAME
    pg_restore --no-owner -d $WEBAPOLLO_DB_NAME ./apollo_backups/apollo-release-production.pg_dump
    ```
 9. Drop and restore the chado postgres database.
    ```bash
    dropdb $CHADO_DB_NAME
    createdb -E UTF-8 -O $CHADO_DB_USERNAME $CHADO_DB_NAME
    pg_restore --no-owner -d $CHADO_DB_NAME ./apollo_backups/apollo-production-chado.pg_dump
    ```
10. Exit the import/export container (ctrl+d) and start the apollo service again
    (this time in the background).
    ```bash
    docker compose up -d wb-apollo-server
    ```
11. The Apollo server will attempt a database schema migration when needed.
    Inspect the logs for the error documented [here](https://github.com/GMOD/Apollo/issues/2522).
    If the error occured, patch the CHADO database manually:
    ```bash
    docker exec -it wb.apollo.server psql -c "ALTER TABLE chadoprop ADD COLUMN cvterm_id int8 not null DEFAULT 1"
    docker compose restart wb-apollo-server
    ```
12. After successful server startup, browse to the web interface (`http://<IP>:8080/apollo/annotator/index`) and log in using the admin user.
    If you get the notification pop-up stating "Unable to write to directory apollo_data.",
    fill in the new apollo temp data directory path (`/temp_apollo/data`) in the textbox below
    and click the "update common data path" button below.

## CLI scripting
The Apollo container provides a set of CLI scripts and utilities for batch import/export and track management,
in the `$CATALINA_BASE/webapps/apollo/jbrowse/bin/` directory (`/var/lib/tomcat9/webapps/apollo/jbrowse/bin/` when using Apollo 2.8.1).
To use any of the Apollo CLI scripts:
```bash
docker exec wb.apollo.server sh -c 'perl $CATALINA_BASE/webapps/apollo/jbrowse/bin/script-name.pl --arguments'
```
Keep in mind that since this runs the process in the apollo container, paths available to it are also defined by the container environment:
 * jbrowse track files found in the `jbrowse_data` subdirectory of the apollo volume on the host system,
   live under `/data` in the container.
 * Files put into the `temp_apollo/io` subdirectory in the apollo data volume
   can be accessed within the container at `/temp_apollo/io` and vice versa.
