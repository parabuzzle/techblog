---
layout: post
title:  Automated Docker Deployments with CircleCI
date:   2015-09-07 12:00:00
categories: Docker
preview: Over the weekend I decided to setup one of my opensource projects to build its docker image, push it to the registry, and deploy that new container to my digital ocean server using CircleCI. It turned out to be super simple and I wanted to share all the details
tags: [CD, CI, CircleCI, Continious Deployment, Docker]
og_type: article
og_description: Over the weekend I decided to setup one of my opensource projects to build its docker image, push it to the registry, and deploy that new container to my digital ocean server using CircleCI. It turned out to be super simple and I wanted to share all the details
disqus_id: 12
---

Over the weekend I decided to setup one of my opensource projects to build its docker image, push it to the registry, and deploy that new container to my digital ocean server using CircleCI. It turned out to be super simple and I wanted to share all the details, so I figured... its time for a blog post! :smile:

Once we've finished here, CircleCI will:

  * build your docker image
  * test that docker image
  * push that docker image to DockerHub
  * deploy & run that docker image to a server

So, first things first, if you don't have a CircleCI account, go get yourself one: [https://circleci.com/signup](https://circleci.com/signup)

Next, choose a project. You need one that's dockerized and can be deployed to a server (ie: a webservice container). Then wire up that repo to do builds in CircleCI (follow their docs for this)

Now that we have builds going we can configure our `circle.yml` and our deployment script.

In the root of your repo's directory create a `circle.yml` file that looks like this:

~~~ yaml
machine:
  services:
    - docker

dependencies:
  override:
    - docker info
    - docker build -t parabuzzle/devopsdayssv-webapp .

test:
  override:
    - docker run -d -p 4567:4567 parabuzzle/devopsdayssv-webapp; sleep 10
    - curl --retry 10 --retry-delay 5 -v http://localhost:4567/schedule.json
~~~

What this config file is telling CircleCI to do is build the docker image as a dependency for this project. Then in the `test` section you can see that its running the built image, and testing that its responding properly. (you need to replace all my stuff with your stuff for your project of course)

Once this builds, We should add the next piece to the file which is the deployment section, like this:

~~~ yaml
machine:
  services:
    - docker

dependencies:
  override:
    - docker info
    - docker build -t parabuzzle/devopsdayssv-webapp .

test:
  override:
    - docker run -d -p 4567:4567 parabuzzle/devopsdayssv-webapp; sleep 10
    - curl --retry 10 --retry-delay 5 -v http://localhost:4567/schedule.json

deployment:
  hub:
    branch: master
    commands:
      - docker login -e $DOCKER_EMAIL -u $DOCKER_USER -p $DOCKER_PASS
      - docker push parabuzzle/devopsdayssv-webapp
~~~

For this you need to add some environment variables to your project in the config section on CircleCI. You will need to add `DOCKER_EMAIL`, `DOCKER_USER`, & `DOCKER_PASS` so CircleCI can login to DockerHub and push your image to the registry. As you can see, we have a named deployment here called `hub` and for every build of the `master` branch it will login and push your freshly built container as the `latest` tag.

But that still doesn't deploy the newly built container to your server(s). So we need to add another file into our repo. We'll call this `deploy.sh` and for ease, we'll also put it in the top level of the repo. (make sure you make it executable `chmod +x deploy.sh`)

~~~ bash
#!/usr/bin/env bash

echo "stopping running application"
ssh $DEPLOY_USER@$DEPLOY_HOST 'docker stop dodsv'
ssh $DEPLOY_USER@$DEPLOY_HOST 'docker rm dodsv'

echo "pulling latest version of the code"
ssh $DEPLOY_USER@$DEPLOY_HOST 'docker pull parabuzzle/devopsdayssv-webapp:latest'

echo "starting the new version"
ssh $DEPLOY_USER@$DEPLOY_HOST 'docker run -d --restart=always --name dodsv -p 80:4567 parabuzzle/devopsdayssv-webapp:latest'

echo "success!"

exit 0
~~~

This is where the magic goes. You need to add 3 more things to your project on CircleCI. First, you need to add the 2 environment variables for `DEPLOY_USER` & `DEPLOY_HOST`. (I leave multiple host support as an exercise for you.) And lastly you need to add your ssh key for the user you set in `DEPLOY_USER` in the `ssh permissions` section for your project.

You can see by this script that we are logging into the host and stopping and removing the current running container. Then we're pulling down the new version from DockerHub. And lastly, we're running that container. (Again, I leave rolling restarts, high-availability, and multi-version support to you :smile:)

Neat huh? But we're not done yet! We need to tell CircleCI to run this script in our `circle.yml` file!

So let's open that file back up and make some changes:

~~~ yaml
machine:
  services:
    - docker

dependencies:
  override:
    - docker info
    - docker build -t parabuzzle/devopsdayssv-webapp .

test:
  override:
    - docker run -d -p 4567:4567 parabuzzle/devopsdayssv-webapp; sleep 10
    - curl --retry 10 --retry-delay 5 -v http://localhost:4567/schedule.json

deployment:
  production:
    branch: master
    commands:
      - docker login -e $DOCKER_EMAIL -u $DOCKER_USER -p $DOCKER_PASS
      - docker push parabuzzle/devopsdayssv-webapp
      - ./deploy.sh
~~~

As you can see, we've changed the deployment name from `hub` to `production` because it seems more appropriate now. We'll still key off of builds of `master` and we still login and push to DockerHub... but we added a new command at the end. Its our new deployment script! Now when we run a build of `master` we will build a docker container, test that docker container, dist that docker container to DockerHub, and deploy that docker container to our server. :boom: BOOM :boom: DevOps.
