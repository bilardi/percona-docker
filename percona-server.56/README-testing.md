Build image
===========

        docker build -f Dockerfile-plus-pt-and-pmm-client -t percona-server:testing --no-cache .

Build containers of pmm
=======================

        # create networking
        docker network create testing

        # start pmm-data container
        docker create --network testing -v /opt/prometheus/data -v /opt/consul-data -v /var/lib/mysql --name pmm-data percona/pmm-server:1.17.1 /bin/true
        # start pmm-server container
        docker run --network testing -d -p 80:80 -e METRICS_RESOLUTION=5s -e METRICS_RETENTION=3h -e METRICS_MEMORY=4194302 --volumes-from pmm-data --name pmm-server --restart always percona/pmm-server:1.17.1

Set up a single instance
========================

        # start mysql container
        docker run -it --network testing --name mysql_test -e MYSQL_ROOT_PASSWORD=secret -d percona-server:testing

        # test services
        docker exec -it mysql_test pt-heartbeat --version
        docker exec -u root -it mysql_test pmm-admin --version
        docker exec -it mysql_test mysql -u root -psecret -e "SHOW DATABASES"

        # register on pmm-server
        docker exec -u root -it mysql_test pmm-admin config --client-name mysql_test --server pmm-server:80
        docker exec -u root -it mysql_test pmm-admin add mysql:metrics --user root --password secret --host mysql_test

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

        # heartbeat set up
        docker exec -it mysql01 pt-heartbeat -D test -h mysql01 --user root --password secret --create-table --check --master-server-id 1
        docker exec -it mysql01 pt-heartbeat --daemonize -D test --user root --password secret --update -h mysql01;
        docker exec -it mysql02 pt-heartbeat --daemonize -D test --user root --password secret --update -h mysql02;
        docker exec -it mysql03 pt-heartbeat --daemonize -D test --user root --password secret --update -h mysql03;

        # test heartbeat
        docker exec -it mysql03 mysql -u root -psecret -e "SELECT * FROM test.heartbeat"

        # register on pmm-server
        docker exec -u root -it mysql01 pmm-admin config --client-name mysql01 --server pmm-server:80
        docker exec -u root -it mysql02 pmm-admin config --client-name mysql02 --server pmm-server:80
        docker exec -u root -it mysql03 pmm-admin config --client-name mysql03 --server pmm-server:80
        docker exec -u root -it mysql01 pmm-admin add mysql:metrics --user root --password secret --host localhost
        docker exec -u root -it mysql02 pmm-admin add mysql:metrics --user root --password secret --host localhost -- -- collect.heartbeat.database=test --collect.heartbeat
        docker exec -u root -it mysql03 pmm-admin add mysql:metrics --user root --password secret --host localhost -- -- collect.heartbeat.database=test --collect.heartbeat

And then, by pmm-server interface, add the dashboard [MySQL Replication](https://github.com/percona/grafana-dashboards/blob/master/dashboards/MySQL_Replication.json) with the graph for the heartbeat collection.
