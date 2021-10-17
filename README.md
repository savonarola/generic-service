# Generic Service Helm Chart

I somewhat like Kubernetes (but not much 😅) and Helm, which allows
install, deploy and remove apps easily.

But I definitely do not like the boilerplate of modifying tons YAML files or
their mustache templates.

Also, most of my apps are quite simple: they have some
* Pods
* Cronjobs
* Services
* Ingresses
and that is all.

So I made a generic Helm chart which is configured through a generic value file.

The downside is that this chart does not have any app-specific semantics, so
if an installation grows, a new boilerplate will appear.

But in my experience
* I used this chart tens of times
* Almost never missed anything for my apps
* My value files never grew big. Below is somewhat the most extensive example I have ever had.

## Value File Example

```yaml
name: doghouse-mortage
instance: prod
namespace: doghouse-services

imagePullSecrets:
- name: dogcker-secret

env:
  - name: RAILS_ENV
    value: "production"

secretRefs:
  - doghouse-mortage-secret

ingress:
  enabled: true
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: 100m
  hosts:
    - host: doghouse-mortage.com
      paths:
        - path: /
          serviceName: web
    - host: admin.doghouse-mortage.com
      paths:
        - path: /
          serviceName: web

services:
  - name: web
    port: 80
    portName: http
    component: web
    targetPort: http


deployments:

  - name: web
    replicaCount: 1
    volumes:
      - name: nginx-conf
        configMap:
          name: doghouse-mortage-nginx
    containers:
      - name: app
        image: docker.io/dog-bank/doghouse-mortage
        imageTag: latest # otherwise, appVersion is used, see Usage
        resources:
          limits:
            cpu: 1000m
            memory: 512Mi
          requests:
            cpu: 100m
            memory: 128Mi
        env:
          - name: PORT
            value: "8081"
      - name: nginx
        image: docker.io/dog-bank/doghouse-mortage-nginx
        volumeMounts:
          - mountPath: /etc/nginx/conf.d/nginx.conf
            name: nginx-conf
            subPath: nginx.conf
        ports:
          - name: http
            containerPort: 8080
            protocol: TCP
        resources:
          limits:
            cpu: 100m
            memory: 64Mi
          requests:
            cpu: 10m
            memory: 32Mi

  - name: clockwork
    replicaCount: 1
    containers:
      - name: app
        image: docker.io/dog-bank/doghouse-mortage
        resources:
          limits:
            cpu: 1000m
            memory: 512Mi
          requests:
            cpu: 100m
            memory: 128Mi
        args: ["bundle", "exec", "clockwork", "config/clock.rb"]


  - name: sidekiq
    replicaCount: 1
    initContainers:
      - name: db-migrate
        image: docker.io/dog-bank/doghouse-mortage
        resources:
          limits:
            cpu: 1000m
            memory: 256Mi
          requests:
            cpu: 100m
            memory: 128Mi
        args: ["bundle", "exec", "rake", "db:migrate"]

    containers:
      - name: app
        image: docker.io/dog-bank/doghouse-mortage
        resources:
          limits:
            cpu: 1000m
            memory: 512Mi
          requests:
            cpu: 100m
            memory: 128Mi
        args: ["bundle", "exec", "sidekiq", "-C", "config/sidekiq.yml"]


cron_jobs:
  - name: cleaner
    schedule: "*/5 * * * *"
    containers:
      - name: app
        image: docker.io/dog-bank/doghouse-mortage
        resources:
          limits:
            cpu: 300m
            memory: 512Mi
          requests:
            cpu: 50m
            memory: 128Mi
        args: ["bash", "/app/run-cleaner.sh"]


configmaps:
  - name: nginx
    data:
      nginx.conf: |-
        server {
          listen 8080;

          deny all;

          location ~ ^/(images|javascripts|stylesheets|assets)/ {
            root /app/public;
            break;
          }

          location / {
            try_files $uri @app;
          }

          location /favicon.ico {
            return 404;
          }

          location @app {
            proxy_pass       http://127.0.0.1:8081;

            proxy_set_header Host             $host;
            proxy_set_header X-Real-IP        $remote_addr;
            proxy_set_header X-Forwarded-For  $proxy_add_x_forwarded_for;

            proxy_set_header  X-Forwarded-Proto $scheme;
            proxy_set_header  X-Forwarded-Ssl on;
            proxy_set_header  X-Forwarded-Port 443;
            proxy_set_header  X-Forwarded-Host $host;
          }
        }

```

## Usage

Assume you have an app called `doghouse-mortage` of version 0.1.2.

* Make value file for your service, e.g. `doghouse-mortage.yaml`.
* Run installation command
```bash
helm3 upgrade --install doghouse-mortage https://github.com/savonarola/generic-service/archive/refs/tags/v0.0.1.tar.gz --set appVersion=0.1.2 -f doghouse-mortage.yaml
```

## LICENSE

[MIT](LICENSE)
