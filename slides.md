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
- Ansible playbooks & inventory
  - Small VMs run 1-2 containers
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

---
![better-way](better-way.jpg "You mean there's a better way?")

Kubernetes runs Docker images, let's dig in with a ...
---
# Hackday

![k8s-hackday](k8s-hackday.png "Kubernetes Hackday")

Kubernetes Hackday!
---

## Openshift:

- Is a distribution of **Kubernetes**
  - `kubectl` and YAML
- Adds Developer friendly features
  - Deploy from source code (PaaS)
- `oc` CLI and Web interface
  - Productive and Easy

---

## Creating Applications

`$ oc new-app ...`

or

![openshift-ui-new](openshift-ui-new.png "Openshift UI - New Application")

---

## Protip (via Darin London)

1. Generate YAML to deploy each Docker image using `oc new-app`
2. Version/edit that YAML locally
3. Deploy those applications using `oc create`

_Compared to writing longhand or k8s: 👍_

---

## Let's run API and DB

I've already got Docker images

- This will be done in like 5 minutes, right?

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

- **postgres:9.5**
  - `useradd -r -g postgres --uid=999`
- **bespin-api:1.2.2-apache**
  - `PORT` 80/443, Apache with mod_wsgi
  - Permissions in `/var/lock/apache2/`

_Why can't I just run **my docker images**, isn't that the whole point?_

---
## No

_Why can't I just run **my docker images**, isn't that the whole point?_

1. **Security**. Principle of least privilege should still apply, even inside containers.
2. Docker is an implementation detail.

_The point is to build, deploy, update, rollback, and scale applications easily._

---

## Starting over

Focus on three services, real-world use cases:

<table>
  <tr><th>Service</th><th>Purpose</th></tr>
  <tr><td>postgres</td><td>Relational Database</td></tr>
  <tr><td>bespin-api</td><td>REST API Server (Python)</td></tr>
  <tr><td>bespin-ui</td><td>Web UI (HTML/JS)</td></tr>
</table>

---

## Running Postgres

**Goal**: Run a Postgres 9.5 database

<div>
**Solution**: Use the Openshift-compatible **sclorg** image:
</div><!-- .element: class="fragment" -->


<div>
```
oc new-app centos/postgresql-95-centos7 \
  --dry-run=true -o yaml > bespin-db.yml
```
_Also what you get by choosing Postgres in the catalog_
</div><!-- .element: class="fragment" -->

![postgres-running](postgres-running.png "Postgres Running")<!-- .element: class="fragment" -->

---

## Running Bespin-API

**Goal**: Run my Python+Apache+JavaScript image, connect it to Postgres.

**Solution**:

<div>
1. Don't do that.
2. Just run the Python application.
</div><!-- .element: class="fragment" -->

---
## Running Bespin-API

**Goal**: Run my Python application, connect it to Postgres.

- Openshift is PaaS, give it source code!
- **S2I** (Source-To-Image) builds Docker images
- **S2I** can be installed locally for development.

<div>
```
$ s2i build \
  https://github.com/Duke-GCB/bespin-api \
  centos/python-27-centos7 \
  bespin-api
$ docker run bespin-api whoami
default
```
_Look Ma, no Dockerfile and no root_
</div>
<!-- .element: class="fragment" -->

---
## S2I Required changes

- Delete the `Dockerfile`
- Use gunicorn instead of Apache/mod_wsgi
- Set build-time ENV vars in `.s2i/environment`

_Docker is just an implementation detail_

---
## Creating Openshift Build

```
oc new-app \
  python:2.7~https://github.com/Duke-GCB/bespin-api \
  --dry-run=true -o yaml > bespin-api.yml
```

<div>
![bespin-api-build](bespin-api-build.png "Bespin API Build")<!-- .element: class="fragment" -->
</div><!-- .element: class="fragment" -->

---
## How is this different?

- For postgres, we just pull and run the Docker image
- For bespin-api, Openshift builds the Docker image, then runs it

<div>
Builds can be started from CLI:

    oc start-build bespin-api

_...or via webhook, when the base image updates, or when you click **build**_
</div><!-- .element: class="fragment" -->

---

## Running Bespin-UI
3. Bespin UI

**Goal**: Serve my HTML/JS application

**Considerations**:

<div>
- Node is required to build the app, but not to run it
- Should serve from same domain as API (CORS/XSS)
</div><!-- .element: class="fragment" -->

---
## Two-stage builds

Can we use S2I with a second stage?

<img/>

<draw this>


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

