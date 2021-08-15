# s3-exposer-helm

## Description

This is a helm chart that deploy a s3-proxy inside the s3-exposer namespace.

[S3-proxy](https://github.com/pottava/aws-s3-proxy) is a project created by [Ryo Nakamaru](https://github.com/pottava) among other contributors under the [MIT License](https://github.com/pottava/aws-s3-proxy/blob/master/LICENSE)

Will work with the provided service account maped to a iam role whithout any hardcoded credentials.

The chart expect to have an ingress controller (nginx), cert-manager and external-dns already deployed on the cluster.

## Usage

1. Copy and edit the values.yaml replacing the aws, ingress, issuer and secret values to fill your needs
1. Run the following command
```
helm install s3-exposer . -f my-values.yaml
```

## Check
```
$ kubectl get pods -n s3-exposer
NAME                        READY   STATUS    RESTARTS   AGE
s3-proxy-64dccbb848-xcbxb   1/1     Running   0          4m
```
```
$ kubectl get ingress -n s3-exposer
NAME                   CLASS    HOSTS                ADDRESS                                                                   PORTS     AGE
proxy-assets-ingress   <none>   foo.tryland.com.es   ab59ff1c6cb064b29a2fa22a39fdd905-1383442750.eu-west-1.elb.amazonaws.com   80, 443   4m3s
```

## How it works

This chart will deploy an s3-proxy pod, a service, ingress, issuer and a secret (for basic auth).
When the Ingress is generated, external-dns will create the defined dns record on route 53. At the same time the cert-manager will generate a certificate issued by let's encrypt.

Once an http requests will be received by a load balancer and if it match the host defined on the chart, it will be forwarded to the nginx ingress controller. The next step is a service, that finally send the request to the s3-proxy pod.



The pod will translate the http requests to aws s3 requests and return through http the requested object. From this point the answer will return the previous path in reverse order.

![schema](img/s3-proxy.png?raw=true)

## Generate secret for basic auth

```
$ htpasswd -c auth foo
New password: <bar>
New password:
Re-type new password:
Adding password for user foo
```
```
$ kubectl create secret generic basic-auth-nginx --from-file=auth
secret "basic-auth" created
```
```
$ kubectl get secret basic-auth-nginx -o yaml
apiVersion: v1
data:
  auth: Zm9vOiRhcHIxJE9GRzNYeWJwJGNrTDBGSERBa29YWUlsSDkuY3lzVDAK
kind: Secret
metadata:
  name: basic-auth
  namespace: default
type: Opaque
```

## Chart parameters

Name                                                             | Description                                              | Value
-----------------------------------------------------------------|----------------------------------------------------------|-------------------------------------------------------
deployment.access_log                                            |Enables requests loggin                                   | true
deployment.aws_region                                            |Bucket aws region                                         | eu-west-1
deployment.aws_s3_bucket                                         |Bucket name                                               | your-bucket
replicaCount                                                     |Number of replicas                                        | 1
image.repository                                                 |Container source                                          | pottava/s3-proxy
ingress.enabled                                                  |Generate an ingress                                       | true
ingress.className                                                |Ingress controller to be used                             | ""
ingress.annotations.kubernetes.io/ingress.class                  |Ingress controller to be used                             | nginx
ingress.annotations.nginx.ingress.kubernetes.io/auth-type        |Enables basic auth                                        | basic
ingress.annotations.nginx.ingress.kubernetes.io/auth-secret      |Secret name who stores the auth information               | basic-auth-nginx
ingress.annotations.cert-manager.io/issuer                       |Issuer name used to generate certificates                 | letsencrypt
ingress.hosts.host                                               |Domain where the bucket will be exposed                   | foo.bar.com.es
ingress.hosts.paths.path                                         |Path where the bucket will be exposed                     | /
ingress.tls.secretName                                           |Secret name who stores the TLS certificate                | foo-tls
ingress.tls.hosts[0]                                             |Domain used to generate the TLS certificate               | foo.bar.com.es
issuer.email                                                     |Email required by let's encrypt                           | foo@bar.com
issuer.server                                                    |Let's encrypt server used to generate the TLS certificate | https://acme-staging-v02.api.letsencrypt.org/directory
secret.auth                                                      |Hashed credentials used by the basic auth                 | auth: Zm9vOiRhcHIxJFJVUm1QSC9XJGhWeDdodlA4cGMxWHBpajRENHR1Wi8K #foo:bar

## Limitations

1. This chart was tested using kubernetes 1.20, on other kubernetes version could cause some problems.
1. Tested with nginx ingress controller on and eks kubernetes cluster.
1. There is no support for cluster autscaller yet.
1. The number of replicas is manually defined.
1. Only one bucket could be exposed. If you need to expose more buckets install the chart again with a different name and different values.
1. You cannot decide yet the service account name