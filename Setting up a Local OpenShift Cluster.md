##Setting up a Local OpenShift Cluster

In this chapter we will use OpenShift Command Line Interface (CLI) to spin up a local cluster on your machine.

**Use Case:** As a developer, I want to develop my applications locally. In the world of container platform, I still want an local setup where I can deploy and test my code before I push the code to a cluster running in my enterprise data center.

Complete documentation for Local Cluster Management is all on this github [https://github.com/openshift/origin/blob/master/docs/cluster_up_down.md](https://github.com/openshift/origin/blob/master/docs/cluster_up_down.md). While, I hate to repeat the same stuff, I had to write it again for a few additional tips that I used to overcome some issues I faced. 

OpenShift Command (oc), beginning v1.3+ or Red Hat supported oc v3.3+ provides an option to quickly spin up a local OpenShift Cluster on your workstation. OpenShift is available as an all-in-one container image that includes master, node, registry, router, a few image streams and default templates. So when you bring up the cluster with the `oc` command, it will download the image from registry and instantiates it to create your own local OpenShift environment.



**Step 1** : Install Docker and OpenShift Client

[Mac Users with Docker for Mac](https://github.com/openshift/origin/blob/master/docs/cluster_up_down.md#macos-with-docker-for-mac)

[Windows Users with Docker for Windows](https://github.com/openshift/origin/blob/master/docs/cluster_up_down.md#windows-with-docker-for-windows)

[Mac Users with Docker Toolbox](https://github.com/openshift/origin/blob/master/docs/cluster_up_down.md#mac-os-x-with-docker-toolbox)

[Windows Users with Docker Toolbox](https://github.com/openshift/origin/blob/master/docs/cluster_up_down.md#windows-with-docker-toolbox)

**Additional Instructions/Tips:**

* **Verify Docker is running**: Once you install *Docker for Mac*, run `docker version` to verify it's running. If it does not respond then run	
```
$ unset ${!DOCKER*}
```		
and try again.

* **Download `oc`** 
    
    Install the `oc v1.4.1` binary using homebrew with: `brew install openshift-cli`

	OR 

	Download from here [https://github.com/openshift/origin/releases/download/v1.4.1/openshift-origin-client-tools-v1.4.1-3f9807a-mac.zip](https://github.com/openshift/origin/releases/download/v1.4.1/openshift-origin-client-tools-v1.4.1-3f9807a-mac.zip)
   and copy `oc` to `/usr/local/bin` or anywhere else in your PATH.
   
   OR
   
   If you have access to RedHat Customer Portal, you can download supported version of `oc v3.4.1` from here [https://access.redhat.com/downloads/content/290](https://access.redhat.com/downloads/content/290)



**Step 2**:  Start OpenShift Cluster

To start an OpenShift cluster run

```
$ oc cluster up
-- Checking OpenShift client ... OK
-- Checking Docker client ... OK
-- Checking Docker version ... OK
-- Checking for existing OpenShift container ... OK
-- Checking for openshift/origin:v1.4.1 image ... 
   Pulling image openshift/origin:v1.4.1
   Pulled 1/3 layers, 36% complete
   Pulled 2/3 layers, 88% complete
   Pulled 3/3 layers, 100% complete
   Extracting
   Image pull complete
-- Checking Docker daemon configuration ... OK
-- Checking for available ports ... OK
-- Checking type of volume mount ... 
   Using Docker shared volumes for OpenShift volumes
-- Creating host directories ... OK
-- Finding server IP ... 
   Using 127.0.0.1 as the server IP
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
-- Server Information ... 
   OpenShift server started.
   The server is accessible via web console at:
       https://127.0.0.1:8443

   You are logged in as:
       User:     developer
       Password: developer

   To login as administrator:
       oc login -u system:admin
```

Note that this pulls the image `openshift/origin:v1.4.1` from docker hub and then installs it. This is the community version i.e, OpenShift Origin.

If you were using `oc v3.4.1`, the Red Hat supported version, then the default image downloaded will be from `registry.access.redhat.com/openshift3/ose:v3.4.1` which is same as the RedHat's OpenShift Container Platform.

With Docker on Mac, OpenShift master url comes up as `https://127.0.0.1:8443`. If you use Docker Tools, then this will be set to ip address assigned to your box instead of `127.0.0.1`.



**Step 3:** Quickly test by deploying a Container image

Once the cluster is up you are logged in automatically. If you want to login at anytime, you run `oc login -u developer` and enter password `developer`.

Check the project you are in

```
$ oc project
Using project "myproject" on server "https://127.0.0.1:8443"
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
welcome   welcome-myproject.192.168.43.252.xip.io             welcome    8080-tcp 
```

Note that the URL is using `xip.io` based domain name. This will resolve back to your machine. So, not just you, your colleagues on the same network would be able to access the application that you just deployed!!

Test the application. You can test it from the browser as well.
```
curl welcome-myproject.192.168.43.252.xip.io
 Welcome to OpenShift 3 !!!

```


**Step 4:** Shutdown the cluster
To bring down the cluster run

```
$ oc cluster down
```
So easy, right!!


**Step 5:** Understand the options

Start the cluster again by running `oc cluster up`.

Now try to find your application that you deployed earlier

```
$ oc get pods -n myproject
No resources found.
```
Well, the application you deployed earlier is not there anymore. It starts from the beginning.

But, you may be thinking, is there a way to preserve the state between restarts?

To understand the options while running `oc cluster up`

```
$ oc cluster up --help
```

Look at these options

```
# Start OpenShift and preserve data and config between restarts
  oc cluster up --host-data-dir=/mydata --use-existing-config
```
If you provide `host-data-dir` directory, you can preserve your applications deployed between the cluster restarts. 

If you use the option `use-existing-config` you can preserve configurations between the restarts. Example, the `domain-name` assigned will be preserved.

**Why is this relevant?** 	
Let us say you go to a different location and join a different network. Your computer gets a different ip address, and hence the URL `welcome-myproject.192.168.43.252.xip.io` won't work. 

In such a case, you will have to recreate your routes to get them to work. If you preserve the configuration, the routes you create will continue to get the same ip address `192.168.43.252.xip.io` even though your network has changed. So you may want to not use `use-existing-config` when the network changes.

Look at these options

```
# Use a different set of images
  oc cluster up --image="registry.example.com/origin" --version="v1.1"
```
They allow you to select a specific version of OpenShift.

I use the following command to bring up my cluster:

```
oc cluster up --image=registry.access.redhat.com/openshift3/ose \
--version= v3.4.1.2-2 --host-data-dir=/Users/veer/occlusterdata/hostdata \
--host-config-dir=/Users/veer/occlusterdata/hostconfig \
--use-existing-config

```
Here I am using OpenShift Container Platform  v3.4.1.2-2 from Red Hat Inc. I created the folders `occlusterdata/hostdata` and `occlusterdata/hostconfig` to save host data and host configurations respectively. I am referring these two directories so that the data is saved and a location of my choice. If I change the network, I don't use --use-existing-config option.



For Windows running Docker for Windows, I use the following:

```
oc cluster up --image=registry.access.redhat.com/openshift3/ose --version=v3.4.1.2-2 --host-config-dir=C:\Users\veer\openshift\hostconfig --host-data-dir=C:\Users\veer\openshift\hostdata
```
In order to use folders created on C: drive you will have to share it in the Docker for Mac `Settings` -> `Shared Drives` and `Apply`. This will restart docker. **Note** If there is firewall it may prevent the share.


If you are running Docker Tools, I suggest creating docker-machine in advance and then use it with `oc cluster up` as shown below

```
docker-machine create -d virtualbox --virtualbox-memory 8192 --virtualbox-cpu-count 4 --engine-insecure-registry 172.30.0.0/16 openshift

eval $(docker-machine env openshift)

oc cluster up --docker-machine=openshift
oc cluster down --docker-machine=openshift
   
``` 












