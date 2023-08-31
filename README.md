# Django Helm Chart 


Django Helm Chart with [Celery](https://github.com/celery/celery), [Celery-Beat](https://docs.celeryq.dev/en/stable/reference/celery.beat.html), [Flower](https://flower.readthedocs.io/en/latest/) and [Redis]((https://redis.io)).
The chart also includes deployments for Celery setup (Worker, Flower.

- Deployment includes a multi container pod with Django and Caddy. [Caddy](https://caddyserver.com) is used as sidecar to serve the static files of backend. Caddyfile can be found [here](https://github.com/sanoguzhan/django-helm-chart/blob/master/django/templates/caddy.yaml).

- The chart includes [Redis](https://redis.io) as an optional subchart.

- The Deployment containers init containers for migrations and staticfiles.


This is my personal take on such a type of chart, thus I might not use the best practices or you might disagree with how I do things. Any and all feedback is greatly appreciated!

## Deployment setup

Set values for Django backend on Deployment Section of values.yaml

**Example:**

```yaml
replicaCount: 1
revisionHistoryLimit: 2
image:
  repository: ghcr.io/sanoguzhan/django-helm-test
  pullPolicy: IfNotPresent
  # Overrides the image tag whose default is the chart appVersion.
  tag: "latest"
  containerPort: 8081
command: 'gunicorn blog.wsgi -b 0.0.0.0:8081'
```

_Note:_ Set the command as a string instead of a list.
## Static Files

Static files collect is done by initContainers.
- **staticfiles:** Add command for Django to collect static files.

```yaml
collect_static:
  enabled: false
  name: staticfiles
  command: "python3 manage.py collectstatic --noinput"
```

Static files and Media directories should be given under data:

```yaml
data:
  staticfiles: "app/staticfiles/"
  data_media: "app/data_media/"
```

Paths should be relative to the working directory of main container.
Static files are shared volume (emptyDir) between initContainer and Proxy container.


## Migrations

Migrations are done with a Job. ServiceAccount, configMap and secret values are loaded before application container starts. Job is deleted after completion.

```yaml
db_migrations:
  enabled: true
  name: db-migration
  command: "python3 manage.py migrate --noinput"
  resources: {}
  safeToEvict: true

```

## ConfigMaps and Secrets

ConfigMaps and Secret values should be set under:

```yaml
# Env ConfigMap values 
envConfigs: {}
# Enc Secret Values (Base64 encoded data)
envSecrets: {}

```


## Celery Setup 

Celery setup includes worker, flower and beat. Celery Beat is optional, set enabled to false if you want to exclude it. Flower includes ingress setup optionally. 

```yaml
celery:
# Celery Beat Values
  beat:
    enabled: true
    componentName: celery-beat
    command: 'celery -A blog.celery beat -l DEBUG'
    replicaCount: 1
    strategy: Recreate
# Celery Worker Settings  
  worker:
    componentName: celery-worker
    command: 'celery -A blog.celery worker -l DEBUG'
    replicaCount: 1
    strategy: RollingUpdate
# Celery Flower Settings
  flower:
    componentName: celery-flower
    command: 'celery -A blog.celery flower'
    replicaCount: 1
    strategy: RollingUpdate   
    service:
      type: ClusterIP
      port: 
        number: 5555
        name: flower
    ingress:
      enabled: false
      annotations: {}
      hosts:
        - host: example.com
          paths:
            - path: / 
              pathType: Prefix
      tls: []  
```

## Redis 

Redis is optional sub-chart, which could be disabled by setting enabled to false. 
[Bitnami Redis Helm Chart](https://github.com/bitnami/charts/tree/master/bitnami/redis) is used, and version of the chart is given in the Chart.yaml file.


```yaml
redis:
  enabled: true
```

Build chart dependency includes

```sh 
helm dependency build ./django
```

## Getting Started

### Use the Github template

First, click the green `Use this template` button near the top of this page.
This will take you to Github's ['Generate Repository'](https://github.com/sanoguzhan/django-helm-chart/generate) page.
Fill in a repository name and short description, and click 'Create repository from template'.
This will allow you to create a new repository in your Github account,
prepopulated with the contents of this project.

Now you can clone the project locally and get to work!

    git clone https://github.com/<user>/<your_new_repo>.git