Build image
===========

        docker build -f Dockerfile-plus-pmm-client -t proxysql:testing --no-cache .

Build containers of pmm
=======================

        # create networking
        docker network create testing

        # start pmm-data container
        docker create --network testing -v /opt/prometheus/data -v /opt/consul-data -v /var/lib/mysql --name pmm-data percona/pmm-server:1.17.3 /bin/true
        # start pmm-server container
        docker run --network testing -d -p 80:80 -e METRICS_RESOLUTION=5s -e METRICS_RETENTION=3h -e METRICS_MEMORY=4194302 --volumes-from pmm-data --name pmm-server --restart always percona/pmm-server:1.17.3

Set up a single instance
========================

        # start mysql container
        docker run -it --network testing -p 3306:3306 -p 3307:3307 -p 6032:6032 -p 6080:6080 --name proxysql_test -d proxysql:testing

        # test services
        docker exec -it proxysql_test pt-heartbeat --version
        docker exec -u root -it proxysql_test pmm-admin --version
        docker exec -it proxysql_test mysql -u admin -padmin -h 127.0.0.1 -P 6032 -e "SHOW DATABASES"

        # register on pmm-server
        docker exec -u root -it proxysql_test pmm-admin config --client-name mysql_test --server pmm-server:80
        docker exec -u root -it proxysql_test pmm-admin add proxysql:metrics --dsn "stats:stats@tcp(127.0.0.1:6032)/"

Set up some containers in replica
=================================

        # create your.cnf.d/ with
        docker cp mysql_test:/etc/my.cnf.d/docker.cnf my.cnf.d/
        cp your.replica.cnf my.cnf.d/ # or you can use  my.cnf.d/ms.cnf

        # start mysql containers
        seq 1 3 | while read n; do mkdir mysql0$n; cp my.cnf.d/* mysql0$n/; echo -e "[mysqld]\nserver-id=$n" > mysql0$n/serverid.cnf; docker run -it --network testing -v $PWD/mysql0$n:/etc/my.cnf.d --name mysql0$n -e MYSQL_ROOT_PASSWORD=secret -d percona-server:testing; done

        # replica set up
        docker exec -it mysql01 mysql -u root -psecret -e "SHOW MASTER STATUS"
        docker exec -it mysql02 mysql -u root -psecret -e "SHOW MASTER STATUS"
        docker exec -it mysql03 mysql -u root -psecret -e "SHOW MASTER STATUS"
        docker exec -it mysql02 mysql -u root -psecret -e "CHANGE MASTER TO MASTER_HOST='mysql01', MASTER_USER='root', master_password='secret', MASTER_PORT=3306, MASTER_LOG_FILE='mysql-bin.000004', MASTER_LOG_POS=120; START SLAVE"
        docker exec -it mysql03 mysql -u root -psecret -e "CHANGE MASTER TO MASTER_HOST='mysql02', MASTER_USER='root', master_password='secret', MASTER_PORT=3306, MASTER_LOG_FILE='mysql-bin.000004', MASTER_LOG_POS=120; START SLAVE"

        # test replica
        docker exec -it mysql01 mysql -u root -psecret -e "CREATE DATABASE test"
        docker exec -it mysql01 mysql -u root -psecret -e "USE test; CREATE TABLE one(id int auto_increment, name varchar(25), datetime timestamp DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP, PRIMARY KEY (id))"
        docker exec -it mysql03 mysql -u root -psecret -e "USE test; SHOW TABLES"

        # heartbeat set up masters
        docker exec -it mysql01 pt-heartbeat -D test -h localhost --user root --password secret --create-table --check --master-server-id 1
        docker exec -it mysql01 pt-heartbeat --daemonize -D test -h localhost --user root --password secret --update
        docker exec -it mysql02 pt-heartbeat --daemonize -D test -h localhost --user root --password secret --update
        
        # test heartbeat
        docker exec -it mysql03 mysql -u root -psecret -e "SELECT * FROM test.heartbeat"
        docker exec -it mysql02 pt-heartbeat -D test -h localhost --user root --password secret --check --master-server-id 1
        docker exec -it mysql03 pt-heartbeat -D test -h localhost --user root --password secret --check --master-server-id 2

And then, by pmm-server interface, add the dashboard [MySQL Replication](https://github.com/percona/grafana-dashboards/blob/master/dashboards/MySQL_Replication.json) with the graph for the heartbeat collection.

Add MySQL instances on ProxySQL instance
========================================

        # enter the container
        docker exec -u root -it proxysql mysql -uadmin -padmin -h 127.0.0.1 -P 6032
        # add servers
        INSERT INTO mysql_servers(hostgroup_id,hostname,port) VALUES (1,'mysql01',3306);
        INSERT INTO mysql_servers(hostgroup_id,hostname,port) VALUES (2,'mysql02',3306);
        INSERT INTO mysql_servers(hostgroup_id,hostname,port) VALUES (2,'mysql03',3306);
        LOAD MYSQL SERVERS TO RUNTIME;
        SAVE MYSQL SERVERS TO DISK;
        # add users
        INSERT INTO mysql_users(username,password) VALUES ('user1','password1');
        LOAD MYSQL USERS TO RUNTIME;
        SAVE MYSQL USERS TO DISK;

And then, by pmm-server interface, see the dashboard [ProxySQL Overview](https://github.com/percona/grafana-dashboards/blob/master/dashboards/ProxySQL_Overview.json).