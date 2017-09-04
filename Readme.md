## Introduction

Containers continue to be a buzz word this year too. A lot of teams try to implement microservices,
dividing application into individual units: mini- website contains only presentation layer, REST API (sometimes several REST APIs) handling all business logic. Databases are hosted externally, in some situations running in containers too. Database — sometimes one, sometimes many (depending on business task)

During development teams also perform testing using Docker. Docker is giving them confidence when testing that we are on the same “environment” as on production (althouth it is not always true). For deployments to shared environments teams usually use docker-compose to quickly start up whole environment with all services. This method is good  — we define in one file what services to run (images, mapping volumes, etc.), what ports to expose from docker container to the host system.

If we are using some docker friendly platform like Openshift, Kubernetes, Amazon EC2 Container Service, Mesos etc. But sometimes we have some simple project, when using mentioned services is possible, but is quite overkilling yet. At that moment, we are launching some VPS, installing docker on it and a bit of mess starts when we expose services to a web. We want to have some kind of reverse proxy that will accept traffic coming to specific sub-domain and route that traffic to its appropriate docker environment.

I wouldn't be mistaken , if I say that you must have seen portions of configs similar to one below.

```
upstream demo1.localhost {
    server 172.17.0.4:5000;
    server 172.17.0.3:5000;
}

server {
    #ssl_certificate /etc/nginx/certs/demo.pem;
    #ssl_certificate_key /etc/nginx/certs/demo.key;

    gzip_types text/plain text/css application/json application/x-javascript
               text/xml application/xml application/xml+rss text/javascript;

    server_name demo1.domain.com;

    location / {
        proxy_pass http://demo.localhost;
        include /etc/nginx/proxy_params;
    }
}
```

Reverse proxy can be executed in many ways, we can make custom service, we can use Nginx as above, but it would be really nice if for such smaller projects there could be easy configurable tool, with dynamic discovery of new subdomains, loadbalancing etc.  In NodeJS world PM2 came. 

In docker world - once of the recent options is Traefik (traefik.io), written in Go language that promises to help us with that. What is good,- project has got big round of investments in 2017 (https://www.crunchbase.com/organization/containous#/entity)  which might be sign,that project will not be abandoned quick. Anyway you always can return back to nginx at any moment


## Setting up traefik as a reverse proxy alternative to nginx on a single host environment

Traefik sets quite ambitious goals:  it is positioned as dynamic reverse proxy. It has bridges also to many popular deployment platforms (docker, openshift, mezos, kubernetes, etc.) and synchronizes information about running services (containers). This means, that your project might grow or change infrastructure without major downtime or refactoring.

Traefik is configured by rules that are used to connect “Frontends” with “Backends”. In terms of Traefik — “Frontend” is a public end point at some internet domain like api.myapp.com. On the other hand “Backend” is our deployed web (docker) service. For example,  we can set a rule in Traefik that for Host:api.myapp.com traffic should be routed to our api service container.

Let's prepare our environment to demonstrate. Manual installation steps can be found on https://docs.traefik.io/. We would use Ansible provisioner to get automated approach for our future deployments.

We would need two ansible roles: sa-docker (https://github.com/softasap/sa-docker - docker and docker-compose) and sa-traefik(https://github.com/softasap/sa-traefik - traefik tool itself)

```yaml

     - {
         role: "sa-docker",
         tags: ["create"]
       }
     - {
         role: "sa-traefik",
         tags: ["create"]
       }


```

Important notice on setup - sa-traefik tool implements managing of the configuration using conf.d configuration pieces. I.e. you don't need to manage whole configuration file, but instead you can modify only needed parts.  By default, traefik is installed "naked". 

Let's add three critical configuration pieces:
`conf.d/entrypoints.toml` - serve http by default.
```
defaultEntryPoints = ["http"]
[entryPoints]
    [entryPoints.http]
    address = ":80"
```

`conf.d/web_backend.toml` - administration UI on port 8080 guarded by admin/test & test2/test
```
[web]

address = ":8080"

[web.auth.basic]
  users = ["admin:$apr1$H6uskkkW$IgXLP6ewTrSuBkTrqE8wj/", "test2:$apr1$d9hr9HBB$4HxwgUir3HP4EsggP/QNo0"]
```

`conf.d/docker.toml` - instruct traefik to use docker as a configuration source.
```
# Enable Docker configuration backend
[docker]
# Docker server endpoint. Can be a tcp or a unix socket endpoint.
endpoint = "unix:///var/run/docker.sock"

# Default domain used.
domain = "docker.localhost"

# Enable watch docker changes
watch = true

# Expose containers by default in traefik
# If set to false, containers that don't have `traefik.enable=true` will be ignored
exposedbydefault = true

# Use Swarm Mode services as data provider
swarmmode = false
```

If everything is ok, you should see traefik configuration backend at :8080


![alt text](https://raw.githubusercontent.com/Voronenko/devops-traefik-deploy-template/master/docs/images/traefik_empty.png "just launched")


## Launching first application with traefik

What is good, is that you can use tool that you know - docker-compose to configure traefik by definition.
Let's take a look on the following example (we will use docker-compose v2 syntax, because at the current time Ansible docker_service module does not fully support v3):

```
version: '2'

services:
  whoami1:
    image: emilevauge/whoami
    networks:
      - web
    labels:
      - "traefik.backend=whoami"
      - "traefik.domain=xenial.test"
      - "traefik.frontend.rule=Host:whoami.xenial.test"
      - "traefik.frontend.entryPoints=http"
      - "traefik.enable=true"
networks:
  web:
    external:
      name: traefik_webgateway

```

Pay attention to labels that start with traefik. Namely it means: "Service whoami1 belongs to backend(application) called whoami, it serves on public domain xenial.test on port http/80 , responfs to hostname whoami.xenial.test and should be handled by traefik reverse proxy".

Let's do docker-compose up on example above. As you see, traefik was immediately notified with changes.

![alt text](https://raw.githubusercontent.com/Voronenko/devops-traefik-deploy-template/master/docs/images/traefik_app1.png "first app")


Let's navigate to website according to rule?  It works!

![alt text](https://raw.githubusercontent.com/Voronenko/devops-traefik-deploy-template/master/docs/images/traefik_app1_response.png "first app response")


Summary - now we are aware how to run and configure our application to be served with traefik.

## Loadbalancing with traefik

Traefik also supports load balancing. If we amend docker-compose definition in a way below

```
version: '2'

services:
  whoami1:
    image: emilevauge/whoami
    networks:
      - web
    labels:
      - "traefik.backend=whoami"
      - "traefik.domain=xenial.test"
      - "traefik.frontend.rule=Host:whoami.xenial.test"
      - "traefik.frontend.entryPoints=http"
      - "traefik.enable=true"
  whoami2:
    image: emilevauge/whoami
    networks:
      - web
    labels:
      - "traefik.backend=whoami"
      - "traefik.domain=xenial.test"
      - "traefik.frontend.rule=Host:whoami.xenial.test"
      - "traefik.frontend.entryPoints=http"
      - "traefik.enable=true"
networks:
  web:
    external:
      name: traefik_webgateway

```

We will notice a change: now single end point is served by two different services.

![alt text](https://raw.githubusercontent.com/Voronenko/devops-traefik-deploy-template/master/docs/images/traefik_1app_2services.png "one app two services")


More over, even in example 1, if we would use `docker-compose up --scale whoami1=2`  (or in older docker-compose  `docker-compose scale whoami1=2`), we would notice that also was correctly handled.

![alt text](https://raw.githubusercontent.com/Voronenko/devops-traefik-deploy-template/master/docs/images/traefik_1app_scaled_services.png "one app scaled services")

## Deploying traefik enabled applications

As you see at that moment you are free to decide who will you manage your applications. You would definitely like to introduce some rolling deployment process. You can use ansible play too for this purposes, execute shell batches, or even manually.

Ansible - template your docker-compose.yml and use docker_service module to launch
```
#  Using templated file and docker_service module

    - name: Application | Template docker-compose.yml for app 1
      template: src="{{playbook_dir}}/templates/docker-compose/application1/docker-compose.yml.j2" dest="{{base_application_dir}}/application1/docker-compose.yml"
      become: yes
      tags:
        - app

    - name: Application | Deploy using compose file
      docker_service:
        project_src: "{{base_application_dir}}/application1/"
        build: no
        restarted: true
      register: app1_output

    -  debug: var="app1_output"

#  / Using templated file and docker_service module
```

Ansible - using direct definition (hard to read for myself)
```
#  Using definition directly in docker_service directive
    - name: Application | Deploy using definition in docker_service
      docker_service:
        build: no
        restarted: true
        project_name: application2_by_def
        definition:
          version: '2'
          services:
            whoami1_by_def:
              image: emilevauge/whoami
              networks:
                - web
              labels:
                - "traefik.backend=whoami_by_def"
                - "traefik.domain=xenial.test"
                - "traefik.frontend.rule=Host:whoami_by_def.xenial.test"
                - "traefik.frontend.entryPoints=http"
                - "traefik.enable=true"
            whoami2_by_def:
              image: emilevauge/whoami
              networks:
                - web
              labels:
                - "traefik.backend=whoami_by_def"
                - "traefik.domain=xenial.test"
                - "traefik.frontend.rule=Host:whoami_by_def.xenial.test"
                - "traefik.frontend.entryPoints=http"
                - "traefik.enable=true"
          networks:
            web:
              external:
                name: traefik_webgateway
      register: app2_output
      become: yes

    -  debug: var="app1_output"

#  / Using definition directly
```

Using templated file and invoking console command: 
```
#  Using templated file and invoking console command

    - name: Application | Template docker-compose.yml for app 3
      template: src="{{playbook_dir}}/templates/docker-compose/application3/docker-compose.yml.j2" dest="{{base_application_dir}}/application3/docker-compose.yml"
      become: yes
      tags:
        - app

    - name: Application | Put in action using docker-compose command
      shell: "docker-compose up -d"
      args:
        chdir: "{{base_application_dir}}/application3"
      become: yes
      tags:
        - app

#  / Using templated file and invoking console command
```

## Conclusion

Traefik is young project, for easy configuration you should pay. And payment's currency is rps (request per second)

On a dummy test with 100 concurrent requests using apache benchmark and HAproxy and traefik acting as simple loadbalancer between two containers (figures by Michael Mueller using test image coreos/example:1.0.0).

```
platform                        HAproxy    Traefik(binary)    Traefik(container)     
50% requests completed (ms)       21             91                   93 
95% requests completed (ms)       72            146                  147 
request per s (mean)             2485,11       1105,15              1072,29
```

This is a bit different from figures given on project's website - https://docs.traefik.io/benchmarks/ , and it is expectable.
You would anyway stress test your application before live run, wouldn't you?

How it will behave on a real production?  Well, you should give it a try. At least, for sure, it is good candidate for any project, where high load is not expected quick.