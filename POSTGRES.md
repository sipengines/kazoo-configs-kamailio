## Notes on upgrading to Kamailio 5.5 and moving to Postgres backend.

Thus removing the dependency on KazooDB which is closed source.

First completely remove the current installation 

```
Removing existing kazoo-kamailo:

  yum remove kazoo-kamailio
  yum remove kamailio
  yum remove kazoo-configs-kamailio
```
Next install the lastest Kamailio (currently 5.5.0)
```
yum -y install yum-utils
yum-config-manager --add-repo https://rpm.kamailio.org/centos/kamailio.repo
yum install kamailio kamailio-kazoo kamailio-outbound kamailio-presence kamailio-tls   kamailio-utils kamailio-uuid    kamailio-websocket kamailio-xmpp kamailio-postgresql
```
Now install latest postgres (currently 12.7)
```
# Install the repository RPM:
sudo yum install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-6-x86_64/pgdg-redhat-repo-latest.noarch.rpm
sudo yum install -y postgresql12-server
/usr/pgsql-12/bin/postgresql-12-setup  initdb

sudo chkconfig postgresql-12 on
sudo service postgresql-12 start
```
Create Kamailio DB
```
su postgres -c '/usr/pgsql-12/bin/createdb kamailio'
```
Add DB user and grant privileges
```
sudo su postgres -c psql
	CREATE USER kamailio WITH PASSWORD 'kamailio';
	GRANT ALL privileges on database kamailio to kamailio;
```
Modify default postgres authentication to allow kamailio access
```
vi /var/lib/pgsql/12/data/pg_hba.conf

# "local" is for Unix domain socket connections only
local   all             all                                     password
# IPv4 local connections:
host    all             all             127.0.0.1/32            password
# IPv6 local connections:
host    all             all             ::1/128                 password
```
Resart postgres and check kamailio access to DB
```
systemctl restart postgresql-12
psql -U kamailio -d postgres://kamailio:kamailio@127.0.0.1/kamailio
```
Get the postgres kamailio configs:
```
git clone https://github.com/kageds/kazoo-configs-kamailio /etc/kazoo/kamailio
cd /etc/kazoo/kamailio
git checkout 4.3-postgres
```
Initialize the kamailio database with all the required tables:
```
psql -U kamailio -d postgres://kamailio:kamailio@127.0.0.1/kamailio -f /etc/kazoo/kamailo/db_scripts/kamailio_initdb_postgres.sql
```
Notice that the backend DB in local.cfg is now postgres
```
#!trydef KZ_DB_MODULE postgres
#!substdef "!KAMAILIO_DBMS!postgres!g"
#!substdef "!KAZOO_DB_URL!postgres://kamailio:kamailio@127.0.0.1/kamailio!g"
```
Start kamailio (NOT kazoo-kamailio)
```
systemctl enable kamailio
systemctl start kamailio
```

