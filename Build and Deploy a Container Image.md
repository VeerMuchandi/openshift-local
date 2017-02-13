## Build and Deploy a Container Image
 
In this chapter, we will learn to create a Container Image using a Dockerfile and deploying it to a Local OpenShift Cluster.

**Use Case:** As a developer, I want to build a container image and test it locally on my local OpenShift cluster before pushing to an external container registry.


**Prerequisites:**

*  You understand containers and played around with Docker
* You have a prior understanding of OpenShift Concepts i.e, pods, services, routes, deployment configuration, imagestream etc.
* `Git` client and `oc` client are installed on your workstation
* OpenShift Local cluster is up and running


**Step 1: Identify your local Registry Service IP**

First login as an administrator to your local OpenShift Environment and shift to the `default` project. 

Since this is your local cluster, you can switch over as administrator on your cluster. Yay!! you are the KING :)

```
oc login -u system:admin -n default
```

List the services in this project. It will show `registry`, `router` and `kubernetes` services

```
$ oc get svc
NAME              CLUSTER-IP       EXTERNAL-IP   PORT(S)                   AGE
docker-registry   172.30.42.182    <none>        5000/TCP                  10d
kubernetes        172.30.0.1       <none>        443/TCP,53/UDP,53/TCP     10d
router            172.30.231.168   <none>        80/TCP,443/TCP,1936/TCP   10d
```

Note the Cluster-IP assigned to the `docker-registry` service. In my case that is `172.30.42.182`. Also note that the registry service has exposed port `5000`.

**Step 2: Permissions to the user account**

We will allow the user `developer` to push to the container registry running on your local cluster. 
* Means the user will need `system:image-builder` role
* Also the user should be able to `docker login` to this registry and for this the user needs `system:registry` role

Also we will assign admin access to the user `developer` to the project named `myproject`. If you were using a different project, then make sure that the user `developer` is an admin on that project too. 

You are still logged in as `system:admin`. Take time to understand and run the following commands.

```
oadm policy add-role-to-user system:registry developer
oadm policy add-role-to-user system:image-builder developer
oadm policy add-role-to-user admin developer -n myproject
```


**Step 3: Switch back to developer**

Login as `developer` again with password `developer`.

```
$ oc login -u developer
Authentication required for https://127.0.0.1:8443 (openshift)
Username: developer
Password: 
Login successful.

You have one project on this server: "myproject"

Using project "myproject".

```

Find the client token assigned to you at login. We will use this to login to the container registry.

```
$ oc whoami -t
wCxMwgzJhQmDUnDDvYAzk4K6Zx0-3LLRrnzinZLLCls
```
Make a note of the resultant `token`

Now login to the container registry using Service IP and Port noted previously, token from the previous step, and as the user `developer`.

------------
**IMPORTANT IMPORTANT IMPORTANT**

Substitute your own `ServiceIP` and `token` values below. I am using mine 172.30.42.182.

**IMPORTANT IMPORTANT IMPORTANT**

----------

```
$ docker login -u developer -p wCxMwgzJhQmDUnDDvYAzk4K6Zx0-3LLRrnzinZLLCls 172.30.42.182:5000
Login Succeeded
```
Note the `login` success. Now you are good to push images to this container registry on your local cluster.


**Step 4: Build a Container Image from Dockerfile**

In this example, we will use a Dockerfile from the git repository `https://github.com/VeerMuchandi/time`.

Clone this git repository to your workstation.

```
git clone https://github.com/VeerMuchandi/time
```

This will create a directory with name `time`. Navigate to `./time/busybox` and notice that there is a `Dockerfile` and `init.sh`. Read the `Dockerfile` to understand its contents. 

```
cd ./time/busybox
$ ls
Dockerfile	init.sh

```

Run `docker build` to create a container image. Tag the image so that it can be pushed to your local container registry. 

-----

**IMPORTANT IMPORTANT IMPORTANT**

Substitute your own `ServiceIP` value below. I am using mine 172.30.42.182.

**IMPORTANT IMPORTANT IMPORTANT**

-----


```
$ docker build . -t 172.30.42.182:5000/myproject/bbtime 
Sending build context to Docker daemon 3.072 kB
Step 1 : FROM busybox
 ---> 7968321274dc
Step 2 : MAINTAINER Veer Muchandi veer@redhat.com
 ---> Using cache
 ---> f812b015fbd5
Step 3 : ADD ./init.sh ./
 ---> Using cache
 ---> 4b7b373f9bac
Step 4 : EXPOSE 8080
 ---> Using cache
 ---> 2e9d009d5a75
Step 5 : CMD ./init.sh
 ---> Using cache
 ---> d4797784432f
Successfully built d4797784432f
```

This will create a ContainerImage. You can verify by running `docker images`.

Now push this container image to your the registry on the local openshift cluster.

```
$ docker push 172.30.42.182:5000/myproject/bbtime
The push refers to a repository [172.30.42.182:5000/myproject/bbtime]
1c08423d52b5: Layer already exists 
38ac8d0f5bb3: Layer already exists 
latest: digest: sha256:8b2963f7b781da762d2ebbbbf8b900b60091b310f6d97bc3f94a73f29ac61879 size: 4246
```
Now we are ready to create our application using this container image.

**Step 5: Deploy your application and test**

Create the application using the container image and expose the service as a route.

```
$ oc new-app 172.30.42.182:5000/myproject/bbtime
--> Found Docker image d479778 (About an hour old) from 172.30.42.182:5000 for "172.30.42.182:5000/myproject/bbtime:latest"

    * This image will be deployed in deployment config "bbtime"
    * Port 8080/tcp will be load balanced by service "bbtime"
      * Other containers can access this service through the hostname "bbtime"
    * WARNING: Image "172.30.42.182:5000/myproject/bbtime:latest" runs as the 'root' user which may not be permitted by your cluster administrator

--> Creating resources ...
    deploymentconfig "bbtime" created
    service "bbtime" created
--> Success
    Run 'oc status' to view your app.
    
$ oc new-app 172.30.42.182:5000/myproject/bbtime
W0209 23:03:58.164810   76202 dockerimagelookup.go:217] Docker registry lookup failed: Internal error occurred: Get https://172.30.42.182:5000/v2/: http: server gave HTTP response to HTTPS client
W0209 23:03:58.197115   76202 newapp.go:338] Could not find an image stream match for "172.30.42.182:5000/myproject/bbtime:latest". Make sure that a Docker image with that tag is available on the node for the deployment to succeed.
--> Found Docker image d479778 (About an hour old) from 172.30.42.182:5000 for "172.30.42.182:5000/myproject/bbtime:latest"

    * This image will be deployed in deployment config "bbtime"
    * Port 8080/tcp will be load balanced by service "bbtime"
      * Other containers can access this service through the hostname "bbtime"
    * WARNING: Image "172.30.42.182:5000/myproject/bbtime:latest" runs as the 'root' user which may not be permitted by your cluster administrator

--> Creating resources ...
    deploymentconfig "bbtime" created
    service "bbtime" created
--> Success
    Run 'oc status' to view your app.

$ oc expose svc/bbtime
route "bbtime" exposed

```
Check the route and test it.

```
$ oc get route
NAME      HOST/PORT                               PATH      SERVICES   PORT       TERMINATION
bbtime    bbtime-myproject.192.168.0.102.xip.io             bbtime     8080-tcp

$ curl http://bbtime-myproject.192.168.0.102.xip.io
Fri Feb 10 04:04:02 UTC 2017
```

You have learnt how to build a Container Image and test it locally without pushing into an external registry!!

**Bonus Points**

Make a small change to the `init.sh` script. As an example, I added a text "Current Date and Time:".

```
$ cat init.sh
#!/bin/sh
while true; do echo -e "HTTP/1.1 200 OK\n\n Current Date and Time: $(date)" | nc -ll -p 8080; done
```

Now build the container image again. 	
**Tip** `docker build` like before.

Push the image again	
**Tip** `docker push` like before.

Deploy the latest image	
```
$ oc deploy dc/bbtime --latest
Flag --latest has been deprecated, use 'oc rollout latest' instead
Started deployment #2
Use 'oc logs -f dc/bbtime' to track its progress.
```

Watch the output	
**Tip** Comeon!! I won't tell you how to do this
 


