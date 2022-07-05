# Magento2 Helm Chart

Easily deploy Magento2 in Kubernetes including MySQL, Elasticsearch, Redis, RabbitMQ and Varnish.
The project includes best-of-breed charts from Bitnami, Elasticsearch and others to give maximum flexibility for deploying and
configuring the services.

The chart has been battle tested in Magento2 OpenSource and Adobe Commerce production environments. The bundled `values.yaml` provides basic settings, which should be adjusted before deployment.

## Magento2 base image
The chart references [PHOENIX MEDIA's](https://www.phoenix-media.eu) Magento2 Docker image. It consists of an
[Alpine nginx+PHP8.1 base image](https://github.com/PHOENIX-MEDIA/docker-nginx-php), [Magento OpenSource 2.4.4 source code](https://github.com/magento/magento2)
and a few Composer packages to add [build+deploy scripts](https://github.com/PHOENIX-MEDIA/magento2-cloud-build). The image is available on [DockerHub](https://hub.docker.com/r/phoenixmedia/magento2)
and the source code including Github Action is available in the [magento2-build repository](https://github.com/PHOENIX-MEDIA/magento2-build).

For more information checkout the article [Running Magento2 in Kubernetes —
Part 2: Building the Docker Image](https://medium.com/@bjoern.kraus/running-magento2-in-kubernetes-part-2-building-the-docker-image-8516c0ed7d48).

## Magento ECE-Tools
> ECE-Tools is a set of scripts and tools designed to manage and deploy Cloud projects.

Using [ECE-Tools](https://github.com/magento/ece-tools/) for building and deploying the Docker image reduces the amount
of custom scripts and also gives great flexibility to adjust process as needed.
It is recommended to get familiar with its [build and deploy mechanisms](https://devdocs.magento.com/cloud/project/magento-env-yaml.html).

It is required to review and adjust the [Magento Cloud environment variables](https://devdocs.magento.com/cloud/env/variables-cloud.html)
for the `magento` and `cronjob` deployment in the `values.yaml:
- MAGENTO_CLOUD_ROUTES
- MAGENTO_CLOUD_RELATIONSHIPS
- MAGENTO_CLOUD_VARIABLES

Their values are simply base64 encoded JSON objects. To decode them either run `echo "<value>" | base64 -d` or use an
[online decoder](https://www.base64decode.org/) (beware when using sensitive data).

> **_Note:_** It is best practice to maintain sensitive values in a [Kubernetes Secret](https://kubernetes.io/docs/concepts/configuration/secret/) instead of keeping them in the `values.yaml`.

## Ingress

Before enabling the ingress make sure to configure `host.name` and the TLS certificate properly. When having
[cert-manager](https://cert-manager.io/docs/) installed set `ingress.certManager: true` to automatically generate a
certificate for the application.

If you prefer to skip Varnish for certain routes simply configure addtional paths:

```
    paths:
    - path: "/"
      serviceName: varnish
      servicePort: 80
    - path: "/pub/media"
      serviceName: magento
      servicePort: 80
```

In case you want to protect the Magento backend by IP or BasicAuth we recommend to duplicate the `templates/ingress.yaml`
(e.g. to your Helm project root) and configure a second Ingress with proper annotations.

## SMTP server
supervisord starts a simple Postfix MTA included in the [Alpine base image](https://github.com/PHOENIX-MEDIA/docker-nginx-php).
However, you should configure a mail relay which accepts mails for your Magento instance and is eligible to send emails for
the configured store email addresses.

Make sure to configure it for the `magento` and `cronjob` deployment:

```
    env:
    - name: RELAYHOST
      value: my.relayhost.com
    - name: SMTP_USE_TLS
      value: "true"
```


## Sample data
For fresh installations it is possible to install Magento's sample data. This requires [Magento Marketplace](https://devdocs.magento.com/guides/v2.4/install-gde/prereq/connect-auth.html)
authentication keys to be configured. They can be set in the environment variables of the cronjob deployment:

```
  env:
    - name: ADD_SAMPLE_DATA
      value: "true"
    - name: COMPOSER_AUTH
      value: |-
        {
          "http-basic": {
            "repo.magento.com": {
              "username": "<public key>",
              "password": "<private key>"
            }
          }
        }

```

## Xdebug
Debugging in a remote environment can help to bring down resolution time for complex issues. Since Xdebug has to
establish a TCP connection to the IDE, the network setup can become complex.

While this Helm chart won't solve the complexity of setting up a VPN and maybe [DBGp proxy](https://xdebug.org/docs/dbgpProxy)
it deploys an additional container which has Xdebug enabled. In order to route traffic to this Xdebug-enabled Magento container
you have to adjust the Varnish VCL prepared in the `values.yaml`:

```
      # whitelist your developer IPs
      acl xdebug-users {
          "192.168.0.0/24";
      }

      # uncommend these lines
      # use xdebug backend
      #    if (req.http.cookie ~ "XDEBUG_SESSION=" && std.ip(req.http.X-Real-IP, "0.0.0.0") ~ xdebug-users) {
      #        set req.backend_hint = magento_director.backend("xdebug");
      #        return (pass);
      #    }
```

For any request from the whitelisted networks which has the XDEBUG_SESSION request cookie (we use [Xdebug Chrome Extension](https://chrome.google.com/webstore/detail/xdebug-chrome-extension/oiofkammbajfehgpleginfomeppgnglk))
the request will be sent to the `xdebug` pod instead of the normal `magento` pods.

In addition the settings for the `xdebug` workload have to be adjusted in the `values.yaml`:

```
xdebug:
  enabled: true
  reuseMagentoEnvs: true
  env:
    - name: XDEBUG_INSTALL
      value: "true"
    - name: XDEBUG_REMOTE_HOST
      value: "dbgp-proxy"
```

> **_Caution:_** Running Xdebug in a public environment can be a security issue. Enable this functionality at your own risk.

## Helm deployment
The chart requires [Helm 3.x](https://helm.sh/) and has been tested with 3.9.0.
Make sure to adjust the `values.yaml` before deployment.

The Helm command is straight forward:

`helm upgrade -i --create-namespace -n my-namespace magento .`

Deploying the whole Magento2 stack is complex operation and not unlikely when trying it the first time. We prefer to use
optional `--wait --timeout 15m` parameters in a deployment pipeline to see if the deployment was actually successful.

To deploy Magento to different environments (develop, staging, production) it is recommended to create a `values_*.yaml`
for each environment and tune the resource limits and configuration values of the services.

## Changelog
### [2.4.0] - 2022-06-24
- Use PHOENIX MEDIA's Magento OpenSource 2.4.4 build as base image
- Updated Bitnami charts
- Use Softonic Varnish chart
- Use upstream Varnish version
- Cleaned up values.yaml

### Changelog
The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).