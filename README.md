# go-sccs-eureka

Playing with Golang with Spring Cloud Config services and Netflix Eureka

~~~bash
# Initialize the repository
$ (rm -Rf data && \
   mkdir -p ./data/git-repo && \
   cd ./data/git-repo && \
   cp ../../conf/* . && \
   git init . && \
   git add . && \
   git commit -m 'Initial config for SCCS demo')

# Startup eureka and config services
$ docker-compose up -d
~~~

Please take some time to study the files in the created `./data/git-config` directory (in order of overlay)

~~~bash
data
└── git-repo
 1  ├── application.properties         # Main configuration
 2  ├── application-test.properties    # Configuration for all apps in the environment "test"  
 3  ├── myapp.properties               # Application configuration
 4  └── myapp-test.properties          # Configuration for myapp specific to the environment "test"
~~~

* Each key/value pair is overwritten when set in a higher level layer
* Variable substitution works bi-directional; no matter if set in a lower or higher level

See for the latter the example `myapp-test.properties` (level 4) overrides the Elasticsearch port set in `application-test.properties` (level 2) which is substituted in `my-app.properties` (level 3).

## Login to registry

Now, you can login to the service registry (Netflix Eureka) [http://localhost:8761](http://localhost:8761) using `eureka/secret`; note the `SCCS` application (the Spring Cloud Config Server) is in status `UP(1)`.

## Access the Spring Cloud Config API

Next, let's access the configuration API by requesting the configuration for our app `myapp`.

~~~bash
$ curl --silent --user config:secret 'http://localhost:8888/myapp/default' | jq .
~~~

... this would give us a `json` which depicts an environment-agnostic view of the application configuration (only including the `application.properties` and `myapp.properties`) settings:

~~~json
{
  "name": "myapp",
  "profiles": [
    "default"
  ],
  "label": null,
  "version": "c2083fc5095e039fdd4b2bd6d2bfde56fa26f073",
  "state": null,
  "propertySources": [
    {
      "name": "file:///git-repo/myapp.properties",
      "source": {
        "elasticsearch.url": "https://${elastic.user}:${elastic.pass}@${elastic.host}:${elastic.port}",
        "rabbitmq.url": "${rmq.url}",
        "resultset.pagesize": "10"
      }
    },
    {
      "name": "file:///git-repo/application.properties",
      "source": {
        "vendor.id": "rdc",
        "container.platform": "kubernetes"
      }
    }
  ]
}
~~~

By specifying an environment:

~~~bash
$ curl --silent --user config:secret 'http://localhost:8888/myapp/test' | jq .
~~~

...we get the myapp `test` environment specific settings:

~~~json
{
  "name": "myapp",
  "profiles": [
    "test"
  ],
  "label": null,
  "version": "c2083fc5095e039fdd4b2bd6d2bfde56fa26f073",
  "state": null,
  "propertySources": [
    {
      "name": "file:///git-repo/myapp-test.properties",
      "source": {
        "log.level": "debug",
        "elastic.port": "9201"
      }
    },
    {
      "name": "file:///git-repo/application-test.properties",
      "source": {
        "elastic.pass": "secret",
        "elastic.host": "es.my.domain",
        "elastic.user": "appuser",
        "elastic.port": "9200",
        "rmq.url": "amqp://broker.my.domain:5672/myvhost"
      }
    },
    {
      "name": "file:///git-repo/myapp.properties",
      "source": {
        "elasticsearch.url": "https://${elastic.user}:${elastic.pass}@${elastic.host}:${elastic.port}",
        "rabbitmq.url": "${rmq.url}",
        "resultset.pagesize": "10"
      }
    },
    {
      "name": "file:///git-repo/application.properties",
      "source": {
        "vendor.id": "rdc",
        "container.platform": "kubernetes"
      }
    }
  ]
}
~~~

As you can see, all information related to the general (`default`) and environment specifics are returned, including the names of the physical files in the repository.

While this is awesome, your application might just "wanna know its configuration" and you do not want to write complex logic to parse (and overlay) the variables and placeholders...

## Render the application configuration ready for use

Let's now access the API requesting `properties` format:

~~~bash
# Request the (default, environment agnostic) configuration in properties format
$ curl --silent --user config:secret 'http://localhost:8888/myapp-default.properties'
~~~

~~~properties
container.platform: kubernetes
elasticsearch.url: https://${elastic.user}:${elastic.pass}@${elastic.host}:${elastic.port}
rabbitmq.url: ${rmq.url}
resultset.pagesize: 10
vendor.id: rdc
~~~

...  or in `yaml` format:

~~~bash
# Request the (default, environment agnostic) configuration in yaml format
$ curl --silent --user config:secret 'http://localhost:8888/myapp-default.yaml'
~~~

~~~yaml
container:
  platform: kubernetes
elasticsearch:
  url: https://${elastic.user}:${elastic.pass}@${elastic.host}:${elastic.port}
rabbitmq:
  url: ${rmq.url}
resultset:
  pagesize: '10'
vendor:
  id: rdc
~~~

As you can see... it **does not make sense** to request a configuration which is disconnected from its environment, let's try again with the `test` environment directive:

~~~bash
# Request the (default, environment agnostic) configuration in yaml format
$ curl --silent --user config:secret 'http://localhost:8888/myapp-test.yaml'
~~~

~~~yaml
container:
  platform: kubernetes
elastic:
  host: es.my.domain
  pass: secret
  port: '9201'
  user: appuser
elasticsearch:
  url: https://appuser:secret@es.my.domain:9201
log:
  level: debug
rabbitmq:
  url: amqp://broker.my.domain:5672/myvhost
resultset:
  pagesize: '10'
rmq:
  url: amqp://broker.my.domain:5672/myvhost
vendor:
  id: rdc
  ~~~

Now, all is there, rendered as final configuration!
