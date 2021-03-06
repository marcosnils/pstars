# Securing Applications with Docker EE Advanced 

In this lab you will integrate Docker EE Advanced in to your development pipeline. You will build your application from a Dockerfile and push your image to the Docker Trusted Registry (DTR). DTR will scan your image for vulnerabilities so they can be fixed before your application is deployed. This helps you build more secure apps!


> **Difficulty**: Intermedio

> **Time**: Approximately 45-60 minutes


> **Tasks**:
>
> * [Prerequisites](#prerequisites)
> * [Task 1: Build a Docker Application](#task1)
>   * [Task 1.1: Inspect the Application Source](#task1.1)
>   * [Task 1.2: Build the Application Image](#task1.2)
>   * [Task 1.3: Deploy the Application Locally](#task1.3)
> * [Task 2: Pushing and Scanning Docker Images](#task2)
>   * [Task 2.1: Creating a Repo](#task2.1)
>   * [Task 2.2: Pushing to the Docker Trusted Registry](#task2.2)
> * [Task 3: Remediating Application Vulnerabilities](#task3)
>   * [Task 3.1: Rebuild the Image](#task3.1)
>   * [Task 3.2: Rescan the Remediated Application](#task3.2)
> * [Task 4: Enabling content trust to sign images](#task4)
>   * [Task 4.1: Set UCP only to deploy trusted images](#task4.1)
>   * [Task 4.2: Pushed singed images to DTR](#task4.2)
>   * [Task 4.3: Sign images that UCP can trust](#task4.3)
>   * [Task 4.4: Deploy a service in UCP with our signed image](#task4.4)


## Document conventions

When you encounter a phrase in between `<` and `>`  you are meant to substitute in a different value. 

Your nodes may be using private IP addresses along with mapped public IP addresses or the lab environment may be entirely private. We will be referring to these IPs as `node0-private-ip`, `node0-public-ip`, or `node0-public-dns`. If you are using only private IPs then you will replace the `public` IP/DNS with the private equivalent.


## <a name="prerequisites"></a>Prerequisites

This lab requires an instance of Docker Trusted Registry. This lab provides DTR for you. If you are creating this lab yourself then you can see how to install DTR [here.](https://docs.docker.com/datacenter/dtr/2.2/guides/admin/install/)

In addition to DTR, this lab requires a single node with Docker Enterprise Edition installed on it. Docker EE installation instructions can be found here. The nodes can be in a cloud provider environment or on locally hosted VMs. For the remainer of the lab we will refer to this node as `node0`


1. Log in to `node0` 

```
$ ssh ubuntu@node-smwqii1akqh.southcentralus.cloudapp.azure.com

The authenticity of host 'node-smwqii1akqh.southcentralus.cloudapp.azure.com (13.65.212.221)' can't be established.
ECDSA key fingerprint is SHA256:BKHHGwzrRx/zIuO7zwvyq5boa/5o2cZD9OTlOlOWJvY.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added 'node-smwqii1akqh.southcentralus.cloudapp.azure.com,13.65.212.221' (ECDSA) to the list of known hosts.
ubuntu@node-smwqii1akqh.southcentralus.cloudapp.azure.com's password:

Welcome to Ubuntu 16.04.2 LTS (GNU/Linux 4.4.0-72-generic x86_64)
```

2. Check to make sure you are running the correct Docker version. At a minimum you should be running `17.03 EE`

```
$ docker version
Client:
 Version:      17.03.0-ee-1
 API version:  1.26
 Go version:   go1.7.5
 Git commit:   9094a76
 Built:        Wed Mar  1 01:20:54 2017
 OS/Arch:      linux/amd64

Server:
 Version:      17.03.0-ee-1
 API version:  1.26 (minimum version 1.12)
 Go version:   go1.7.5
 Git commit:   9094a76
 Built:        Wed Mar  1 01:20:54 2017
 OS/Arch:      linux/amd64
 Experimental: false
 
```



## <a name="task1"></a>Task 1: Build and Running a Docker Application
The following task will guide you through how to build your application from a Dockerfile.

### <a name="task1.1"></a>Task 1.1: Inspect the App Source



1. Clone your application from the [GitHub repo](https://github.com/mark-church/docker-pets) with `git`. Go to the `/web` directory. This is the directory that holds the source for our application.

```
~$ git clone https://github.com/mark-church/docker-pets
~$ cd docker-pets/web
```

Inspect the directory.

```
~/docker-pets/web $ ls
admin.py  app.py  Dockerfile  static  templates
```

- `admin.py` & `app.py` are the source code files for our Python application.
- `/static` & `/templates` hold the static content, such as pictures and HTML, for our app.
- `Dockerfile` is the configuration file we will use to build our app.


2. Inspect contents of the `Dockerfile` for the web frontend image.

```
~/docker-pets/web $ cat Dockerfile
FROM alpine:3.3

RUN apk --no-cache add py-pip libpq python-dev curl

RUN pip install flask==0.10.1 python-consul

ADD / /app

WORKDIR /app

HEALTHCHECK CMD curl --fail http://localhost:5000/health || exit 1

CMD python app.py & python admin.py
```

Our Dockerfile includes a couple notable lines:

- `FROM alpine:3.3` indicates that our Application is based off of an Alpine OS base image.
- `RUN apk` & `RUN pip` lines install software packages on top of the base OS that our applications needs.
- `ADD / /app` adds the application code into the image.

### <a name="task1.2"></a>Task 1.2: Build the Application Image

1. Build the image from the Dockerfile. You are going to specify an image tag `docker-pets` that you will reference this image by later. The `.` in the command indicates that you are building from the current directory. Docker will automatically build off of any file in the directory named `Dockerfile`.

```
~/docker-pets/web $ docker build -t docker-pets .
Sending build context to Docker daemon 26.55 MB
Step 1/7 : FROM alpine:3.3
 ---> baa5d63471ea
Step 2/7 : RUN apk --no-cache add py-pip libpq python-dev curl
 ---> Running in 382419b97267
fetch http://dl-cdn.alpinelinux.org/alpine/v3.3/main/x86_64/APKINDEX.tar.gz
...
...
...
```

It should not take more than a minute to build the image.

2. You should now see that the image exists locally on your Docker engine by running `docker images`.

```
~/docker-pets/web $ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
docker-pets         latest              f0b92696c13f        3 minutes ago       94.9 MB
```

### <a name="task1.2"></a>Task 1.3: Deploy the App Locally

You will now deploy the image locally to ensure that it works. Before you do this you need to turn Swarm mode on in your engine so that we can take advantage of Docker Services.

1. Return to the `docker-pets` directory and establish a Swarm.

```
~/docker-pets/web $ cd ..
~/docker-pets $ docker swarm init
```

Confirm that you now have a Swarm cluster of a single node.

```
~/docker-pets $ docker node ls
ID                           HOSTNAME  STATUS  AVAILABILITY  MANAGER STATUS
fd3ovikiq7tzmdr70zukbsgbs *  moby      Ready   Active        Leader
```

2. Deploy your application from the compose file. In the `/docker-pets` directory there is a compose file that you will use to deploy this application. You will deploy your application stack as `pets`.

```
~/docker-pets $ docker stack deploy -c pets-container-compose.yml pets
Creating network pets_backend
Creating service pets_web
```

You have now deployed your application as a stack on this single node swarm cluster.

3. Verify that your application has been deployed.

```
~/docker-pets $ docker ps
CONTAINER ID        IMAGE                COMMAND                  CREATED             STATUS                    
a73f87ba147a        docker-pets:latest   "/bin/sh -c 'pytho..."   16 minutes ago      Up 16 minutes (healthy)
```

4. Go to your browser and in the address bar type in `<node0-public-DNS>`. (The app is serving on port 80 so you don't have to specify the port). This is the address where your local app is being served. If you see something similar to the following then it is working correctly. It may take up to a minute for the app to start up, so try to refresh until it works.

![](images/single-container-deploy.png) 

5. Now remove this service.

```
~/docker-pets $ docker stack rm pets
Removing service pets_web
Removing network pets_default
```

## <a name="task2"></a>Task 2: Pushing and Scanning Docker Images

### <a name="Task 2.1"></a>Task 2.1: Creating a Repo

Docker Trusted Registry (DTR) is the enterprise-grade image storage solution from Docker. You install it behind your firewall so that you can securely store and manage the Docker images you use in your applications.

In this lab, DTR has already been set up for you so you will log in to it and use it.

1. Log in to DTR. Before the lab you should have been assigned a cluster and a username. Use these when logging in.

Go to `https://<dtr-cluster>.dockerdemos.com` and login with your given username and password from your lab welcome email. You should see the DTR home screen.

![](images/dtr-home.png) 

2. Before you can push an image to DTR you must create a repository. Click on Repositories / New repository. Fill it with the following values:

- Account: select your username
- Repository Name: `docker-pets`
- Description: Write any description
- Visibility: Private
- Scan on Push: On

![](images/dtr-repo.png) 

3. Click Save. If you click on the `<account>/docker-pets` repo you'll see that it is empty as we have not pushed any images to it yet.


### <a name="Task 2.1"></a>Task 2.2: Pushing to the Docker Trusted Registry

Next we will push our local image to DTR. First we will need to authenticate with DTR so that our local engine trusts it. We will use a simple tool that pulls certificates from DTR to our development node.

1. Run the following command. Insert the name of your assigned cluster in the command.

```
~/docker-pets $ docker run -it --rm -v /etc/docker:/etc/docker \
  mbentley/trustdtr <dtr-cluster>.dockerdemos.com
  
  
Using the root CA certificate for trusting DTR
Adding certificate to '/etc/docker/certs.d/dtr.church.dckr.org/ca.crt'...done
Verifying format of certificate...done
```

For instance if `dtr3.dockerdemos.com` was your cluster then you would issue the command:

```
docker run -it --rm -v /etc/docker:/etc/docker mbentley/trustdtr dtr3.dockerdemos.com
```

2. Log in to DTR with your username and password.

```
~/docker-pets $ docker login <dtr-cluster>.dockerdemos.com
Username: <username>
Password: <password>
Login Succeeded
```

3. Tag the image with the URL of your DTR cluster and with the image tag `1.0` since this is the first version of your app.

```
~/docker-pets $ docker tag docker-pets <cluster-name>.dockerdemos.com/<username>/docker-pets:1.0
```

4. Now push your image to DTR.

```
~/docker-pets $ docker push <dtr-cluster>.docker-pets/<username>/docker-pets:1.0
The push refers to a repository [dtr2.dockerdemos.com/mark/docker-pets]
273eb8eab1c9: Pushed
7d68ed329d0d: Pushed
02c1439e0fdc: Pushed
9f8566ee5135: Pushed
1.0: digest: sha256:809c6f80331b9d03cb099b43364b7801826f78ab36b26f00ea83988fbefb6cac size: 1163
```

5. Go to the DTR GUI, click on your `docker-pets` repo, and click Images. The image vulnerability scan should have started already and DTR will display the status of the scan. Once the scan is complete, DTR will display the number of vulnerabilities found. For the `docker-pets` image a critical vulnerability was found.

![](images/scan-status.png) 

> You may need to refresh the page to show the status of a scan

6. Click on View details. The Layers tab shows the results of the scan. The scan is able to identify the libraries that are installed as a part of each layer.

![](images/layers.png) 

7. Click on Components. The Components tab lists out all of the libraries in the image, arranged from highest to least severity. The CVE is shown in addition to the usage license of the component. If you click on the CVE link it will take you to the official description of the vulnerability.

![](images/components.png) 

## <a name="task3"></a>Task 3: Remediating Application Vulnerabilities

We have built and pushed our application to DTR. DTR image scanning identified a vulnerability and now we are going to remediate the vulnerability and push the image again.

### <a name="Task 3.1"></a>Task 3.1: Rebuild the Image

We identified that our application has the known vulnerability `CVE-2016-8859`. We can see in the Layers tab that the affected package `musl 1.1.14-r14` is located in the top layer of our image. This is the base layer that was specified in the Dockerfile as `FROM alpine:3.3`. To remediate this we are going to use a more recent version of the `alpine` image, one that has this vulnerability fixed.

1. Go to the `~/docker-pets/web` directory and edit the Dockerfile.

```
~/docker-pets $ cd web
~/docker-pets/web $ vi Dockerfile
```

2. Change the top line `FROM alpine:3.3` to `FROM alpine:3.6`. `alpine:3.6` is newer version of the base OS that has this vulnerability fixed. Images change quite often, 
so it's recommended to check if there's a new alpine version as you might still get some scanning warnings.

3. Rebuild the image as version `2.0`

```
~/docker-pets/web $ docker build -t docker-pets .

```

4. Tag the image as the `2.0` and also with the DTR URL.

```
~/docker-pets/web $ docker tag docker-pets <dtr-cluster>.dockerdemos.com/<username>/docker-pets:2.0
```

5. Move back to the `docker-pets` directory and deploy the image locally again to ensure that the change did not break the app.

```
~/docker-pets/web $ cd ..

~/docker-pets $ docker stack deploy -c pets-container-compose.yml pets
Creating network pets_backend
Creating service pets_web
```

6. Go to your browser and in the address pane type in `<node0-public-ip>`. You should see that the app has succesfully deployed with the new change.

### <a name="Task 3.2"></a>Task 3.2: Rescan the Remediated Application

We have now remediated the fix and verified that the new version works when deployed locally. Next we will push the image to DTR again so that it can be scanned and pulled by other services for additional testing.

1. Push the new image to DTR.

```
~/docker-pets $ docker push <dtr-cluster>.dockerdemos.com/<username>/docker-pets:2.0
```

2. Go to the DTR UI and wait for the scan to complete. Once the scan has completed DTR will report that the critical vulnerabilities no longer exist, but we still have some major and minor things to check.

> You may need to refresh the page to show the status of a scan

![](images/partial_fix.png) 

3. Let's check our scanning details again to see what might be the problem this time. Go to the "view details" option and select the components or layers that are marked as conflictive.

![](images/partial_details.png) 

Seems like we still have some problems with some postgres and berkeleydb libraries that have some problems. Can you detect which might be the problem in our application Dockerfile?

4. Make the necessary changes to your `Dockerfile` to remediate this situation, build and push a new version of your Docker image. After pushing, check that all the vulnerabilities have dissapeared and that your 
image is ready to be deployed in production.

> You may need to refresh the page to show the status of a scan

![](images/clean.png) 


Congratulations! You just built an application, discovered two vulnerabilities, and patched it in just a few easy steps. Pat yourself on the back for helping create safer apps!!

### <a name="cleanup"></a>Lab Clean Up

If you plan to do other labs with these lab nodes then make sure to clean up after yourself! Run the following command and make sure that you remove any running containers.

```
~/docker-pets $ docker swarm leave --force
Node left the swarm.
```

## <a name="task4"></a>Task 4: Enabling contest trust in to sign images

By default, when you push an image to DTR, the Docker CLI client doesn’t sign the image.


![](images/sign-an-image-1.png) 

With Docker Universal Control Plane you can enforce applications to only use Docker images signed by UCP users you trust. When a user tries to deploy an application to the cluster, UCP checks if the application uses a Docker image that is not trusted, and won’t continue with the deployment if that’s the case.

![](images/run-only-the-images-you-trust-1.png) 

By signing and verifying the Docker images, you ensure that the images being used in your cluster are the ones you trust and haven’t been altered either in the image registry or on their way from the image registry to your UCP cluster.

Here’s an example of a typical workflow:

1. A developer makes changes to a service and pushes their changes to a version control system
2. A CI system creates a build, runs tests, and pushes an image to DTR with the new changes
3. The quality engineering team pulls the image and runs more tests. If everything looks good they sign and push the image
4. The IT operations team deploys a service. If the image used for the service was signed by the QA team, UCP deploys it. Otherwise UCP refuses to deploy.

### <a name="Task 4.1"></a>Task 4.1: Set UCP to only deploy trusted images

To configure UCP to only allow running services that use Docker images you trust, go to the UCP web UI, navigate to the `Settings` page, and click the `Content Trust` tab.

Select the `Run only signed images` option to only allow deploying applications if they use images you trust.

![](images/run-only-the-images-you-trust-2.png) 

With this setting, UCP allows deploying any image as long as the image has been signed. It doesn’t matter who signed the image. Additionally, you can enforce that the image needs to be signed by specific teams, include those teams in the `Require signature from` field. 

> If you specify multiple teams, the image needs to be signed by a member of each team, or someone that is a member of all those teams.

Click `Update` for UCP to start enforcing the policy.

Now we'll try to deploy a new service using a standard `nginx` image and if everything works as expected, you'll get an error indicating that
the image is not signed. Proceed to the `Resources` menu and validate this by creating a new service.

![](images/signer_required.png) 

Let's go ahead and configure our Docker CLI to sign our images in DTR.

### <a name="Task 4.2"></a>Task 4.2: Push signed images to DTR

You can configure the Docker CLI client to sign the images you push to DTR. This allows whoever pulls your image to validate if they are getting the image you created, or a forged one.

To sign an image you can run:

```
export DOCKER_CONTENT_TRUST=1
docker push <dtr-domain>/<repository>/<image>:<tag>
```

This pushes the image to DTR and creates trust metadata. It also creates public and private key pairs to sign the trust metadata, and push that metadata to the Notary Server internal to DTR.


![](images/sign-an-image-2.png) 

Let's try to tag an image locally and upload it to DTR using `content trust` enabled so we can check that's signed as expected.

```
export DOCKER_CONTENT_TRUST=1
docker pull nginx:latest
docker tag nginx:latest <dtr_repo>/<user>/nginx:latest
```

Wait, if we enabled content trust, how could we pull the nginx image from the Docker registry?. Because `nginx:latest` is an `official` Docker Store image, Docker takes care of signing them so you can use them right away securely. Let's try to pull an `unofficial` image using content trust enabled?

```
docker pull webdevops/nginx
Error: remote trust data does not exist for docker.io/webdevops/nginx: notary.docker.io does not have trust data for docker.io/webdevops/nginx
```

We have verified that we can't `pull` or `build` anything that hasn't been signed. Let's try to push our recently tagged image to DTR. If you get an `access denied` error, make sure that your Docker CLI is logged in to the registry and that the repository exists. If you get an `error contacting notary server: x509: certificate signed by unknown authority` run the following command to tell notary to trust the certificate

```
mkdir ~/.docker/tls
cp -r /etc/docker/certs.d/* ~/.docker/tls/
```

After running the previous steps, you should be able to push your signed image to DTR and Docker should ask for the `root` and `repository` key. Make sure to enter something easy to remember and after the image gets pushed, go to DTR and check the repository `images` tab. This pushes the image to DTR and creates trust metadata. It also creates public and private key pairs to sign the trust metadata, and push that metadata to the Notary Server internal to DTR.

![](images/signed_image.png) 

Bravo!, our image now is signed. Let's go ahead and deploy it in UCP as we did previously. Remember to use the full DTR image url to deploy our service in this case as we're not using the public registry. If everything goes "well", you'll see something like this:

![](images/signed_image_error.png) 

Hmm... what happened here?. The image appears as signed in DTR, shoudln't this work?. Well... there are some steps that we need to do first. 

### <a name="Task 4.3"></a>Task 4.3: Sign images that UCP can trust

With the steps above we were able to sign our DTR images, but UCP won’t trust them because it can’t tie the private key you’re using to sign the images to your UCP account.

To correctly sign images in a way that UCP trusts them you need to:

1. Configure your Notary client
2. Initialize trust metadata for the repository
3. Delegate signing to the keys in your UCP client bundle

First we'll configure our `notary` client to import our UCP private keys. Let's begin by downloading the `Notary` client for our platform from the [releases](https://github.com/docker/notary/releases) page or directly with the following command in a linux machine:

```
curl -L https://github.com/docker/notary/releases/download/v0.4.2/notary-Linux-amd64 -o /usr/bin/notary
chmod +x /usr/bin/notary
```

We need to configure the configure `notary` to speak with our DTR registry. For that purpose we'll create the following file in `~/.notary/config.json`

```
mkdir ~/.notary
```

With the following contents:

```
{
  "trust_dir" : "~/.docker/trust",
  "remote_server": {
    "url": "<dtr-url>",
    "root_ca": "<dtr-ca.pem>"
  }
}
```

The `ca.pem` should be located in the `/etc/docker/certs.d/<dtr_url>/ca.crt` folder. To verify that notary has been configured correctly, run the following command:

```
notary list <dtr_endpoint>/<user>/nginx
```
The command should print a list of digests for each signed image on the repository. Let's add our UCP key now; to accomplish that we need to download our UCP client bundle to get our private key and import it into notary. 
Head to the `Profile` screen in UCP and download the CLI bundle to your computer. Unzip it, copy the key.pem and cert.pem files where your configured `notary` client is and run the following command:

```
notary key import key.pem
```

The private key is copied to ~/.docker/trust, and you’ll be prompted for a password to encrypt it. You can validate what keys Notary knows about by running:

```
notary key list
```

The key you’ve imported should be listed with the role delegation.


The final step is to delegate trust to our UCP keys so that we can sign images with the private keys in our UCP client bundle. When delegating trust we associate a public key certificate with a role name. UCP requires that you delegate trust to two different roles:

- `targets/releases`
- `targets/<role>`, where `<role>` is the UCP team the user belongs to

In this example we’ll delegate trust to targets/releases and targets/admin as we're working with an admin user. Run the following commands to delete trust:

```
# Delegate trust, and add that public key with the role targets/releases
notary delegation add --publish \
  <dtr_endpoint>/<user>/nginx targets/releases \
    --all-paths cert.pem

# Delegate trust, and add that public key with the role targets/admin
notary delegation add --publish \
  <dtr_endpoint>/<user>/nginx targets/admin \
    --all-paths cert.pem
```

To push the new signing metadata to the Notary server, you’ll have to push the image again:

```
docker push <dtr_repo>/<user>/nginx:latest
```

### <a name="Task 4.4"></a>Task 4.4: Deploy a service in UCP with our signed image

If we did the previous steps correctly, we should be able to deploy our updated image using contest trust in UCP now. Let's go to the `Resources` menu and give one more try.


![](images/signed_ok.png) 

Fantastic!. We've configured notary in our machine and delegated proper trust to our UCP user so we can sign images and deploy them securely in our enviroment. We'll briefly see now how this workflow would work in an organization with different teams and responsibilities.


Instead of signing all the images yourself, you can delegate that task to other users.

A typical workflow looks like this:

1. A repository owner creates a repository in DTR, and initializes the trust metadata for that repository
2. Team members download a UCP client bundle and share their public key certificate with the repository owner
3. The repository owner delegates signing to the team members
4. Team members can sign images using the private keys in their UCP client bundles
5. In this example, the IT ops team creates and initializes trust for the dev/nginx. Then they allow users in the QA team to push and sign images in that repository.

![](images/delegate-image-signing-1.png) 
