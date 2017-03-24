# MySQL Replication

## [Setting Up MySQL Replication Without the Downtime](https://plusbryan.com/mysql-replication-without-downtime)

### We will use Docker-Compose to test this method.

### This will cover two scenarios:

1.  Two MySQL images(container).
  *   First image will be the Master MySQL.
  *   Second MySQL image will be the Slave MySQL.

2.  One MySQL image(container) and one localhost MySQL.
  *   Localhost MySQL's will be the Master MySQL.
  *   MySQL image will be the Slave MySQL.
 
---

## Two MySQL Images(container):

### Here's the main steps to complete this scenario:
1. Create and configure docker-compose.yml.
 * Two [MySQL](https://hub.docker.com/_/mysql/) images.
 * Networks setting.

2. Start up docker-composer.

3. Login to master MySQL and create dummy table and data.

4. Follow the steps in [blog](https://plusbryan.com/mysql-replication-without-downtime) to set up replication
 * Dump one DB instead of dumping all DBs.

---

### Step 1
Will start by first creating **docker-compose.yml** file and configure it as the following:

Under *services* will add two MySQL images. Will call the first image *master* and the second one *slave*.

- *master* and *slave* will be almost identical.

#### **Master Image/Container Configuration:**
Under the *environment* will pass two variables:

1. Set user root password.

2. Create a new DB.


```
  environment:
```

```
   - MYSQL_ROOT_PASSWORD=root
```

```
   - MYSQL_DATABASE=master_db
```

Under *volumes* will create a mount data volume.
(Will use it to pass my.cnf and dump file)


```
  volumes:
```

```
      - ./dbconfig-m:/etc/mysql/dbconfig
```

Under *networks* will link this container to docker-compose network and will set the *master*'s ip to "177.18.0.10".

```
  networks:
```

```
   app_net:
```

```
     ipv4_address: 177.18.0.10
```

#### **Slave Image/Container Configuration:**
Under the *environment* will pass two variables:

1. Set user root password.

2. Create a new DB. (Should be same as in master DB to be able to dump it)


```
  environment:
```

```
   - MYSQL_ROOT_PASSWORD=root
```

```
   - MYSQL_DATABASE=master_db
```

Under *volumes* will create a mount data volume.
(Will use it to pass my.cnf and dump file)


```
  volumes:
```

```
      - ./dbconfig-s:/etc/mysql/dbconfig
```

Under *links* will link *slave* container to *master* container.

```
  links:
```

```
   - "master:mysql"
```

Under *networks* will link this container to docker-compose network and will set the *slave*'s ip to "177.18.0.11".

```
  networks:
```

```
   app_net:
```

```
     ipv4_address: 177.18.0.11
```

#### **[Networks Configuration](https://runnable.com/docker/docker-compose-networking):**
Under the *config* will set:

1. Subnet.

2. Gateway.


```
 config:
```

```
  - subnet: 177.18.0.0/24
```

``` 
  gateway: 177.18.0.1
```

---

### Step 2
This step is pretty forword, we just start up docker-compose and make sure that everything is working.

**First** in the "terminal" we "cd" into the directory where *docker-compose.yml* is located then start it up in the background (to be able to use terminal later).

```
~$ cd two-mysql-images
```

"-d" option will run the container in the background


```
~/two-mysql-images$ docker-compose up -d
```

**Second** we check if the container is running and there's no errors.


```
~/two-mysql-images$ docker-compose logs
```

---


### Step 3
Login to master MySQL and create dummy table and data.

**1** we access master container

```
~/two-mysql-images$ docker exec -it master bash
```

**2** we login to MySQL but we have to wait until all setup are done since we are running in the background mood.


```
root@f72086644b4f:/# mysql -u root -p
```

```
Enter password: root
```

If login failed and give socket error then we have to wait a bit more until the setup in the background is finished. 


**3** now after we logged in we switch to *master_db* and then create table and insert some data.


```mysql
create table books
( bookid int not null primary key auto_increment,
  title varchar(100) not null,
  author varchar(100) not null
) engine = innodb;
```

```mysql
INSERT INTO books VALUES
(null, 'Harry Potter and the Goblet of Fire', 'J. K. Rowling'),
(null, 'Harry Potter and the Half-Blood Prince', 'J. K. Rowling'),
(null, 'Wind in the Willows', 'Kenneth Grahame'),
(null, 'Great Expectations', 'Charles Dickens'),
(null, 'A Christmas Carol', 'Charles Dickens'),
(null, 'Knot and Crosses', 'Ian Rankin'),
(null, 'The Hanging Garden', 'Ian Rankin');
```

---

### Step 4
We simply follow the steps in the [blog](https://plusbryan.com/mysql-replication-without-downtime) to set up mysql replication without downtime.

However, there are a few changing we will do to make it dump only one DB insteal of dumping all DBs. Also, since we are in docker containers we will copy the dumped file through the mounted volumes between the two containers not through "scp".

**Note:** when trying to restart mysql server through "service mysql restart" that will stop docker container and log you out of the container(lose access). simply just run again "docker-compose up -d" and then access again to master or slave container (depend on the setp) and that will make the same effect as the normal service restart.

**1** when creating the dump will use the following to only dump the wanted DB in the "dbconfig-m" directory.

```
root@f72086644b4f:#/ mysqldump -u root -p --skip-lock-tables --single-transaction --flush-logs --hex-blob --master-data=2 master_db > /etc/mysql/dbconfig-m/dump.sql
```

**2** now through the localhost simply copy the dump file from "dbconfig-m" to "dbconfig-s"

```
~/two-mysql-images$ cp -a ./dbconfig-m/dump.sql.gz ./dbconfig-s/
```

**3** after reaching configure and restart slave step, now we import dump file.
(at this point we will be inside slave container). Also, here we need to add the wanted DB name to be able to make the dump.

```
root@6ece2160f73c:/# mysql -u root -p master_db < /etc/mysql/dbconfig-s/dump.sql
```

```
Enter password: root
```

Contine the blog steps.

---

**The End**

At this point, we should have a working master/slave replication.

We can test it by adding new data, deleting data, create new table or droping table in master and we should see the affect immediately in slave.

Also, we can broke the replecation by modifing slave DB and then fix it as mentioned at the end of blog by

```
STOP SLAVE;SET GLOBAL SQL_SLAVE_SKIP_COUNTER = 1;START SLAVE;
```

---

## One MySQL image(container) and one localhost MySQL:

### Here's the main steps to complete this scenario:
1. Create and configure docker-compose.yml.
 * One [MySQL](https://hub.docker.com/_/mysql/) image.
 * Networks setting.

2. Start up docker-composer.

3. Follow the steps in [blog](https://plusbryan.com/mysql-replication-without-downtime) to set up replication
 * Dump one DB instead of dumping all DBs.


**Note:** This is exactly the same as in the first scenario apart from a few small changes.
Also, we will suppose that we already have a wanted DB in our localhost, that will be represent as *master_db* and that it have tables and data, so we don't need to create a DB and insert some dummy data.

---

### Step 1
The **docker-compose.yml** file will have only *slave and networks* instead of having *master, slave and networks*.
Also, *slave and networks* are exactly the same but will delete the **link** part from slave since we don't have *master* container.

### Step 2
Again exactly the same.

### Step 3
We simply follow the steps in the [blog](https://plusbryan.com/mysql-replication-without-downtime) to set up mysql replication without downtime.

However, there are a few changing we will do to make it dump only one DB insteal of dumping all DBs. Also, since we are in docker container we will copy the dumped file through the mounted volume between localhost and slave (dbconfig-s) not through "scp".

**Note:** when trying to restart mysql server through "service mysql restart" that will stop salve container and log you out of the container(lose access). simply just run again "docker-compose up -d" and then access again to salve container and that will make the same effect as the normal service restart.

**1** when creating the dump will use the following to only dump the wanted DB in the "localhost-master" directory.

```
~/one-mysql-image$ mysqldump -u root -p --skip-lock-tables --single-transaction --flush-logs --hex-blob --master-data=2 master_db > ./localhost-master/dump.sql
```

**2** now through the localhost simply copy the dump file from "localhost-master" to "dbconfig-s"

```
~/one-mysql-image$ cp -a ./localhost-master/dump.sql.gz ./dbconfig-s/
```

**3** after reaching configure and restart slave step, now we import dump file.
(at this point we will be inside slave container). Also, here we need to add the wanted DB name to be able to make the dump.

```
root@6ece2160f73c:/# mysql -u root -p master_db < /etc/mysql/dbconfig-s/dump.sql
```

```
Enter password: root
```

Contine the blog steps.

---

**The End**

