# Use AWS DMS to do database migration

## Overview
[AWS DMS(Database Migration Server)](https://aws.amazon.com/tw/dms/) is a convenient service for DB migration. It provided a simpler, faster and cheaper method to migrate your database. At the same time, keep your databases in sync while doing migration.  

## Scenario
In this lab, we will use AWS DMS (Database Migration Server) to migrate databases and compare with traditional database Export/Import. AWS DMS supports a lot of databases engine and Source/Target types, so we have several cases in this lab. For more convenient to test, we will use MySQL database to do migration.

## Case 1 - MySQL on EC2 migrate to another EC2

<p align="center">
	<img src="/case1-001.jpg" width="1000" height="1000%">
</p>

## Step by step



### Prepare a VPC

1. Create a **VPC** `DMS lab`.

2. Create two **Subnets** at **different AZs**.

3. Create a **IGW** and attach to the VPC.

4. Edit the **Route Table** to associate the two subnets.

5. Add route "**0.0.0.0/0**" and select the IGW for the Route Table.

6. Create a **Security Group** `sg-mysql` and enable port "**22**" and "**3306**".



### Create two EC2 Instances

1. Launch instance with `Amazon Linux AMI 2018.03.0(HVM)`.

2. Choose the type `t2.micro`.

3. Enter `2` instances and choose the networking configuration.

4. Paste the following lines in **User data**.

```
#!/bin/bash  
sudo yum update -y
sudo yum install mysql56-server -y
service mysqld start
```

5. Configure the storage size to **10** GB.

6. Select the **Security Group**.

7. **Review and Launch**.

8. Back to the EC2 Instances page and **rename** the 2 Instances.
Set one is `MySQL Source`, and set another is `MySQL Target`.

9. Allocate 2 **Elastic IPs** and associate it to the 2 Instances.



### Configure MySQL server and database

1. Login the Instance `MySQL Source` via SSH.

2. Execute the command line.

```
# sudo mysql_secure_installation
```

* Click `Enter` to skip it,
* Enter `Y` and enter a new password,
* Enter `Y`,
* Enter `N` to enable root login remotely,
* Enter `Y`,
* Enter `Y`.

3. Login local MySQL via command.

```
# mysql -u root -p
```

4. Enable root login remotely via password.

```
mysql> use mysql;
     > GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY 'yourpassword' WITH GRANT OPTION;
     > FLUSH PRIVILEGES;
```

5. Create a database `test`.

```
mysql> CREATE DATABASE test;
```

6. Try to import or insert 1 GB data into MySQL database.

```
# mysql -u [username] -p [database_name] < [.sql file]
```
```
mysql> INSERT INTO [table_name] ([column1],[column2]…) values ([value1],[value2]…);
```
```
mysql> load data local infile [.txt or .csv file] into [table_name];
```

7. Login the another instance `MySQL Target` via SSH and repeat **steps 2.** to **4.**



### Database Export and Import

1. Open another session and login `MySQL Source` via **SSH**.

2. **Export** database as a **.sql file** via command line.

```
# mysqldump -u root -p > database_source.sql --all-database
```

3. **Upload** the .sql file to a **S3 bucket**.

```
# aws s3 cp database_source.sql s3://[your-bucket-name]/ 
```

4. Open another session and login `MySQL Target` via **SSH**.

5. **Download** the .sql file from S3 bucket at `MySQL Target`.

```
#aws s3 cp s3://[your-bucket-name]/database_source.sql database_source.sql
```

6. **Import** the .sql file into database to MySQL via command line.

```
# mysql -u root -p [database_name] < database_source.sql
```

### Using DMS to do migration again

1. Delete the database on `MySQL Target` MySQL.

```
mysql> drop database AWS-DMS;
mysql> drop database test;
```

2. Enable MySQL **binary logs** at `MySQL Source`.

* Open a session and login ‘MySQL Source’ via SSH
* Modify the MySQL configuration

```
#vi /etc/my.cnf
```
>Post the following lines to the **[mysqld]** section and save it
```
	server_id=1
	log-bin=/var/lib/mysql/bin
	max_binlog_size=100M
	binlog_format=row
	expire_logs_days=1
	binlog_checksum=NONE
	binlog_row_image=FULL
	log_slave_updates=TRUE
```

* Restart the MySQL service

```
#Service mysqld restart
```

* Check the binary logs has been enabled

```
mysql> show binary logs;
```

3. Open AWS web console and link to **DMS** page.

4. Create a **Replication instances**.

* In the navigation pane, click **Replication Instance** and click **Create replication instance**.
* Enter `DBreplicaiton` into **Name** and **Description** column
* Choose Instance class `dms.t2.micro`
* Configure Allocated storage to `20` GB
* Choose the VPC `DMS lab`
* Expand **Advanced security and network configuration**
* Choose the **AZ** and **Security Group**
* Click **Create**

5. Configure the **Source endpoint** for `MySQL Source`.

* In the navigation pane, click **Endpoints** and click **Create endpoint**.
* Choose Endpoint type `Source Endpoint`
* Enter `MySQL-Source-endpoint` into **Endpoint identifier** column 
* Choose engine `MySQL`
* Enter the instance `MySQL Source` private IP into **Server name** and enter **port** `3306`
* Enter the DB **User name** and **Password**
* Expand **Test endpoint connection (optional)**
* Choose **VPC** `DMS lab` and **Replication instance** `DBreplicaiton` , then click **Run test**
* After test **status** displayed `successful`, Click **Create endpoint**

6. Repeat the **Step 5.** to create the **Target endpoint** for `MySQL Target`.

* Choose Endpoint type `Target Endpoint`
* Enter `MySQL-Target-endpoint` into **Endpoint identifier** column 

7. Create the **Database migration task**.

* In the navigation pane, click **Database migration tasks** and click **Create task**.
* Enter `EC2toEC2` into Task identifier
* Choose **Replication instance**, **Source database endpoint** and **Target database endpoint**
* Choose **Migration type** `Migrate existing data and replication ongoing changes`
* Uncheck the option `Start task on create`
* Check the option `Enable validation`
* Expand **Table mappings** and click **Add new selection rule**, Then choose a **Schema**
* Expand **Advanced task settings** and enter `AWS-DMS` into `Create control table in target using schema`
* Click **Create task**
* Wait the task **status** is `Ready`

8. Start the Database migration task.

* Prepare the two session.  
	>One is login MySQL database at instance `MySQL Source`, another is login MySQL database at instance `MySQL Target`.
* Select the task and click **Actions**, then select **Restart/Resume**
* After the task started, check the database has been created at `MySQL Target` database

```
mysql> show databases;
```

* **Update** a row at `MySQL Source` database during the task status is `Starting`. Then count how long will the data be sync to the `MySQL Target` database
* After the task **status** turn to `Load complete, replication ongoing`, repeat the **last step** again and compare the time cost.

### Don’t terminate your lab environment. We will use it to do migration at Case 2.

## Conclusion

Congratulations! You now have learned how to:
* Setup a MySQL server and basic operation on it
* Upload/Download file between EC2 and S3 bucket
* AWS DMS for EC2 DB migrating solution
* Understanding the different between AWS DMS and DB Export/Import
