# Flyshift
 Flyway Migrations with RedShift Demo

# HOWTO

## Install Flyway CLI
```
brew install flyway
flyway --version
```
or https://flywaydb.org/download

## Install Spawn
```
curl -sL https://run.spawn.cc/install | sh
export PATH=$PATH:$HOME/.spawnctl/bin
spawnctl version
spawnctl auth
spawnctl onboard
```
or https://www.spawn.cc/docs/getting-started.html

## Star flyway demo database
```
spawnctl create data-container \
--image postgres:flyway-getting-started \
--name flyway-container \
--lifetime 24h
```

## Setup flyway environment variables
```
export FLYWAY_LOCATIONS=filesystem:./sql
export FLYWAY_PORT="$(spawnctl get data-container flyway-container -o json  | jq -r ".port")"
export FLYWAY_URL="jdbc:postgresql://instances.spawn.cc:${FLYWAY_PORT}/foobardb"
export FLYWAY_USER="$(spawnctl get data-container flyway-container -o json  | jq -r ".user")"
export FLYWAY_PASSWORD="$(spawnctl get data-container flyway-container -o json  | jq -r ".password")"
```

## Migrate the database on spawn
```
flyway migrate
```

expected result:
```
Flyway Community Edition 7.8.1 by Redgate
Database: jdbc:postgresql://instances.spawn.cc:30741/foobardb (PostgreSQL 11.0)
Successfully validated 2 migrations (execution time 00:00.128s)
Creating Schema History table "public"."flyway_schema_history" ...
Current version of schema "public": << Empty Schema >>
Migrating schema "public" to version "1 - Create person table"
Migrating schema "public" to version "2 - Add people"
1 rows affected
1 rows affected
1 rows affected
Successfully applied 2 migrations to schema "public", now at version v2 (execution time 00:01.734s)
```


## Create cluster on Redshift
*warning* costs money: https://aws.amazon.com/redshift/pricing/

```
export RS_USER="master"
export RS_PASS="BruceLeroy1985"

aws redshift create-cluster \
    --cluster-identifier foobardb \
    --master-username "$RS_USER" \
    --master-user-password "$RS_PASS" \
    --node-type dc2.large \
    --cluster-type single-node

aws redshift wait cluster-available --cluster-identifier foobardb 

export RS_ADDR="$(aws redshift describe-clusters --cluster-identifier foobardb --query "Clusters[0].Endpoint.Address" --output text)"
export RS_PORT="$(aws redshift describe-clusters --cluster-identifier foobardb --query "Clusters[0].Endpoint.Port" --output text)"     
```

## Download Redshift JDBC Driver
```
mdir .p ./drivers
curl --output ./drivers/redshift-jdbc42-2.0.0.4.jar https://s3.amazonaws.com/redshift-downloads/drivers/jdbc/2.0.0.4/redshift-jdbc42-2.0.0.4.jar
```
Optional but recommended. Postgresql driver also works.

## Point Flyway to Redshift
```
export FLYWAY_LOCATIONS=filesystem:./sql
export FLYWAY_URL="jdbc:redshift://${RS_ADDR}:${RS_PORT}/dev"
export FLYWAY_USER="$RS_USER"
export FLYWAY_PASSWORD="$RS_PASS"
```

## Migrate Redshift
```
flyway migrate
```

works :)
```
Flyway Community Edition 7.8.1 by Redgate
Database: jdbc:redshift://foobardb.ckqcsnc5fhc8.us-east-1.redshift.amazonaws.com:5439/dev (Redshift 8.0)
Successfully validated 2 migrations (execution time 00:00.360s)
Creating Schema History table "public"."flyway_schema_history" ...
Current version of schema "public": << Empty Schema >>
Migrating schema "public" to version "1 - Create person table"
Migrating schema "public" to version "2 - Add people"
1 rows affected
1 rows affected
1 rows affected
Successfully applied 2 migrations to schema "public", now at version v2 (execution time 00:08.083s)
```

## Verify
```
export PGPASSWORD="$RS_PASS"
psql -h ${RS_ADDR} -p ${RS_PORT} -d dev -U $RS_USER -c 'SELECT * FROM PERSON;'
```

done \o/
```
 id |  name   
----+---------
  1 | Axel
  2 | Mr. Foo
  3 | Ms. Bar
```

Don't forget to delete the Redshift cluster and spawn.cc data images.
