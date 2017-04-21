## Connecting your application to a local database

In this exercise we will add a backend database to the previous deployed SpringBoot application.

**Use Case**: As a developer, I want my own local database running on my private OpenShift cluster and I want to connect my application to this database.


We created the SpringBoot application in the previous lab. Let's just test one more thing. Try your application url with `/dbtest` (http://bootapp-myproject.192.168.0.102.xip.io/dbtest) extension and it should list you some data from the HSQLDB configured within the same app.

Let's take a moment to understand how this application is connecting to the HSQLDB. Look at the `application.properties` file in the code [https://github.com/RedHatWorkshops/spring-sample-app/blob/master/application.properties](https://github.com/RedHatWorkshops/spring-sample-app/blob/master/application.properties) and we have configured these spring datasource variables to use hsqldb.

```
spring.datasource.platform=hsqldb
spring.datasource.url= jdbc:hsqldb:file:/opt/app-root/src/mydb;shutdown=true
spring.datasource.username=user
spring.datasource.password=password
```

Of course, [https://github.com/RedHatWorkshops/spring-sample-app/blob/master/pom.xml](https://github.com/RedHatWorkshops/spring-sample-app/blob/master/pom.xml) has the required dependencies. So springboot is able to create the in memory database.

Where is the data coming from? See these two files:

* [https://github.com/RedHatWorkshops/spring-sample-app/blob/master/src/main/resources/schema-hsqldb.sql](https://github.com/RedHatWorkshops/spring-sample-app/blob/master/src/main/resources/schema-hsqldb.sql) is creating the schema
* [https://github.com/RedHatWorkshops/spring-sample-app/blob/master/src/main/resources/data-hsqldb.sql](https://github.com/RedHatWorkshops/spring-sample-app/blob/master/src/main/resources/data-hsqldb.sql) is adding data.

This is the data displayed when you invoke dbtest endpoint. Pretty straight forward.. isn't it!!


**Step 1: Add a MySQL database to the project**

Go to WebConsole	
Make sure you are in `myproject`, the same project where you created the `bootapp` before.	
Click on `AddToProject`.		
Find `mysql-ephemeral` in data stores
and deploy the database container with following values assigned to the parameters.	


**Database Service Name:** mysql	
**MySQL Connection Username:** user		
**MySQL Connection Password:** password		
**MySQL Database Name:** sampledb	

Click on the `Create` button and within a few min or two your MySQL database pod should be up and running.

Also note the following message that displays after you press the `Create` button.

```
The following service(s) have been created in your project: mysql.

       Username: user
       Password: password
  Database Name: sampledb
 Connection URL: mysql://mysql:3306/

```



**Step 2: Configuring database connection params**

The simplest way is to edit `application.properties` file to the values you noted in the last step i.e.

```
spring.datasource.platform=mysql
spring.datasource.url= jdbc:mysql://mysql:3306/sampledb?useSSL=false
spring.datasource.username=user
spring.datasource.password=password
```

But wait, that requires you to rebuild he code and deploy. I know what you are thinking - *This is not a code change, this is a configuration change. Why should I rebuild the code?*

So, is there another way where I don't have to build the code but just push these configuration changes?.. **Yes** Let's use a `ConfigMap`.

ConfigMaps allow you to keep the configuration artifacts decoupled from the image content. More details [here](https://docs.openshift.com/container-platform/3.3/dev_guide/configmaps.html). 

Lets go to CLI.

Change to `myproject` project
```
oc project myproject
```

Create a new file with name `application.properties` with the following content.

```
spring.datasource.platform=mysql
spring.datasource.url= jdbc:mysql://mysql:3306/sampledb?useSSL=false
spring.datasource.username=user
spring.datasource.password=password
```
You need to make sure that you substitute the correct values you noted in the last step when you wre creating the service. **Be extra-careful.. read instructions below.**

Specifically note the datasource url. It is in the following format:

`spring.datasource.url = jdbc:<<databasetype>>://<<service-host>>:<<service-port>>/<<dbname>>?useSSL=false`

You can replace `service-host` by the IP address of your MySQL service or the Service name.
In the above example, I am using the service name for example `mysql`. I can further qualify it with the project name as well `mysql.myproject` in which case `mysql` is the name of the service and `myproject` is the project name. This is a fully qualified way to let your application do service discovery in OpenShift (it uses SkyDNS).

Now let's create a ConfigMap with name `app-props` by running

```
$ oc create configmap app-props --from-file=application.properties
configmap "app-props" created
```

Try a couple of more commands

```
$ oc describe configmap app-props
Name:		app-props
Namespace:	spring-UserName
Labels:		<none>
Annotations:	<none>

Data
====
application.properties:	328 bytes
```

If you made a mistake you can always edit the ConfigMap using
```
oc edit configmap app-props
```

So far, we have created a ConfigMap in the project but your springboot application does not know how to use it.


**Step 3: Edit Deployment Configuration **

Now we will mount the ConfigMap so that the springboot application can use it. You can either edit from CLI or from WebConsole.

* If using WebConsole, navigate to `Applications`->`Deployments`->`bootapp`. The screen should show Deployment Configuration details for the `bootapp` application. On the right top, choose `Actions` dropdown and `Edit YAML`.

* If you are doing from CLI, you can run `oc edit dc bootapp`

Scroll down to container spec, that looks like this:
```
    spec:
      containers:
        -
          name: bootapp
          image: '172.30.85.130:5000/myproject/bootapp@sha256:79e188d712e1209b933870c7d064423af281f16b371fb5e5911dfb09a6867776'
          ports:
            -
              containerPort: 8080
              protocol: TCP
          resources:
          terminationMessagePath: /dev/termination-log
          imagePullPolicy: Always
      restartPolicy: Always
      terminationGracePeriodSeconds: 30
      dnsPolicy: ClusterFirst
      securityContext:
```
Note there could be multiple `spec`s in your DC.

We will now add a volume that points to our  ConfigMap right under `spec`. It is explained here [https://docs.openshift.com/container-platform/3.3/dev_guide/configmaps.html#configmaps-use-case-consuming-in-volumes](https://docs.openshift.com/container-platform/3.3/dev_guide/configmaps.html#configmaps-use-case-consuming-in-volumes)

```
spec:
  volumes:
    - name: app-props-volume
      configMap:
        name: app-props
```
**Be super-careful with indentation**

We will now add `volumeMount` to mount the `volume` that we just added into the pod. It should be right under the container `name:` as shown below.

```
      containers:
        -
          name: bootapp
          volumeMounts:
          - name: app-props-volume
            mountPath: /opt/app-root/src/config
```
**Be super-careful with indentation**

After the changes, the `template` section in the dc, should now look like this

```
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: bootapp
        deploymentconfig: bootapp
    spec:
      volumes:
        - name: app-props-volume
          configMap:
            name: app-props
      containers:
        -
          name: bootapp
          volumeMounts:
          - name: app-props-volume
            mountPath: /opt/app-root/src/config
          image: '172.30.85.130:5000/spring-veer/bootapp@sha256:79e188d712e1209b933870c7d064423af281f16b371fb5e5911dfb09a6867776'
          ports:
            -
              containerPort: 8080
              protocol: TCP
          resources:
          terminationMessagePath: /dev/termination-log
          imagePullPolicy: Always
      restartPolicy: Always
      terminationGracePeriodSeconds: 30
      dnsPolicy: ClusterFirst
      securityContext:
```


So what is this location `/opt/app-root/src/config`?

If you get into the terminal of the pod (you should know how to do this by now!) and run `pwd`, it will show that the `home` directory is `/opt/app-root/src`. If you copy the `application.properties` file in the `config` folder, SpringBoot will pick that first. Hence we mounted the folder `/opt/app-root/src/config`.

Save the changes and exit. If you now got the `Overview` page, you will see that the pod gets re-deployed. Yes, redeployed, not rebuilt (no S2I build process).

**Step 4: Verify the changes**

Once the deployment is complete		
* click on the pod circle		
* click on the pod name		
* get into the `Terminal` tab		
* verify that your `application.properties` are now available in the `config` folder		

```
sh-4.2$ ls config                                                                                                                                                 
application.properties                                                                                                                                            
sh-4.2$ cat config/application.properties                                                                                                                                                                                                             
spring.datasource.platform=mysql                                                                                                                                  
spring.datasource.url= jdbc:mysql://mysql:3306/sampledb?useSSL=false                                                                                  
spring.datasource.username=user                                                                                                                                   
spring.datasource.password=password                                                                                                                               

```

Note the contents of this file are what you added to the ConfigMap.

**Step 5: Test your application**

Go back to the `Overview` page. Click on your application url which would be something like `http://bootapp-myproject.192.168.0.102.xip.io/`

It will open a new tab and your running application will greet you

`Hello from bootapp-2-06a4b`

Now move back to your webconsole and watch the pod logs. You can also do this from CLI by running
```
oc logs -f bootapp-2-06a4b
```

Now access the application with the `/dbtest` extension - `http://bootapp-myproject.192.168.0.102.xip.io/dbtest`

It should show the data from your MySQL database.

```
Customers List


CustomerId: 2 Customer Name: Joe Mysql Age: 88
CustomerId: 3 Customer Name: Jack Mysql Age: 54
CustomerId: 4 Customer Name: Ann Mysql Age: 32
```

Where did this data come from? Look at
* [https://github.com/RedHatWorkshops/spring-sample-app/blob/master/src/main/resources/schema-mysql.sql](https://github.com/RedHatWorkshops/spring-sample-app/blob/master/src/main/resources/schema-mysql.sql) was used to initialize the MySQL database
* [https://github.com/RedHatWorkshops/spring-sample-app/blob/master/src/main/resources/data-mysql.sql](https://github.com/RedHatWorkshops/spring-sample-app/blob/master/src/main/resources/data-mysql.sql) was used to populate data. I added 'Mysql' as part of the names to make it easy ;)

Also note that your logs show the connection url, just to verify which database you are connecting to.

```
connection url: jdbc:mysql://mysql:3306/sampledb?useSSL=false
```

So far, have learnt how to set up a multi-tiered application and also to pass configuration information using ConfigMaps. 

Let's try a few more things.

**Step 6: Access your database and check it contents**

Find your running database pod.

```
$ oc get pods
NAME              READY     STATUS      RESTARTS   AGE
bootapp-1-build   0/1       Completed   0          2h
bootapp-2-build   0/1       Completed   0          1h
bootapp-3-7cztq   1/1       Running     0          23m
mysql-1-dieh1     1/1       Running     0          42m
```

Connect to the database pod. This will connect you to the database pod. 

**Note** You can skip these two steps and directly get into the terminal using web console.

```
$ oc rsh mysql-1-dieh1
sh-4.2$
```

Find all the environment variables in the container that start with MYSQL.

```
sh-4.2$ env | grep MYSQL
MYSQL_PREFIX=/opt/rh/rh-mysql56/root/usr
MYSQL_VERSION=5.6
MYSQL_DATABASE=sampledb
MYSQL_PASSWORD=password
MYSQL_PORT_3306_TCP_PORT=3306
MYSQL_PORT_3306_TCP=tcp://172.30.150.106:3306
MYSQL_SERVICE_PORT_MYSQL=3306
MYSQL_PORT_3306_TCP_PROTO=tcp
MYSQL_PORT_3306_TCP_ADDR=172.30.150.106
MYSQL_SERVICE_PORT=3306
MYSQL_USER=user
MYSQL_PORT=tcp://172.30.150.106:3306
MYSQL_SERVICE_HOST=172.30.150.106
```

So you have everything you need to connect to the database. Let's use them:

```
sh-4.2$ mysql -h $MYSQL_SERVICE_HOST -P$MYSQL_SERVICE_PORT -u$MYSQL_USER -p$MYSQL_PASSWORD
Warning: Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 548
Server version: 5.6.34 MySQL Community Server (GPL)

Copyright (c) 2000, 2016, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql>

```

Check the list of databases
```
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| sampledb           |
+--------------------+
2 rows in set (0.00 sec)
```

Switch to `sampledb` database

```
mysql> use sampledb
Database changed
```
Show tables

```
mysql> show tables;
+--------------------+
| Tables_in_sampledb |
+--------------------+
| customer           |
+--------------------+
1 row in set (0.00 sec)
```

Look at the table contents

```

mysql> select * from customer;
+---------+------------+-----+
| CUST_ID | NAME       | AGE |
+---------+------------+-----+
|       2 | Joe Mysql  |  88 |
|       3 | Jack Mysql |  54 |
|       4 | Ann Mysql  |  32 |
+---------+------------+-----+
3 rows in set (0.01 sec)
```
Exit mysql client and the pod

```
mysql> exit
Bye
sh-4.2$ exit
exit
```

So, that's how you can connect to the database and check the contents.

*Wait!, what if I want to use a client (such as Toad) from my local machine. I don't want to get into the pod.*
You are so greedy!
But yes, we can do that.

**Step 7: Port forward your database pod **

Let's port forward our database pod to your localhost.

```
$ oc port-forward mysql-1-dieh1 3306:3306
Forwarding from 127.0.0.1:3306 -> 3306
Forwarding from [::1]:3306 -> 3306
```

Now open another terminal or your SQL client and connect to the database at 127.0.0.1:3306

I have mysql client on my box, so I will connect this way

```
$ mysql -h127.0.0.1 -P3306 -uuser -ppassword
mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 645
Server version: 5.6.34 MySQL Community Server (GPL)

Copyright (c) 2000, 2016, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> use sampledb;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> select * from customer;
+---------+------------+-----+
| CUST_ID | NAME       | AGE |
+---------+------------+-----+
|       2 | Joe Mysql  |  88 |
|       3 | Jack Mysql |  54 |
|       4 | Ann Mysql  |  32 |
+---------+------------+-----+
3 rows in set (0.00 sec)

mysql>

```

Now you can end the port-forwarding by pressing `Ctrl+C`.

**Conclusion**: We deployed a database on the local cluster and learnt to connect the application to the database without rebuilding the application by using ConfigMaps. We also learnt to connect to the database by accessing the pod as well as using port-forwarding from pod to your workstation.




 


	