# BAMOE-KieServer-PostgreSQL

Install PostgreSQL and Configure to use with IBM BAM OE KieServer

## Install PostgreSQL in RH9

### Installation
```
sudo su -

#-------------------------
# install package, client 'psql',  and create user 'postgres'
dnf install -y postgresql-server

ls -al /var/lib/pgsql/

# data is empty
ls -al /var/lib/pgsql/data

#-------------------------
# initialize db
postgresql-setup --initdb

#-------------------------
# set password
passwd postgres

#-------------------------
# how to install postgresql client tool only
sudo dnf install -y postgresql
```

### Basic configuration

```
#-------------------------
# backup config file
PGCONF=/var/lib/pgsql/data/postgresql.conf

if [ -f "${PGCONF}" ]; then
  echo "Backup file "${PGCONF}
	cp ${PGCONF} ${PGCONF}-original
else
  echo "File "${PGCONF}" not found"
	exit
fi

#-------------------------
# change password encryption
cat ${PGCONF} | grep password_encryption

if [ -f "${PGCONF}" ]; then
    echo "Updating password encryption mode in file "${PGCONF}
	cp ${PGCONF} ${PGCONF}-original
	sed 's/#password_encryption = md5/password_encryption = scram-sha-256/g' -i ${PGCONF} 
else
    echo "File "${PGCONF}" not found"
	exit
fi

cat ${PGCONF} | grep password_encryption

#-------------------------
# change number of connections from default (100) to 200
cat ${PGCONF} | grep max_connections

if [ -f "${PGCONF}" ]; then
  echo "Updating max connections in file "${PGCONF}
	sed 's/max_connections = 100/max_connections = 200/g' -i ${PGCONF} 
else
  echo "File "${PGCONF}" not found"
	exit
fi

cat ${PGCONF} | grep max_connections

#-------------------------
# enable listen on any address
cat ${PGCONF} | grep listen_addresses

sed "s/#listen_addresses = 'localhost'/listen_addresses = '*'/g" -i ${PGCONF}

cat ${PGCONF} | grep listen_addresses

#-------------------------
# backup file
PGCLIENTAUTH=/var/lib/pgsql/data/pg_hba.conf

if [ -f "${PGCLIENTAUTH}" ]; then
  echo "Backup file "${PGCLIENTAUTH}
	cp ${PGCLIENTAUTH} ${PGCLIENTAUTH}-original
else
  echo "File "${PGCLIENTAUTH}" not found"
	exit
fi

#-------------------------
# update host access method
cat ${PGCLIENTAUTH} | grep ^host

cp ${PGCLIENTAUTH} ${PGCLIENTAUTH}-original
sed 's/host.*all.*all.*127.0.0.1\/32.*ident/host\tall\t\tall\t\t127.0.0.1\/32\t\tscram-sha-256/g' -i ${PGCLIENTAUTH}

cat ${PGCLIENTAUTH} | grep ^host

#-------------------------------------------------------
# allow remote access from IP range

# !!! use your IP range
echo "host all all 192.168.13.0/24 trust" >> ${PGCLIENTAUTH}

#-------------------------
# start db and enable autostart
systemctl start postgresql.service
systemctl enable postgresql.service

#-------------------------
# set password for postgres user
su - postgres

psql -c "\conninfo"
psql -c "\du+;"


# change password
psql -c "ALTER USER postgres WITH PASSWORD 'post01gres';"

exit
```


### Commands to uninstall PostgreSQL
```
#-------------------------
# uninstall postgres

sudo su -

systemctl disable postgresql.service
systemctl stop postgresql.service

dnf remove -y postgresql-server
rm -fR /var/lib/pgsql

userdel -r postgres
```

### Reference

https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html/configuring_and_using_database_servers/using-postgresql_configuring-and-using-database-servers


## Configure PostgreSQL and JBoss EAP datasource for IBM BAM OE KieServer

### Create database for KieServer
```
#-------------------------
su - postgres

# create new user (!!! use your own credentials)
psql -c "CREATE USER kieserver WITH PASSWORD 'kie01server' CREATEROLE CREATEDB;"

# list users
psql -c "\du+;"

#-------------------------
# drop old if already present
PGPASSWORD=kie01server psql -U kieserver -h 127.0.0.1 -d postgres -c "DROP DATABASE kieserver01;"

#-------------------------
# create new db
PGPASSWORD=kie01server psql -U kieserver -h 127.0.0.1 -d postgres -c "CREATE DATABASE kieserver01;"

#-------------------------
# list db
PGPASSWORD=kie01server psql -U kieserver -h 127.0.0.1 -d postgres -c "\l+"

# list schemas
PGPASSWORD=kie01server psql -U kieserver -h 127.0.0.1 -d postgres -c "\dn+"
```

### Create schema and tables
```
# set the location of 'postgresql-jbpm-schema.sql' file (extract from bamoe-8.x.y-migration-tool.zip)
JBPM_SCHEMA_FILE=/tmp/postgresql-jbpm-schema.sql

# load into db
PGPASSWORD=kie01server psql -U kieserver -h 127.0.0.1 -d kieserver01 -a -f ${JBPM_SCHEMA_FILE}

# list tables
PGPASSWORD=kie01server psql -U kieserver -h 127.0.0.1 -d kieserver01 -c "\dt+"
```

### Configure PostgreSQL driver in JBoss EAP server

Download PostgreSQL jdbc driver from https://jdbc.postgresql.org/download/
```
#-------------------------
# add postgresql driver (set your file fullpath)
./jboss-cli.sh

module add --name=com.postgresql --resources=/home/marco/Downloads/postgresql-42.5.1.jar --dependencies=javaee.api,sun.jdk,ibm.jdk,javax.api,javax.transaction.api

exit
```

### Configure PostgreSQL datasource in JBoss EAP server

In section '<system-properties>' of your configuration file ( one of standalone.xml, standalone-full.xml, etc...) add the following properties
<ul>org.kie.server.persistence.ds</ul>
<ul>org.kie.server.persistence.dialect</ul>
<ul>org.kie.server.persistence.org.jbpm.ejb.timer.tx</ul>

set your datasource name as for example <b>java:jboss/datasources/PostgresDS</b>, the db dialect specific to PostgreSQL <b>org.hibernate.dialect.PostgreSQLDialect</b> and to enable transactionality for timers and concurrent access to <b>true</b>.
```
<server xmlns="urn:jboss:domain:16.0">
  <extensions>
  ...
  </extensions>
  <system-properties>
    ...
    <!-- Datasource, Dialect, TX-->
    <property name="org.kie.server.persistence.ds" value="java:jboss/datasources/PostgresDS"/>
    <property name="org.kie.server.persistence.dialect" value="org.hibernate.dialect.PostgreSQLDialect"/>
    <property name="org.jbpm.ejb.timer.tx" value="true"/>
```

#### Change the configuration of 'timer-service' in ejb subsytem section 'urn:jboss:domain:ejb3'

From original configuration of type 'default-file-store'
```
  ...
  <subsystem xmlns="urn:jboss:domain:ejb3:9.0">
    ...
    <timer-service thread-pool-name="default" default-data-store="default-file-store">
      <data-stores>
          <file-data-store name="default-file-store" path="timer-service-data" relative-to="jboss.server.data.dir"/>
      </data-stores>
    </timer-service>
    ...

```

to 'db-store' using your 'datasource-jndi-name' value
```
  ...
  <subsystem xmlns="urn:jboss:domain:ejb3:9.0">
    ...
    <timer-service thread-pool-name="default" default-data-store="db-store">
      <data-stores>
          <database-data-store name="db-store" datasource-jndi-name="jboss/datasources/PostgresDS"/>
      </data-stores>
    </timer-service>
    ...
```
#### Add your datasource and driver configuration in datasources subsytem section 'urn:jboss:domain:datasources'

Set name,url,credentials,conn-pool values
```
    <subsystem xmlns="urn:jboss:domain:datasources:6.0">
      <datasources>
        ...
        <datasource jndi-name="java:jboss/datasources/PostgresDS" pool-name="PostgresDS">
            <connection-url>jdbc:postgresql://localhost:5432/kieserver01</connection-url>
            <driver>postgresql</driver>
            <security>
              <user-name>kieserver</user-name>
              <password>kie01server</password>
            </security>
            <validation>
              <valid-connection-checker class-name="org.jboss.jca.adapters.jdbc.extensions.postgres.PostgreSQLValidConnectionChecker"/>
              <validate-on-match>true</validate-on-match>
              <background-validation>false</background-validation>
              <exception-sorter class-name="org.jboss.jca.adapters.jdbc.extensions.postgres.PostgreSQLExceptionSorter"/>
            </validation>
            <pool>
              <min-pool-size>50</min-pool-size>
              <max-pool-size>150</max-pool-size>
            </pool>
        </datasource>
        
        <drivers>
            <driver name="postgresql" module="com.postgresql">
              <xa-datasource-class>org.postgresql.xa.PGXADataSource</xa-datasource-class>
            </driver>
        </drivers>            
        ...

```

Start your KieServer.

If you are interested in datsource consumption statistics run the following commands
```
# enable pool metrics
./jboss-cli.sh -c "/subsystem=datasources/data-source=PostgresDS/statistics=pool:write-attribute(name=statistics-enabled,value=true)"

# enable jdbc metrics
./jboss-cli.sh -c "/subsystem=datasources/data-source=PostgresDS/statistics=jdbc:write-attribute(name=statistics-enabled,value=true)"
```

the query the metrics using the following commands
```
# query pool metrics
./jboss-cli.sh -c "/subsystem=datasources/data-source=PostgresDS/statistics=pool:read-resource(include-runtime=true)"

# query jdbc metrics
./jboss-cli.sh -c "/subsystem=datasources/data-source=PostgresDS/statistics=jdbc:read-resource(include-runtime=true)"

# query pool metrics grepping connections counters
./jboss-cli.sh -c "/subsystem=datasources/data-source=PostgresDS/statistics=pool:read-resource(include-runtime=true)" | egrep "InUseCount|MaxUsedCount|AverageGetTime"
```


### References

PostgreSQL configuration guide

https://access.redhat.com/documentation/en-us/red_hat_jboss_enterprise_application_platform/7.3/html-single/configuration_guide/index#example_postgresql_datasource

Datasource statistics

https://access.redhat.com/documentation/en-us/jboss_enterprise_application_platform/6.3/html/administration_and_configuration_guide/datasource_statistics





