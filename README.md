
# Let's have an Elastic Beanstalk walkthrough :seedling:

## Prologue
This repo houses a simple Node web app using [Mithril JS](expressjs.com/en/starter/hello-world.html) and [Express JS](https://expressjs.com/en/starter/hello-world.html) for the frontend and backend pieces. You should be able to read along the code and easily pick-up what it's doing. We'll use this app to play around Beanstalk.

## Setting Up
  1. Configure your AWS credentials locally including the necessary IAM policies for `Elastic Beanstalk`.
  1. Install EB CLI by following [the official guide](https://github.com/aws/aws-elastic-beanstalk-cli-setup).
  1. Install `Node` (I'm using `v14.15.0`) on your machine or on your Docker container, whichever is fine.
  1. Install `Docker` (I'm using `v19.03.12`) into your Host OS.
  1. Clone [this repo](https://github.com/maronavenue/elastic-beanstalk-sample-app) and get into the root directory. Make sure you're in the default branch (`main`).

## Let's get started
Time to get our hands dirty! :clap:

### 0. Test sample app locally
Just to make sure things are going smoothly, start the app by running `npm start`. Be sure to export the `PORT` environment variable. Check on your browser: `http://localhost:8080`.

### 1. Initialize your app from the root work directory
Initialize the EB app using this command. Select the appropriate region and platform (`Node`). Skip `CodeCommit` and `SSH settings`. We don't need them in this hands-on.
```bash
$ eb init

Select a default region
Enter Application Name (default is "elastic-beanstalk-sample-app"):
It appears you are using Node.js. Is this correct? (Y/n):
Select a platform branch.
Do you wish to continue with CodeCommit? (Y/n):
Do you want to set up SSH for your instances? (Y/n):
```
You will notice right away that it just generated a hidden config file at `/.elasticbeanstalk/config.yml`. The EB service will use this configuration throughout your dev workflow cycle. It does not do anything with your Elastic Beanstalk account.

### 2. Create your first environment
Now that you have your app, it's time to create your first environment. By default, `eb create` will prompt to build a standard `Load Balancer` + `Auto-scaling Group` with `1:1:4 desired:min:max` capacity settings to guarantee that your app will always be running — **_we don't need that_**. Let's just spin-up a single EC2 to minimize costs using `--single`. You can also pass `t2.micro` using the `--instance-types` arg if you're using `Free Tier` eligible and it's not in your default settings.
```bash
$ eb create --single

Enter Environment Name
(default is elastic-beanstalk-sample-app-env):
Enter DNS CNAME prefix
(default is elastic-beanstalk-sample-app-env):
Would you like to enable Spot Fleet requests for this environment? (y/N): N
Creating application version archive "app-e724-201127_231608".
Uploading elastic-beanstalk-sample-app/app-e724-201127_231608.zip to S3. This may take a while.
Upload Complete.
Environment details for: elastic-beanstalk-sample-app-env
  Application name: elastic-beanstalk-sample-app
  Region:
  Environment ID:
  Platform:
  Tier:
  CNAME:
  Updated:
Printing Status:
...
2020-11-27 15:18:14    INFO    Instance deployment: You didn't specify a Node.js version in the 'package.json' file in your source bundle. The deployment didn't install a specific Node.js version.
2020-11-27 15:18:20    INFO    Instance deployment completed successfully.
```
**Warning :warning::** *This will **incur** small charges to your account if you're not `Free Tier` eligible.*

## 3. Verify your environment.
There are tons of ways to check your EB environment. You can check it directly on the `AWS Console` or via `EB CLI` under `events` and `logs`. Check the actual deployment using `eb status`. Pay attention to `Status` and `Health` as it should dictate if the deployment has completed successfully.
```bash
$ eb status

Environment details for: elastic-beanstalk-sample-app-env
  Application name: elastic-beanstalk-sample-app
  Region:
  Deployed Version:
  Environment ID:
  Platform:
  Tier:
  Updated:
  Status: Launching
  Health: Grey

$ eb events

20-11-27 15:16:11      INFO    createEnvironment is starting.
...
2020-11-27 15:17:54    INFO    Waiting for EC2 instances to launch. This may take a few minutes.
2020-11-27 15:18:14    INFO    Instance deployment: You didn't specify a Node.js version in the 'package.json' file in your source bundle. The deployment didn't install a specific Node.js version.
2020-11-27 15:18:20    INFO    Instance deployment completed successfully.

$ eb logs  

Retrieving logs...
============= i-074ab76ddf4729ff4 ==============
...

$ eb open
```
Note that `eb open` simply opens the web app using your browser. Aside from navigating through the EB Console and CLI, I recommend checking out the underlying resources that EB created to truly appreciate the power of EB. Here are some:
&nbsp;&nbsp;:white_check_mark: `EC2 Instances` including Security Groups, Elastic IPs, Load Balancers, Target Groups and Auto-scaling groups
&nbsp;&nbsp;:white_check_mark: `S3 Bucket` to internally store the application and logs.
&nbsp;&nbsp;:white_check_mark: `CloudFormation Stack` for the overall infrastructure management.

## 4. Deploy new versions using common policies
EB uses `All-At-Once` as the default deployment policy which directly deploys the changes into the instance/s, hence the app is expected to go `out-of-service` until the deployment has completed. As such, we define this as `mutable`. There are other mutable deployments such as `Rolling`, but we'll take a look at `Immutable` in this example. Deployment policy can be configured by creating `JSON` or `YAML` files with `*.config` extension under a special folder you'll create as well: `/.ebextensions/`.

### 4.a. Immutable
You will notice that there's already an `/.ebextensions/deployment_policy.config`. Yes, you don't have to do anything since the policy was already defined as `Immutable`. Modify `/public/app.js` to simulate a change in the frontend. Be sure to commit and push your changes prior to actual deployment.
```bash
$ eb deploy

Creating application version archive "app-9314-201127_235718".
Uploading elastic-beanstalk-sample-app/app-9314-201127_235718.zip to S3. This may take a while.
Upload Complete.
2020-11-27 15:57:20    INFO    Environment update is starting.
2020-11-27 15:57:28    INFO    Immutable deployment policy enabled. Launching one instance with the new settings to verify health.
2020-11-27 15:57:59    INFO    Created temporary auto scaling group awseb-e-ifjhvssvsd-immutable-stack-AWSEBAutoScalingGroup-QF84XXNS8XLV.
2020-11-27 15:59:03    INFO    Instance deployment: You didn't specify a Node.js version in the 'package.json' file in your source bundle. The deployment didn't install a specific Node.js version.
2020-11-27 15:59:10    INFO    Instance deployment completed successfully.
```
**_...Takes a while right?_** Go get some `coffee :coffee:` while waiting. Cheers. Anyway, here's where `Immutable` shines: It does not touch your running instance/s within the existing `Auto-scaling group`. Instead, it creates an entirely new `Auto-scaling group` and replaces the old group once its health checks are passing. `This is why it's noticably longer (and will cost more)`, but you can imagine how easy it is to rollback automatically when things go wrong unlike with `Mutable`. There is no `downtime` as well. Personally, I like this deployment policy for my `production-grade` applications. :+1:

### 4.b. All-At-Once
I know we glossed over this guy, but just `git rm /.ebextensions/deployment_policy.config` then modify `/public/app.js` again for yet another change. Please keep the "Blue version" as is for now, okay? We'll use that to demonstrate the next section. Again, don't forget to commit and push your changes so EB will recognize them then hit `eb deploy`. You will eventually notice that it's way faster, but your app goes down for a bit of time. This deployment policy is okay if you or your users can tolerate downtime. This is the most cost-effective deployment too. I like to standby on the `EC2 Console` just to compare how both deployment policies take down the running instances. Try it too.

## 5. Blue-Green Deployment
Now that we're at the topic of `Immutable` deployments, we can see and understand that it operates **within the context of running instances**. Now, let's look at the bigger picture: how about we look at our project **within the context of environments**? It suddenly changes things. We can now logically say that it is no longer `Immutable` because it makes changes into the `same environment`, albeit it creates new Auto-scaling groups inside. **Blue-Green Deployment** is a general paradigm that is outside of `Elastic Beanstalk`'s features and introduces a new, second environment — called `"Green"`. We'll call the first environment `"Blue"`. We'll deploy everything into our `Green` environment as if it's a completely separate app and then swap its `CNAME` with the `Blue` environment once we're happy with its behavior. We can take advantage of our full control over the environment here: We have the option to observe it for several days. We could also synergize with `Route53` in order to gradually distribute traffic using `weighted routing policy`, e.g. 90%-10%. We can manage user impact gracefully. It's up to us how long we should keep the other environment up and running. Heck, we can even terminate it immediately after the swap or the weighted routing goes to 100%. Point is, rollback is within our hands.
```bash
# Link your first environment (Blue) to the current branch
# This just tells EB to deploy to this environment when making changes into this branch
(main)  $ eb use elastic-beanstalk-sample-app-env

# Create and link your second environment (Green) using 'green' branch for the work directory
# Linking is just simply being tracked in: /.elasticbeanstalk/config.yml
(main)  $ git checkout green
(green) $ eb init
(green) $ eb create --single
(green) $ eb use elastic-beanstalk-sample-app-green
(green) $ eb deploy

# Green version should should up in the title
(green) $ eb open

# Perform an environment swap
(green) $ eb swap elastic-beanstalk-sample-app-green --destination_name elastic-beanstalk-sample-app-env

# Wait a couple of minutes then verify if "Blue version" is now showing "Green version"
(green) $ git checkout main
(main)  $ eb open
```
**Warning :warning::** As you know, terminating the entire EB environment also deletes all underlying resources that were created... including the database. As best practice, create your database outside of the EB environment and just source it as you would normally do in your code. And oh, since `Blue-Green` deployment utilizes a `CNAME` swap it's just natural to expect some respectable delays since it will have to propagate over the Public DNS. Okay, I think you should now be able to start picturing its advantages and disadvantages, including the right use cases to apply it. :+1:


## 6. Cleaning Up
As a `PaaS` itself, `Elastic Beanstalk` is capable of handling deletion quite well especially that it leverages and utilizes `CloudFormation` under the hood. You shouldn't directly delete any resources created by EB if you don't want to encounter `configuration drifts`. Let the platform do its job. :slightly_smiling_face:
```bash
# Option 1: Terminate specific environment
$ eb terminate
The environment "elastic-beanstalk-sample-app-env" and all associated instances will be terminated.
To confirm, type the environment name:

# Option 2: Terminate app and all its environments
$ eb terminate --all
The application "elastic-beanstalk-sample-app" and all its resources will be deleted.
This application currently has the following:
Running environments: 2
Configuration templates: 0
Application versions: 10

To confirm, type the application name: elastic-beanstalk-sample-app
Removing application versions from s3.
2020-11-27 17:21:10    INFO    deleteApplication is starting.
2020-11-27 17:21:11    INFO    Invoking Environment Termination workflows.
2020-11-27 17:23:43    INFO    The environment termination step is done.
2020-11-27 17:23:44    INFO    The application has been deleted successfully.
```

## Single-container configurations using Docker
We previously learned how to deploy simple web apps using `Node.js` as the platform. It's okay and all, but what if we can ship our entire `config` together with our `code`? Enter `Elastic Beanstalk :seedling:` and `Docker :whale2:`!

Time to containerize our app! :package:
### 0. Test sample app locally with Docker
Checkout `dockerized` branch then run the ff. commands. Here's a brief `Docker` crash course:
```bash
# To check for running containers
(dockerized) $ docker ps

# To list existing images
(dockerized) $ docker images

# To build an image based on the Dockerfile on current working directory
(dockerized) $ docker build --tag beanstalk-sample-app:1.0 .

# To run the container from the image
# Note: It does a few extra things since we're dealing with a containerized web application such as forwarding the Host OS's port to map inside the container's port.
(dockerized) $ docker run --env PORT=8080 --publish 8080:8080 beanstalk-sample-app:1.0

# Open your browser
http://localhost:8080/
```
### 1. Initialize your app from the root work directory
Make sure to checkout `dockerized` branch then initialize the EB app using the same commands. Nothing fancy here. If you terminated your app from the previous section, you should be able to notice that `eb init` regenerates the `/.elasticbeanstalk/config.yml` file. I forgot to mention that it also updates your `.gitignore` to skip some of these files (Beanstalk is that thoughtful). Anyway, EB CLI will notice the `Dockerfile` in root directory then prompt for confirmation.

```bash
(green)      $ git checkout dockerized
(dockerized) $ eb init 

Select a default region (default is 3):

Enter Application Name (default is "elastic-beanstalk-sample-app"):
Application elastic-beanstalk-sample-app has been created.

It appears you are using Docker. Is this correct? (Y/n): Y
Select a platform branch.
1) Docker running on 64bit Amazon Linux 2
2) Multi-container Docker running on 64bit Amazon Linux
3) Docker running on 64bit Amazon Linux
(default is 1):

Do you wish to continue with CodeCommit? (Y/n): n
Do you want to set up SSH for your instances?
(Y/n): n
```

### 2. Create your environment using local Dockerfile
Run `eb create --single` again then EB should build the image from the `Dockerfile` which also defines all the config including commands to start the app and port to expose. You noticed that I've changed `/.ebextensions/env_vars.config` to match the `PORT` environment to `8080` in the `Dockerfile`. Not that it matters to be honest, because again with `Docker` we've defined the port as part of its config. `env_vars.config` is just here to demonstrate EB-specific config we can do, but it's not essential for this section.

Feel free to `eb terminate --all` after playing for awhile.

### 3. Create your environment using remote Docker image (Docker Hub)
With the previous step, we tasked EB to build the image inside the EC2 instance using the `Dockerfile` as part of the initialization/deployment. You will notice that we still have the entire codebase that is necessary to build the `Docker image`. We can go one step further to leverage `Docker` by deploying a hosted image in some registry such as [Docker Hub](https://hub.docker.com/) which **containerizes** our, well, `config` and `code`. Checkout `dockerized-remote` branch then you'll see that there aren't pretty much files left other than this special file: `Dockerrun.aws.json`. This must sit at the root project directory since EB will use this to be able to pull any images including additional configurations. I have containerized the entire app and pushed the image into [maronavenue/beanstalk-sample-app:latest](https://hub.docker.com/r/maronavenue/beanstalk-sample-app). It's a public image on `Docker Hub` that can readily be used via `docker pull maronavenue/beanstalk-sample-app`. Alright sir, standard operating procedures!
```bash
(dockerized-remote) $ eb create --single
Enter Environment Name
(default is elastic-beanstalk-sample-app-dev): elastic-beanstalk-sample-app-dockerized
Enter DNS CNAME prefix
(default is elastic-beanstalk-sample-app-dockerized):

Would you like to enable Spot Fleet requests for this environment? (y/N): N
Creating application version archive "app-812e-201128_145846".
Uploading elastic-beanstalk-sample-app/app-812e-201128_145846.zip to S3. This may take a while.
Upload Complete.
Environment details for: elastic-beanstalk-sample-app-dockerized
  Region:
  Deployed Version:
  Environment ID:
  Platform:
  Tier:
  CNAME:
  Updated:
Printing Status:
2020-11-28 06:58:49    INFO    createEnvironment is starting.
...
2020-11-28 07:01:24    INFO    Instance deployment completed successfully.

(dockerized-remote) $ eb status

(dockerized-remote) $ eb open
```