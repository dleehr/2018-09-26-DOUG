# Intro

- I like docker for building parts of applications
- Build an image on one system, pack it up and run it somewhere else



# Example application: bespin

https://github.com/duke-gcb/bespin

- half-dozen services,
- some in-house, some off-the-shelf (RabbitMQ, Postgres)
- UI <-> API <-> [Database, MQ, Workers], 
- Pretty standard application

Dockerizing an Application

- Goal of each container is to provide one service (usually a process and probably listening on a port)
- Find an official for common services (database, message queue, key-value store). Documentation and tutorials really leans into this
- Write your own Dockerfile to install your application and its dependencies.
- Write a docker-compose (or similar) file that links these services together, and provides service discovery. 
- provide for injecting passwords and private keys

# What was I building and how do I deploy it

- docker images for each service (dukegcb/bespin-api:apache, postgres:9.5, etc)
- deployed to a fleet of VMs with ansible `docker_container` module
- similar to docker-compose 
https://github.com/Duke-GCB/gcb-ansible-roles/blob/fb387b7f8f2ba7dec2fd7ffe25b8f0b947e5ca4e/bespin_web/tasks/run-server.yml#L27-L41

# Deployment strategy

- Auto build docker images from GitHub with Docker Hub
- Create a release, wait for docker hub
- Then run ansible to load out 

Advantages:

- infrastructure as code
- easy migration to other servers (compared to manually installing packages)
- All the other docker advtangages (dependencies, restart times, etc)

Drawbacks:
- Not automated. time consuming to wait for builds.
- Rollbacks are possible but require deliberate and specific versioning
- No real record of who did what when (unless we move to ansible tower or something)
- Not terribly efficient (containers scheduled to single VMs)

Feels like a temporary solution, and we'd been meaning to try kubernetes so...

# Hackday

Kubernetes Hackday!

Well, OIT is experimenting with openshift, and that runs k8s under the hood. 

https://github.com/Duke-GCB/hackday-bespin-api-devcloud (private repo)

Start with just the API service and the underlying database. How hard could it be? I mean I've already got docker images for these things.

```
oc new-app

Create a new application by specifying source code, templates, and/or images 

This command will try to build up the components of an application using images, templates, or code that has a public
repository. It will lookup the images on the local Docker installation (if available), a Docker registry, an integrated
image stream, or stored templates. 

If you specify a source code URL, it will set up a build that takes your source code and converts it into an image that
can run inside of a pod. Local source must be in a git repository that has a remote repository that the server can see.
The images will be deployed via a deployment configuration, and a service will be connected to the first public port of
the app. You may either specify components using the various existing flags or let new-app autodetect what kind of
components you have provided. 

If you provide source code, a new build will be automatically triggered. You can use 'oc status' to check the progress.
```


```
oc new-app postgres:9.5 --dry-run=true -o yaml > bespin-db.yml
oc new-app dukegcb/bespin-api:apache --dry-run=true -o yaml > bespin-api.yaml
# edit the files to fill in usernames/passwords/etc
oc create -f bespin-db.yml
oc create -f bespin-api.yml
```

Generate an openshift "app" using a docker image and save the YAML out.

- This is a great technique that Darin London showed us
- One of the many things that I find tough about k8s is the idea that you'll sit down and write a bunch of YAML off the top of your head.
- Sure there's tools that handle that for you, but I wanted something developer friendly, so right away this is a good start.

# Quickly hit a wall on both services.

The docker images I was using assume root

- Official postgres image assumes a privileged user
- My bespin-api image included apache on ports 80/443 to serve production django, do TLS termination, and serve the JS app too

That's a bummer. I already dockerized, why can't I just run my docker images, isn't that the whole point?

No. The point isn't to run docker images. Docker is an implementation detail. The point is to use that to make our lives easier.

# Back to basics

3 main Services:

1. Postgres Database
2. Bespin API - python django application
3. Bespin UI - Ember JS Javascript application

And a TLS route for a single secure hostname to API and UI


## Postgres

How do I run postgres on openshift?

It's in the catalog:

oc new-app centos/postgresql-95-centos7 

- Edit the YAML to map secrets to environment variables

How does this change the deployment?

[postgres image] -RUN-> [pod]

- well, if I decide to upgrade to a new version of postgres, i can change out that version number and re-deploy.

## Bespin API

How do I run my apache+python+javascript image in openshift?

Well, don't. Break it up. Just run the python application. 

- Openshift started as a PAAS like heroku, designed to be able to build applications from source
- Turns out it still does that, but now it does it with S2I, kubernetes and Docker.
- [source repo] + [base image] -> build -> [app image]

[python image] + [source code] -BUILD-> [app image] -RUN-> [pod]

What did I have to change?

- Delete dockerfile
- added gunicorn and whitenoise - gunicorn is a web server for python applications and whitenoise handles the static files/
- put some required build-time variables in `.s2i/environment`

oc new-app python:2.7~https://github.com/dleehr/bespin-api#openshift 

<graphic here>

How does this change the deployment?

- `oc start-build bespin-api` is one way. That'll pull the latest github code, build a new docker image, and 

3. Bespin UI

How do I build and serve my javascript frontend in openshift?

Bespin UI is an Ember application. Node/NPM are required to build, but the output is compiled HTML/JS that should be served statically.

- Need node to build, but don't want it in the final application image
- two step builds!
  - Build 1: Docker image centos/nodejs-8-centos7 + https://github.com/dleehr/bespin-ui - outputs files in `dist/` directory
  - Build 2: 2 line Dockerfile. Uses httpd as base image (centos/httpd-24-centos7:latest), and copies `dist/` from build 1. Produces an image with just apache and the static HTML

[nodejs image] + [source code] -BUILD-> [artifact] + [http image] -BUILD-> [app image] -RUN-> [pod]

VVVVVVVVVV


Making that work for production

- Picking on Django here but you probably have something similar.
- Python app? Start with python base image
- Django's included webserver is not production-quality. Doesn't terminate TLS and won't serve static files
- Decisions, decisions: Do I want to add more containers to do these things that seem like the job of this one?
- I've used django with apache+mod_wsgi before. that works, so `apt-get install apache` in my Dockerfile
- Wait a minute! am I running python or apache? Making a mess, better document that and provide all kinds of configs
- And now I need to write a script to start apache and clean up PID files? RED FLAG!
- When do I rebuild this container? what's the inheritance graph?
- Rails people are probably telling me I picked the wrong framework.

Simple guidelines meet real world requirements. Swallow a lot of complexity for the "simplicity" of portable docker images

- Web application? It should serve HTTP on a port. Development? Make the source code a volume
- Database? grab official postgres and move on
- tie it all together with docker-compose, easy right?

