## Change Application and Test Locally

In this chapter, we will learn to do local application development on an OpenShift cluster on your machine. 

**Use Case:** As a developer, I want to repeatedly make changes to my code and test locally on OpenShift. Only when I am happy with my code changes I want to push my code to the source control repository. 

**Prerequisites:**

* You have a prior understanding of OpenShift Concepts i.e, pods, services, routes, build configuration, deployment configuration, imagestream etc.
* `Git` client and `oc` client are installed on your workstation
* OpenShift Local cluster is up and running

We will begin with simple java code that runs as SpringBoot fat jar in a container. It requires a plain java builder image that is built and available on dockerhub.

We will begin with source code from a git repository at [https://github.com/RedHatWorkshops/spring-sample-app](https://github.com/RedHatWorkshops/spring-sample-app), deploy the application from this repo, make changes locally and test. 

**Step 1: Import SpringBoot/Java Container image from DockerHub** 

If you logged into your local cluster as a user `developer`, you should be in the project named `myproject`. If not switch to the project.

```
$ oc project myproject
```

If you deployed other things in this project and you want to start with a clean slate, you may want to clean it up

```
$ oc delete all --all

```

A plain java container image is available on dockerhub in my docker repository `docker.io/veermuchandi/spring-mvn-base`. This is an S2I builder image i.e, it can take your source code, run a maven build and turn it into a docker image. 
We will first import this S2I builder into our project by running `oc import-image`. We are naming the imagestream as `springboot`.

```
$ oc import-image --from=veermuchandi/spring-mvn-base:latest springboot --confirm
The import completed successfully.

Name:			springboot
Namespace:		myproject
Created:		Less than a second ago
Labels:			<none>
Annotations:		openshift.io/image.dockerRepositoryCheck=2017-02-10T04:33:02Z
Docker Pull Spec:	172.30.42.182:5000/myproject/springboot
Unique Images:		1
Tags:			1

latest
  tagged from veermuchandi/spring-mvn-base:latest

  * veermuchandi/spring-mvn-base@sha256:6fd145a92bfb569cbf3f86c36996ab953e41ab90fd4a5876bc71ba45390b7b95
      Less than a second ago
```

Now if you search for the imagestream, you should find it. 

```
$ oc new-app --search springboot
Image streams (oc new-app --image-stream=<image-stream> [--code=<source>])
-----
springboot
  Project: myproject
  Tags:    latest
```

**Step 2: Deploy the application ** 

Let us now deploy the application from github using the springboot imagestream we created in the last step. We are naming this application `bootapp`.

```
$ oc new-app springboot~https://github.com/RedHatWorkshops/spring-sample-app.git --name=bootapp
--> Found image c3ddd9e (3 months old) in image stream "myproject/springboot" under tag "latest" for "springboot"

    Spring Boot Maven 3 
    ------------------- 
    Platform for building and running Spring Boot applications

    Tags: builder, java, java8, maven, maven3, springboot

    * A source build using source code from https://github.com/RedHatWorkshops/spring-sample-app.git will be created
      * The resulting image will be pushed to image stream "bootapp:latest"
      * Use 'start-build' to trigger a new build
    * This image will be deployed in deployment config "bootapp"
    * Port 8080/tcp will be load balanced by service "bootapp"
      * Other containers can access this service through the hostname "bootapp"

--> Creating resources ...
    imagestream "bootapp" created
    buildconfig "bootapp" created
    deploymentconfig "bootapp" created
    service "bootapp" created
--> Success
    Build scheduled, use 'oc logs -f bc/bootapp' to track its progress.
    Run 'oc status' to view your app.
```

Now let's create a `route` by exposing the `service`

```
$ oc expose svc bootapp
route "bootapp" exposed

$ oc get route
NAME      HOST/PORT                                PATH      SERVICES   PORT       TERMINATION
bootapp   bootapp-myproject.192.168.0.102.xip.io             bootapp    8080-tcp 
```

In the meanwhile, since we started with source code, our SpringBoot S2I builder would have started building the source code to create a Container Image for your application. Let's check the build logs.

```
$ oc logs -f bootapp-1-build
Cloning "https://github.com/RedHatWorkshops/spring-sample-app.git" ...
	Commit:	b11dc6d4e0bee0ee29d9a63a5dcc125e1c292633 (added lastnames matching databases for ease)
	Author:	VeerMuchandi <veer.muchandi@gmail.com>
	Date:	Thu Nov 17 17:13:15 2016 -0500


.....
.....
.....


[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 52.730 s
[INFO] Finished at: 2017-02-10T04:58:06+00:00
[INFO] Final Memory: 32M/233M
[INFO] ------------------------------------------------------------------------


Pushing image 172.30.42.182:5000/myproject/bootapp:latest ...
Pushed 0/8 layers, 0% complete
Pushed 1/8 layers, 14% complete
Pushed 2/8 layers, 26% complete
Pushed 3/8 layers, 40% complete
Pushed 4/8 layers, 52% complete
Pushed 5/8 layers, 72% complete
Pushed 6/8 layers, 97% complete
Pushed 7/8 layers, 99% complete
Pushed 8/8 layers, 100% complete
Push successful

```

The above is a partial paste of the logs. **Specifically note**, that the first step was to clone the source code from the git repository we provided. Once the build completes, it will push the container image into the internal registry. After that the application will be deployed as a pod. Well, you know all this!!!

Wait until the application pod is running. In my case the object `bootapp-1-gnkoz` is my application running as a pod on my local OpenShift cluster.

```
$ oc get pods
NAME              READY     STATUS      RESTARTS   AGE
bootapp-1-build   0/1       Completed   0          14m
bootapp-1-gnkoz   1/1       Running     0          12m
```

Also check the pod logs to make sure your app is ready. This may take a minute or two as the container would first bring up SpringBoot and run your app. Partial output is shown below.

```
$ oc logs bootapp-1-gnkoz

...
..
...
..
...

2017-02-10 04:59:01.114  INFO 9 --- [           main] com.example.SpringSampleAppApplication   : Started SpringSampleAppApplication in 7.505 seconds (JVM running for 8.194)
...
...

```

Now, let's test our application

```
$ curl http://bootapp-myproject.192.168.0.102.xip.io/
<h1>Hello from bootapp-1-gnkoz</h1>
```
Or you can also run it from the browser. Honestly, I am lazy to paste the screen image into docs and hence used curl ;) 
While it is simple and silly code, it looks better than this in a browser, though!!


**Step 3: Making changes locally**

In order to make changes locally, we need code on our box. So let's `git clone` it. It creates a directory named `spring-sample-app`. Staying in that directory is **Important**. 

```
$ git clone https://github.com/RedHatWorkshops/spring-sample-app.git
Cloning into 'spring-sample-app'...
remote: Counting objects: 546, done.
remote: Total 546 (delta 0), reused 0 (delta 0), pack-reused 546
Receiving objects: 100% (546/546), 47.59 KiB | 0 bytes/s, done.
Resolving deltas: 100% (146/146), done.


$ cd ./spring-sample-app/

```



Let's change the code now.

```
$ vi ./src/main/java/com/example/SpringSampleAppApplication.java
```
Locate this code snippet
```
class HomeRestController {

        boolean healthy=true;
    String hostname="";
        public  HomeRestController(){
                try {
                        hostname= "Hello from " + InetAddress.getLocalHost().getHostName().toString();
                }
                catch (UnknownHostException ex){
                        hostname= "error";
                }
        }

```
Change the line that assigns HostName to the variable String variable hostname as shown below. While I am using `vi` you can use your favorite editor.

```
hostname= "This Pod's Host Name is " + InetAddress.getLocalHost().getHostName().toString();
```
Save the file. No need to commit.

**Absolutely Important** Make sure you are still in `spring-sample-app` folder. If not return there.

```
$ pwd
<<MYWORKDIR>>/spring-sample-app
```
To recap, we made a code change. We did not commit to the git repo. We just want to test the change first. Let's do it.

Find the build configuration. **Note** that the Type is `Source` and From says `Git`.

```
$ oc get bc
NAME      TYPE      FROM      LATEST
bootapp   Source    Git       1
```

If you see the output as json, you will find the following snippet which is pointing to the git repository we used while creating this app.

```
$ oc get bc/bootapp -o json

...
...
...

        "source": {
            "type": "Git",
            "git": {
                "uri": "https://github.com/RedHatWorkshops/spring-sample-app.git"
            }
        },
        "strategy": {
            "type": "Source",
            "sourceStrategy": {
                "from": {
                    "kind": "ImageStreamTag",
                    "namespace": "myproject",
                    "name": "springboot:latest"
                }
            }
            
...
...
...

```

Let's start a new build by pointing to our current directory (i.e, the code is all in spring-sample-app directory.. do you understand why I was pressing so much on your current location?).


```
$ oc start-build bc/bootapp --from-dir=.
Uploading directory "." as binary input for the build ...
build "bootapp-2" started

```
This initiates what we call a `binary` build. This will override the `source` section in build configuration while running the build as

```
        "source" : {
            "type" : "Binary",
            "binary" : {}
        },
```
You won't see it, but that's what happens behind the scene. 

**Note** For some strange reason, if the build fails and it complains, you can manually edit the build configuration and change the `source` section as above. How to edit? `oc edit bc/bootapp -o json`


Now check that be build is running.

```
$ oc get pods
NAME              READY     STATUS      RESTARTS   AGE
bootapp-1-build   0/1       Completed   0          20m
bootapp-1-gnkoz   1/1       Running     0          17m
bootapp-2-build   1/1       Running     0          29s
```

Look at the build logs. You will notice that this time the source is not cloned from the git repository, but instead received from your local machine. Yay!! this is exactly what we need to test the changes locally.

```
$ oc logs -f bootapp-2-build

Receiving source from STDIN as archive ...

---> Installing application source

...
..
...
..
...
...



[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 48.858 s
[INFO] Finished at: 2017-02-10T05:16:55+00:00
[INFO] Final Memory: 31M/218M
[INFO] ------------------------------------------------------------------------


Pushing image 172.30.42.182:5000/myproject/bootapp:latest ...
Pushed 7/8 layers, 88% complete
Pushed 8/8 layers, 100% complete
Push successful
```

Once build is complete, the application pods comes up, make sure you give enough time for it to be up and ready.

```
$ oc get pods
NAME              READY     STATUS      RESTARTS   AGE
bootapp-1-build   0/1       Completed   0          21m
bootapp-2-build   0/1       Completed   0          2m
bootapp-2-ojg4z   1/1       Running     0          1m
```

And test it with browser or curl and note the changes you made are here.

```
$ curl http://bootapp-myproject.192.168.0.102.xip.io/
<h1>This Pod's Host Name is bootapp-2-ojg4z</h1>
```

**Conclusion:** In this chapter, we learnt do make changes to code and test locally.






