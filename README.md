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
```
```

```
```

```
```

```
```

```
```

```
```

```
```



