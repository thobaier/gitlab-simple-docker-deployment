# Gitlab Docker deployment example
> A simple example for a docker deployment with Gitlab CE and the built-in docker registry. Just a starting point for dev-deployments with Gitlab tools. No need for extra CI Tools.  

We will use:

* Gitlab as VCS
* Gitlab as Docker registry
* Gitlab as CI / CD deployment tool

## Basics
We will run all of the dependencies in docker containers:

* Gitlab
* Gitlab runner
* Docker dind

## Host
We will use [https://gitlab/](https://gitlab/) as host so make sure you have added it in your `/etc/hosts` file.

### Bring up the project
For the example docker configuration check the `docker-compose.yml` provided by this project.  
Now let's bring up the containers we need with:
```bash
docker-compose up
```

After a short startup (and initialisation of each container) the project should be running.  
(For the first start it will take some minutes. We are done when you reach [https://gitlab/](https://gitlab/) via your browser) 

You can verify that all containers where bring up correctly by
```console
$ docker ps
CONTAINER ID  IMAGE                        COMMAND                 CREATED            STATUS         NAMES
c05e61d16a34  gitlab/gitlab-ce:latest      "/assets/wrapper"       12 seconds ago     Up 11 seconds  gitlab
dde31f19a26c  gitlab/gitlab-runner:latest  "/usr/bin/dumb-init …"  12 seconds ago     Up 11 seconds  runner                                                                                                           runner
23eed28af396  docker:18.05.0-ce-dind       "dockerd-entrypoint.…"  About an hour ago  Up 11 seconds  registry
```

Fine, everything is running!

***Do not forget to add your SSH key in Gitlab backend for communicate via SSH with Gitlab.***

## Get the IP from host
We need the IP of the host (wich is running the compose file). In my case its my local maschine, so to grab your ip you could use something like: 


```bash
ip route get 1 | awk '{print $NF;exit}'
# or
ip route get 8.8.8.8 | awk '{print $NF; exit}'
# or
hostname -I | cut -d' ' -f1
...
```
and copy the IP for the next step:

## Register the runner in Gitlab
Now it's time to setup and register our runner for Gitlab. We need a `registration token` from Gitlab to register our runner.  

Go to [https://gitlab/admin/runners](https://gitlab/admin/runners) and copy your token under the "Setup a shared Runner manually" section.

Now we will login to the `runner` container
```bash
docker exec -it runner bash
```
and execute the register runner script (change the token and IP you copied before):

```bash
gitlab-runner register -n \
  --url https://gitlab/ \
  --registration-token TOKEN_FROM_GITLAB \
  --executor docker \
  --description "Docker Runner" \
  --docker-image "docker:stable" \
  --docker-privileged \
  --docker-extra-hosts "gitlab:IP_FROM_SERVER_WHERE_COMPOSE_IS_RUNNING" \
  --docker-volumes "/var/run/docker.sock:/var/run/docker.sock"
```

If everything went fine you should see a console output like this:
```console
Runner registered successfully. Feel free to start it, but if it's running already the config should be automatically reloaded! 
```
and see the runner in Gitlabs backend when you refresh [https://gitlab/admin/runners](https://gitlab/admin/runners).

***Great, you registered your runner***

## Create a application repository
Now we will make a fresh project repository. Let's call it `application` in our new Gitlab.  
For this example we will use a basic `NGINX` webserver. So create a new file called `Dockerfile` in your new repository with this content:
```bash
FROM nginx:latest
RUN echo "<h1>Deployment</h1>" > /usr/share/nginx/html/index.html
```

That's all for our basic application, cos we want to test the deployment and dont want to make a big application.

## Enable the runner for the new repository
Again got to [https://gitlab/admin/runners](https://gitlab/admin/runners) and "edit" the runner for taking effect on our new project (enable it).

## Create pipeline / CI file
Our runner need to know what to exactly to do for our project. That we will define in a file called `.gitlab-ci.yml`.
Create this file in our application repository and after finish push the new changes to the repo.
```yml
image: docker:stable

variables:
  DOCKER_DRIVER: overlay2

build:
  stage: build
  script:
    - docker login -u gitlab-ci-token -p $CI_JOB_TOKEN $CI_REGISTRY
    - docker build -t gitlab:4567/root/application:dev .
    - docker push gitlab:4567/root/application:dev
```

**you can find the instructions to pull/push to the gitlab docker registry in the registry tab of the repo navigation.**

And if you now push the changes to Gitlab, the pipeline will start as soon as Gitlab receive your push. 
Go to `CI / CD` --> `pipelines` in Gitlab menu of your repo and check your build state.

GREAT   
***You just run your first docker deployment***

## Test the image
No its time test the image locally. First we will login to our gitlab registry:
```bash
docker login gitlab:4567
```

after that we will pull the image:
```bash
docker pull gitlab:4567/root/application:dev
```

and now run it:
```bash
docker run -p "8080:80" gitlab:4567/root/application:dev
```

Now you should see a output on [http://localhost:8080/](http://localhost:8080/) like this:
```bash
Deployment
```

## SSL
I have added some dummy SSL certificates and a root CA certificate to let the systems trust the certificates.  
You should generate and use your own certificates! (see docker-compose.yml volumes)  
See my [Local SSL repository](https://github.com/thobaier/ssl-local-development) for generating your own certs.