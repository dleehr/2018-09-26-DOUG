## You're not root here

### Adapting Dockerized applications for Openshift

Dan Leehr

Duke Openshift Users Group | September 12, 2018
---
![docker-logo](docker-logo.png "Docker")

- Docker image is the currency
- Build, ship, run

---
## Application: Bespin
Collection of services to run reproducible computational workflows on Openstack cloud.

[github.com/Duke-GCB/bespin](https://github.com/Duke-GCB/bespin)

<img style="width: 65%" src="lando-diagram.png"></img>

---

## Step 1: Building (Dockerizing)

- 1 Service -> 1 Container
- Docker Hub: Official images or auto-build `Dockerfile`s
  - `postgres:9.5`, `dukegcb/bespin-api`
- Config and secrets to `ENV` vars
- Docker images = 💰
---

## Step 2: Deployment

- List of images and versions
  - `dukegcb/bespin-api:1.2.2`
  - `dukegcb/lando:0.8.0`
  - `postgres:9.5`
- List of VMs where they get deployed
  - Ansible inventory/playbooks/roles
---

## Step 2: Deployment

Ansible playbooks: [Duke-GCB/gcb-ansible-roles](https://github.com/Duke-GCB/gcb-ansible-roles)
```
- name: Create app container
  docker_container:
    image: "dukegcb/bespin-api:{{ bespin_settings.web.version }}-apache"
    name: bespin-web
    env: "{{ web_environment }}"
    volumes:
      - /etc/external/:/etc/external/
      - "{{ bespin_settings.ui.srv_dir }}:/srv/ui/"
    ports:
      - "80:80"
      - "443:443"
    pull: true
    etc_hosts: "{{ bespin_hosts_list }}"
    state: started
    restart_policy: always
```

---

## Step 3: Automation ¯\\_(ツ)_/¯

Not really automated.

1. Release code (`git tag`)
2. Wait for Docker Hub to build tagged image
3. Edit, commit, push ansible changes
4. Run `ansible-playbook`

k8s runs Docker images, and I *really* like those, so let's dig in with a...

---
## How's that working for you?

may drop this slide

<table>
  <tr><th>Pros</th><th>Cons</th></tr>
  <tr><td>Versioned Deployments</td><td>Time Consuming</td></tr>
  <tr><td>Portable Images</td><td>Manual Rollbacks</td></tr>
  <tr><td>Quick Startup/Upgrade</td><td>Inefficient scheduling</td></tr>
</table>


we'd been meaning to try kubernetes so...

---

# Hackday

![k8s-hackday](k8s-hackday.png "Kubernetes Hackday")

Kubernetes Hackday!

---

## Openshift:

- Sits on top of Kubernetes
  - `kubectl apply -f mystuff.yaml`
- Delivers PaaS features
  - Deployment from source code
  - Catalog of applications
- Demos from Chris and Darin
  - CLI and Web UI

---

## Creating Applications

`$ oc new-app ...`

or

![openshift-ui-new](openshift-ui-new.png "Openshift UI - New Application")

---

## Just what I wanted

Generate an openshift "app" using a docker image and save the YAML out.

- Up and running quickly (like a GUI)
- Editable, understandable, versionable configs
- More attractive than writing k8s YAML out longhand

---

## oc new-app

```
Create a new application by specifying source code,
templates, and/or images

This command will try to build up the components of an
application using images, templates, or code that has a
public repository. It will lookup the images on the local
Docker installation (if available), a Docker registry, an
integrated image stream, or stored templates.
```

---

## Let's run API and DB

I've already got Docker images, **how hard could it be**?

- `oc new-app <image>` for each service
- Tweak YAML for config/secrets
- This will be done in like 5 minutes, right?<!-- .element: class="fragment" -->
```
# Generate DeploymentConfigs from docker images
oc new-app postgres:9.5 \
    --dry-run=true -o yaml > bespin-db.yml
oc new-app dukegcb/bespin-api:1.2.2-apache \
    --dry-run=true -o yaml > bespin-api.yaml
# Tweak YAML
oc create -f bespin-db.yml
oc create -f bespin-api.yml
```
<!-- .element: class="fragment" -->

---

## Nope

```
initdb: could not change permissions of directory "/var/lib/postgresql/data": Operation not permitted
fixing permissions on existing directory /var/lib/postgresql/data ...
```

```
(13)Permission denied: AH00023: Couldn't create the
  rewrite-map mutex (file /var/lock/apache2/rewrite-map.19)
AH00016: Configuration Failed
Action '-DFOREGROUND' failed.
```

![crash-loop](crash-loop.png "Postgres Crash Loop")

---

## You're Not Root Here

> **By default, OKD runs containers using an arbitrarily assigned user ID.** This provides additional security against processes escaping the container due to a container engine vulnerability and thereby achieving escalated permissions on the host node.

[docs.okd.io/latest/creating_images/guidelines.html](https://docs.okd.io/latest/creating_images/guidelines.html)
---
## Ruh-Roh

- **postgres** official:
  - `useradd -r -g postgres --uid=999`
- **bespin-api:1.2.2-apache**
  - `PORT` 80/443, Apache with mod_wsgi
  -

_Why can't I just run **my docker images**, isn't that the whole point?_

---
## No

_Why can't I just run **my docker images**, isn't that the whole point?_

1. **Security**. Principle of least privilege should still apply, even inside containers.
2. Docker is an implementation detail.

_The point is to build, deploy, update, rollback, and scale applications easily._

---

## Starting over

Focus on three services:

<table>
  <tr><th>Application</th><th>Purpose</th></tr>
  <tr><td>PostgreSQL</td><td>Relational Database</td></tr>
  <tr><td>bespin-api</td><td>REST API Server (Python)</td></tr>
  <tr><td>bespin-ui</td><td>Web UI (JS)</td></tr>
</table>

---

## Running PostgresQL

**Goal** Run a Postgres 9.5 database

**Solution** Use the Openshift-compatible **sclorg** image:

```
oc new-app centos/postgresql-95-centos7 \
  --dry-run=true -o yaml > bespin-db.yml
```
<!-- .element: class="fragment" -->


![postgres-running](postgres-running.png "Postgres Running")<!-- .element: class="fragment" -->

---

## Running Bespin-API

**Goal**: Run my Python+Apache+JavaScript image, connect to Postgres.

**Solution**:

1. Don't do that.<!-- .element: class="fragment" -->
2. Just run the Python application.<!-- .element: class="fragment" -->

---
## Running Bespin-API

**Goal**: Run my Python application, connect it to Postgres.

- Openshift is PaaS, give it source code!
- Uses **S2I** (Source to Image) to build app images
- **S2I** can be installed locally for development.

<div>
```
$ s2i build <code_dir> centos/python-27-centos7 bespin-api
$ docker run bespin-api whoami
default
```

_Look Ma, no Dockerfile and no root_
</div>
<!-- .element: class="fragment" -->

---
## Changes to Bespin API

- Delete the `Dockerfile`
- Use gunicorn instead of Apache/mod_wsgi
- Set build-time ENV vars in `.s2i/environment`

_Encouraging better practices_

---
## Running Bespin API

```
oc new-app \
  python:2.7~https://github.com/Duke-GCB/bespin-api \
  --dry-run=true -o yaml > bespin-api.yml
```

<div>
![bespin-api-build](bespin-api-build.png "Bespin API Build")<!-- .element: class="fragment" -->
</div><!-- .element: class="fragment" -->

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

