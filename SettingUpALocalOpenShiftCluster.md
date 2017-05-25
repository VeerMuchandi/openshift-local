## Setting up a Local OpenShift Cluster

In this chapter we will use OpenShift Command Line Interface (CLI) to spin up a local cluster on your machine.

**Use Case:** As a developer, I want to develop my applications locally. In the world of container platform, I still want a local setup where I can deploy and test my code before I push the code to a cluster running in my enterprise data center.


### Using Minishift

Minishift launches a single node OpenShift cluster on your desktop by launching a virtual machine. You can use it on Windows, Mac OS, or Linux. Minishift spins up a VM for you using [libmachine](https://github.com/docker/machine/tree/master/libmachine) and uses `oc cluster up` to run OpenShift. 

**Opensource version:** Minishift opensource version runs OpenShift Origin and the documentation is here [https://www.openshift.org/minishift/](https://www.openshift.org/minishift/). 

If you plan to use this version, understand the pre-requisites and setup minishift as explained [here](https://docs.openshift.org/latest/minishift/getting-started/installing.html#installing-instructions).  

When you use the opensource version `minishift version` will indicate the same. 

```
$ minishift version
Minishift version: 1.0.1
```

**CDK:** Red Hat Container Development Kit 3 (CDK 3) is the Red Hat supported version of Minishift. This is currently beta as described in this [blog](https://developers.redhat.com/blog/2017/02/28/using-red-hat-container-development-kit-3-beta/). You can download the CDK 3 beta from [https://developers.redhat.com/products/cdk/download/](https://developers.redhat.com/products/cdk/download/). CDK Minishift runs OpenShift Container Platform supported by Red Hat. CDK registers the VM to Red Hat subscription manager, and hence you will need Red Hat credentials such as [Red Hat developer subscription](https://developers.redhat.com/blog/2016/03/31/no-cost-rhel-developer-subscription-now-available/).

If you set up CDK, then the `minishift version` will show up as follows.

```
$ minishift version
Minishift version: 1.0.0
CDK Version: 3.0.0
```


Using CDK involves two steps a) Setup CDK and b)Running Minishift. If you are using opensource version, you can skip setup-cdk part. 

**Step 1:** Run setup-cdk to download minishift-rhel7.iso. This command will extract and put the ISO and the `oc` binary in the ~/.minishift/cache directory and create the configuration file with defaults for properties along with the cdk marker file in ~/.minishift/cdk.

```
$ minishift setup-cdk
Setting up CDK 3 on host using '/Users/veer/.minishift' as Minishift's home directory
The MINISHIFT_HOME directory '/Users/veer/.minishift' exists. Continuing will delete any existing VM and all other data in this directory. Do you want to continue? [y/N]
y
Copying minishift-rhel7.iso to '/Users/veer/.minishift/cache/iso/minishift-rhel7.iso'
Copying oc to '/Users/veer/.minishift/cache/oc/v3.5.5.5/oc'
Creating configuration file '/Users/veer/.minishift/config/config.json'
Creating marker file '/Users/veer/.minishift/cdk'
Default add-ons anyuid, admin-user, eap-templates, fuse-templates installed
Default add-ons anyuid, admin-user, eap-templates, fuse-templates enabled
```

**Step 2:** Use your Red Hat credentials to start minishift CDK. While you can pass these on command line, the better approach, IMO, is to set the values for MINISHIFT_USERNAME and MINISHIFT_PASSWORD (as of now, it doesn't prompt you for password!!). This step will download openshift container platform image from registry.access.redhat.com.


```
$ export MINISHIFT_USERNAME=<<your RHN username>>
$ export MINISHIFT_PASSWORD=<<your RHN password>>
$ minishift start 
Starting local OpenShift cluster using 'xhyve' hypervisor...
Registering machine using subscription-manager
-- Checking OpenShift client ... OK
-- Checking Docker client ... OK
-- Checking Docker version ... OK
-- Checking for existing OpenShift container ... OK
-- Checking for openshift3/ose:v3.5.5.5 image ... 
   Pulling image openshift3/ose:v3.5.5.5
   Pulled 1/4 layers, 25% complete
   Pulled 1/4 layers, 25% complete
   Pulled 1/4 layers, 25% complete
   Pulled 2/4 layers, 50% complete
   Pulled 3/4 layers, 75% complete
   Pulled 4/4 layers, 100% complete
   Extracting
   Image pull complete
-- Checking Docker daemon configuration ... OK
-- Checking for available ports ... OK
-- Checking type of volume mount ... 
   Using nsenter mounter for OpenShift volumes
-- Creating host directories ... OK
-- Finding server IP ... 
   Using 192.168.64.2 as the server IP
-- Starting OpenShift container ... 
   Creating initial OpenShift configuration
   Starting OpenShift using container 'origin'
   Waiting for API server to start listening
   OpenShift server started
-- Adding default OAuthClient redirect URIs ... OK
-- Installing registry ... OK
-- Installing router ... OK
-- Importing image streams ... OK
-- Importing templates ... OK
-- Login to server ... OK
-- Creating initial project "myproject" ... OK
-- Removing temporary directory ... OK
-- Checking container networking ... OK
-- Server Information ... 
   OpenShift server started.
   The server is accessible via web console at:
       https://192.168.64.2:8443

   You are logged in as:
       User:     developer
       Password: developer

   To login as administrator:
       oc login -u system:admin
```

Note that openshift master is started at `https://192.168.64.2:8443` and the user credentials to login are also provided. Make a note of these.

If you are using open source minishift, you can directly start your cluster as follows

```
$ minishift start
Starting local OpenShift cluster using 'xhyve' hypervisor...
-- Checking OpenShift client ... OK
-- Checking Docker client ... OK
-- Checking Docker version ... OK
-- Checking for existing OpenShift container ... 
   Deleted existing OpenShift container
-- Checking for openshift/origin:v1.5.0-rc.0 image ... OK
-- Checking Docker daemon configuration ... OK
-- Checking for available ports ... OK
-- Checking type of volume mount ... 
   Using Docker shared volumes for OpenShift volumes
-- Creating host directories ... OK
-- Finding server IP ... 
   Using 192.168.64.2 as the server IP
-- Starting OpenShift container ... 
   Starting OpenShift using container 'origin'
   Waiting for API server to start listening
   OpenShift server started
-- Removing temporary directory ... OK
-- Checking container networking ... OK
-- Server Information ... 
   OpenShift server started.
   The server is accessible via web console at:
       https://192.168.64.2:8443

   To login as administrator:
       oc login -u system:admin
```

**Note** that in this case it runs openshift/origin.


**Step 3** Stop and Restart the cluster

To stop the cluster use

```
$ minishift stop
Stopping local OpenShift cluster...
Unregistering machine
Cluster stopped.
```

When you restart the cluster, any applications you added to the cluster are preserved by default.

By default the cluster is given only 2GB of memory which ,in my experience, is very insufficient. You would want to change the CPU and Memory resources per your needs. However, the only way you can change the configuration is by first deleting the previous cluster and start a new one.

```
$ minishift delete
``` 

Now, start a new cluster with your own memory and cpu settings

```
$ minishift start --metrics --cpus 4 --memory=8000
Starting local OpenShift cluster using 'xhyve' hypervisor...
Registering machine using subscription-manager
-- Checking OpenShift client ... OK
-- Checking Docker client ... OK
-- Checking Docker version ... OK
-- Checking for existing OpenShift container ... OK
-- Checking for openshift3/ose:v3.5.5.5 image ... 
   Pulling image openshift3/ose:v3.5.5.5
   Pulled 1/4 layers, 25% complete
   Pulled 1/4 layers, 25% complete
   Pulled 1/4 layers, 25% complete
   Pulled 2/4 layers, 50% complete
   Pulled 3/4 layers, 75% complete
   Pulled 4/4 layers, 100% complete
   Extracting
   Image pull complete
-- Checking Docker daemon configuration ... OK
-- Checking for available ports ... OK
-- Checking type of volume mount ... 
   Using nsenter mounter for OpenShift volumes
-- Creating host directories ... OK
-- Finding server IP ... 
   Using 192.168.64.2 as the server IP
-- Starting OpenShift container ... 
   Creating initial OpenShift configuration
   Starting OpenShift using container 'origin'
   Waiting for API server to start listening
   OpenShift server started
-- Adding default OAuthClient redirect URIs ... OK
-- Installing registry ... OK
-- Installing router ... OK
-- Installing metrics ... OK
-- Importing image streams ... OK
-- Importing templates ... OK
-- Login to server ... OK
-- Creating initial project "myproject" ... OK
-- Removing temporary directory ... OK
-- Checking container networking ... OK
-- Server Information ... 
   OpenShift server started.
   The server is accessible via web console at:
       https://192.168.64.2:8443

   The metrics service is available at:
       https://metrics-openshift-infra.192.168.64.2.nip.io

   You are logged in as:
       User:     developer
       Password: developer

   To login as administrator:
       oc login -u system:admin

Applying addon admin-user:.user "admin" created
.cluster role "cluster-admin" added: "admin"

Applying addon anyuid:.

Applying addon eap:.template "eap70-basic-s2i" created
.template "eap70-mysql-persistent-s2i" created
.imagestream "jboss-webserver30-tomcat7-openshift" created
imagestream "jboss-webserver30-tomcat8-openshift" created
imagestream "jboss-eap64-openshift" created
imagestream "jboss-eap70-openshift" created
imagestream "jboss-decisionserver62-openshift" created
imagestream "jboss-decisionserver63-openshift" created
imagestream "jboss-processserver63-openshift" created
imagestream "jboss-datagrid65-openshift" created
imagestream "jboss-datavirt63-openshift" created
imagestream "jboss-amq-62" created
imagestream "redhat-sso70-openshift" created


Applying addon fuse:.imagestream "fis-java-openshift" created
imagestream "fis-karaf-openshift" created
.template "s2i-karaf2-camel-amq" created
.template "s2i-karaf2-camel-log" created
.template "s2i-karaf2-camel-rest-sql" created
.template "s2i-karaf2-cxf-rest" created
.template "s2i-spring-boot-camel-amq" created
.template "s2i-spring-boot-camel-config" created
.template "s2i-spring-boot-camel-drools" created
.template "s2i-spring-boot-camel-infinispan" created
.template "s2i-spring-boot-camel-rest-sql" created
.template "s2i-spring-boot-camel-teiid" created
.template "s2i-spring-boot-camel" created
.template "s2i-spring-boot-camel-xml" created
.template "s2i-spring-boot-cxf-jaxrs" created
.template "s2i-spring-boot-cxf-jaxws" created
```

**Step 4:** Quickly test by deploying a Container image

Once the cluster is up you are logged in automatically. If you want to login at anytime, you run `oc login -u developer` and enter password `developer`.

Check the project you are in

```
$ oc project
Using project "myproject" on server "https://192.168.64.2:8443"
```

Create an application from an existing container image `veermuchandi/welcome` from docker hub.

```
$ oc new-app veermuchandi/welcome
--> Found Docker image f5756b7 (3 months old) from Docker Hub for "veermuchandi/welcome"

    * An image stream will be created as "welcome:latest" that will track this image
    * This image will be deployed in deployment config "welcome"
    * Port 8080/tcp will be load balanced by service "welcome"
      * Other containers can access this service through the hostname "welcome"

--> Creating resources ...
    imagestream "welcome" created
    deploymentconfig "welcome" created
    service "welcome" created
--> Success
    Run 'oc status' to view your app.
    
```

Create a route

```
$ oc expose svc welcome
route "welcome" exposed

$ oc get route
NAME      HOST/PORT                                 PATH      SERVICES   PORT       TERMINATION
welcome   welcome-myproject.192.168.64.2.nip.io             welcome    8080-tcp 
```

Note that the URL is using `nip.io` based domain name. This will resolve back to your machine. So, not just you, your colleagues on the same network would be able to access the application that you just deployed!!
	
Test the application. You can test it from the browser as well.
```
curl welcome-myproject.192.168.64.2.nip.io
 Welcome to OpenShift 3 !!!

```

Congratulations!! Your application is now running. We are ready to get to the next topic.
















