# WebCenter Content on OpenShift

## Overview

This repository contains a [Dockerfile](wcc/Dockerfile) for building [Oracle WebCenter Content](https://www.oracle.com/technetwork/middleware/webcenter/content/overview/index.html) 12.2.1.4 on [Oracle WebLogic Server](https://www.oracle.com/middleware/technologies/weblogic.html) 12.2.1.4 using [Oracle Database](https://www.oracle.com/database/technologies/) Enterprise Edition 12.2.0.1.

## Requirements

### Accounts

- [Oracle account](https://profile.oracle.com/myprofile/account/create-account.jspx) (to use pre-built images with binaries)
- You will need [sudo / superuser rights](https://en.wikipedia.org/wiki/Sudo) on your machine in order to install Docker and Docker Compose

### Disk Space

- [WebCenter Content zip binary](#download-webCenter-content-zip-binary) = 1.6 GiB
- image `middleware/fmw-infrastructure:12.2.1.4-200316` = 2.3 GiB
- image `database/enterprise:12.2.0.1` = 3.4 GiB
- image `oracle/wccontent:12.2.1.4` = 5.6 GiB

### Software

#### The Main Server Where OpenShift will run

- OpenShift 3.11

#### Your Local Machine

- [Docker 18.09.0 or greater](https://docs.docker.com/install/#supported-platforms)
  - Version 18.09.0 or greater is required to use the `chown` [flag when copying files](https://github.com/moby/moby/pull/35521)
  - Docker is to build the WebCenter Content image since, unlike Oracle Database and Oracle FMW Infrastructure, Oracle does not provide a pre-built WebCenter Content image.
  - Ensure the Docker Daemon is running: `sudo service docker start`
 on
- [OpenShift CLI](https://docs.okd.io/3.11/cli_reference/get_started_cli.html)

### OpenShift Access

1. Validate you can login to the [OpenShift Web Console](https://docs.openshift.com/container-platform/3.11/getting_started/developers_console.html)

2. Validate you can login using the OpenShift CLI:
`oc login https://myserver:8443 --token=Wbx2yk9a16s6FizAh_XcZwfptgJUtIQX9VGTLZ-Xji1`.

    Your token can be found from the OpenShift Web Console by clicking on your profile name (in the top right), then "Copy Login Command":
    ![login Token secret](imgs/token.png)

3. Create a new OpenShift project:

`oc new-project oracle-apps`

    In this case my project is named `oracle-apps`. You can name your project differently but if you do, you will need to ensure any files or commands which reference `oracle-apps` are updated to use your project name.

## Build

### Login to the Oracle Container Registry

1. Navigate to the [Oracle Container Registry](https://container-registry.oracle.com)
2. Sign In using your Oracle account
3. Navigate to [*Middleware*](https://container-registry.oracle.com/pls/apex/f?p=113:1:13639930739021::NO:1:P1_BUSINESS_AREA:2) > *fmw-infrastructure*
4. Click Continue to agree and accept the Oracle terms and restrictions
5. Navigate to [*Database*](https://container-registry.oracle.com/pls/apex/f?p=113:1:12598430189688::NO:1:P1_BUSINESS_AREA:3) > *enterprise*
6. Click Continue to agree and accept the Oracle terms and restrictions
7. Run this command in a terminal and enter your your Oracle account credentials: `sudo docker login container-registry.oracle.com`

### Build Oracle Database Image

Oracle provides a pre-built image which can be imported directly into OpenShift from the Oracle Container Registry (container-registry.oracle.com/database/enterprise:12.2.0.1).

1. Create a new OpenShift secret which will log us into the Oracle Container Registry:
`oc create secret docker-registry oracle-container-registry-secret --docker-server=container-registry.oracle.com --docker-username=YourOracleAccountEmailGoesHere --docker-password=YourOracleAccountPasswordGoesHere --docker-email=YourOracleAccountEmailGoesHere

    In the above command, be sure to replace `YourOracleAccountEmailGoesHere` with your Oracle account email and `YourOracleAccountPasswordGoesHere` with your Oracle account password.

2. Import the Oracle Database Image
`oc import-image oracle12c:12.2.0.1 -confirm reference-policy='local' -from=container-registry.oracle.com/database/enterprise:12.2.0.1`

    If successful, you will see output similar to this:
    ????????????

3. Tag the Oracle Database Image
`oc tag oracle12c:12.2.0.1 oracle12c:latest`

    If successful, you will see output similar to this:

### Deploy Oracle Database

1. Run the following to deploy Oracle Database:
`oc process -f db/database-deployment.yaml | oc create -f -`

    If you changed your OpenShift project name from `oracle-apps`, then you will need to edit the `database-deployment.yaml` and alter `namespace` to be your project name.

2. After about 3-15 minutes, the Database service should be up and running.  You can track the progress of the deployment by doing the following:

    1. Get the pod name: `oc get pods`
        The pod name should be similar to `oracle12c-1-bltft`

    2. Check the logs either via the pod *Logs* tab in the Web Console or by running this: `oc logs oracle12c-1-bltft -c oracle12c`
        ![Pods Logs Screen ](imgs/dbdeploy4.png)

3. Validate that you can login to the Oracle Database view `sqlplus`. Go to the pod, then the Terminal tab and run the following 2 commands:
    `source /home/oracle/.bashrc`
    `sqlplus sys/Oradoc_db1@ORCLCDB as sysdba`

### Build the WebCenter Content Image

#### Download the Required Files

1. Navigate to a temporary directory (such as /tmp) where you have full permissions. Ensure you can access the Internet and have at least 10 GiB of free disk space.

2. Download [this zip file](https://github.com/praveenmogili/wcc-on-openshift/archive/master.zip)
`curl -o wcc-on-openshift.zip https://github.com/praveenmogili/wcc-on-openshift/archive/master.zip`

3. Extract the zip file
`unzip wcc-on-openshift.zip`

#### Download the WebCenter Content installation file

1. Download the WebCenter Content installation zip file from [here](https://www.oracle.com/middleware/technologies/webCenter-content-download.html).

    This zip file might be downloaded with one of these file names:
    - V983399-01.zip
    - fmw_12.2.1.4.0_wccontent_Disk1_1of1.zip

2. Place the downloaded zip file under the directory where you extracted wcc-on-openshift.zip (e.g. `/tmp/wcc-on-openshift/wcc`). The zip file needs to be named `fmw_12.2.1.4.0_wccontent_Disk1_1of1.zip` (rename the downloaded zip file if need be).

#### Build the image

1. Navigate to  `wcc/` folder in your local cloned repository directory
2. Run this command in a terminal: `docker build .`
3. You should eventually(~ 15-20 minutes later) see a message indicating success:

#### Push the image to OpenShift

1. Run the following command to push the image we just built:

`oc import-image --insecure <image_stream_name>[:<tag>] --from=<docker_image_repo> --confirm`

Once this is done successfully, you should be able to see the ImageStream by running this command:
`oc get is`

```
% oc get is  
NAME                 DOCKER REPO                               TAGS                                                   UPDATED
docker               172.30.1.1:5000/oracle-apps/docker               latest                                                 3 days ago
enterprise           172.30.1.1:5000/oracle-apps/enterprise           12.2.0.1                                               9 days ago
fmw-infrastructure   172.30.1.1:5000/oracle-apps/fmw-infrastructure   12.2.1.3-200316,12.2.1.4-200316,12.2.1.3 + 1 more...   3 days ago
instantclient        172.30.1.1:5000/oracle-apps/instantclient        latest                                                 9 days ago
oraclelinux          172.30.1.1:5000/oracle-apps/oraclelinux          7                                                      9 days ago
oradb                172.30.1.1:5000/oracle-apps/oradb                12.2.0.1-ee                                            
wccontent            172.30.1.1:5000/oracle-apps/wccontent            12.2.1.3                                               24 hours ago
```

### Deploy Oracle WebCenter Content

### Create the Deployment in the OpenShift Web Console

Now, it's time to deploy WebCenter Content. Click Deploy Image as shown below and select the wccontent image stream uploaded above.

![WCC Deploy 1 ](imgs/wcc-deploy1.png)

### Create the Persistent Storage and Attach it to the deployment
Add persistent storage for domain configuration data and log files.

![WCC Storage ](imgs/wcc-storage1.png)
Then attach it in the deployment configuration as shown below:

![WCC Storage ](imgs/wcc-storage1.png)

### Create the Environment Variable

Based on the *wcc.env* file create the environment for WCC app

![WCC Environment](imgs/wcc-environ.png)

### Deploy the WCC Pod

1. Deploy the application and watch the logs for the pods to finish (takes anywhere from 15-20 minutes).
    ![WCC Pod Logs ](imgs/wcc-podlogs.png)

2. Verify the Pod is running:
    ![WCC Pod Run ](imgs/wcc-podrun.png)

### Create the Service

1. Run the following:
`oc apply -f wcc/wcc-service.yaml`

### Create Routes

Create two routes:

1. Weblogic Server Admin Console (port 7001):
    ![WCC Route1 ](imgs/wcc-admin-route.png)

2. WebCenter Content application (port 16200):
    ![WCC Route2 ](imgs/wcc-app-route.png)

## Accessing the Application

### URLs

- WebLogic Server Administration Console: <http://server:7001/console/>
- WebCenter Content: <http://server:16200/cs/>

Credentials can be found in the [wcc.env file](wcc/wcc.env).

### First Login

1. The first time you access the WebLogic Server Administration Console (after a WebLogic Server restart), it will deploy the Console and redirect you to the login page.
    ![WCC App1 ](imgs/app1.png)
    ![WCC App2 ](imgs/app2.png)
    ![WCC App3 ](imgs/app3.png)

2. The very first time you access WebCenter Content, it will ask you to select configuration options and restart the node.
    ![WCC App41 ](imgs/app41.png)

## WebCenter Content Restart

To restart WebCenter Content, you can remove the pod and it will restart as the auto start/rollout is enablled in the deployment.

Here is an example screenshot of removing a pod:

![WCC Pod Delete ](imgs/pod-delete.png)

`oc` CLI can also be used for deleteing pods:
first get the name of the pod using

`oc get pods`

```
NAME                         READY     STATUS             RESTARTS   AGE
fmw-infrastructure-9-6gx9n   1/1       Running            0          24m
instantclient-1-pvvg7        0/1       CrashLoopBackOff   9          24m
oradb1-9-xcdnl               1/1       Running            0          35m
wccontent-13-6wtp6           1/1       Running            1          26m
```

and then using `oc delete pod wccontent-13-6wtp6`

Once the pod has started again, you will see the login screen as shown below:

![WCC App4 ](imgs/app4.png)
![WCC App5 ](imgs/app5.png)