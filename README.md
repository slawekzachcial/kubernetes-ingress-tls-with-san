# Multi-domain TLS Certificate in Kubernetes Ingress

This repository shows an example use of a TLS certificate with subject
alternative names in Kubernetes Ingress.

The example contains the following manifests:
* Namespace - `echoserver` namespace where all other resources are created
* Deployment - deploys an Echo Server pod
* Service - provides access to the pod
* Secret - contains TLS certificate key and certificate
* Ingress - provides TLS-protected endpoint to the Echo Server

## TLS Certificate

To generate a self signed TLS certificate with subject alternative names of
`*.example.com` (wildcard DNS name) and `example2.com` you can use the
following command:

```
openssl req -x509 -newkey rsa:4096 -sha256 -days 3560 -nodes \
  -keyout tls.key -out tls.crt \
  -subj '/CN=*.example.com' \
  -extensions san -config <(cat << EOF
[req]
distinguished_name=req
[san]
subjectAltName=@alt_names
[alt_names]
DNS.1=*.example.com
DNS.2=example2.com
EOF
)
```

The command creates 2 files: `tls.key` and `tls.crt`.

The content of each of these files must be Base64-encoded (i.e. `base64 tls.crt`)
and put into `secret.yml`, in `data` section.

`ingress.yml` contains the following section to ensure that the Ingress
controller can properly associate the certificate:

```
  tls:
  - hosts:
    - my.example.com
    - example2.com
    secretName: echoserver-tls
```

## Deployment

Once the certificate generated and its content put in `secret.yml` use the
following command to deploy the resources:

```
kubectl apply \
  -f namespace.yml \
  -f deployment.yml \
  -f service.yml \
  -f secret.yml \
  -f ingress.yml
```

## Tests

To test the deployment use the following command:

```
curl --verbose \
  --resolve my.example.com:443:<a_k8s_lb_ip> \
  --cacert tls.crt \
  https://my.example.com
```

Put one of the IP addresses of your Kubernetes load balancer in the `--resolve`
parameter value above. This will allow to use the DNS name of `my.example.com` but
resolve it to the IP address of your Kubernetes cluster.

Using `--cacert tls.crt` prevents the TLS error even though we use a
self-signed certificate.

You can also use `example2.com` name which is present in the certificate as a
subject alternative name:

```
curl --verbose \
  --resolve example2.com:443:<a_k8s_lb_ip> \
  --cacert tls.crt \
  https://example2.com
```

Both of these tests (the `curl` commands) should return the response from the
Echo Server (client, server values and request headers). This proves that TLS
certificate was properly deployed by the Ingress controller and that the
controller is able to server requests coming to any of the subject alternative
names present in the certificate, assuming they were properly listed in
`ingress.yml`.

## Cleanup

To delete all the resources above you can simply delete the namespace where
they were created:

```
kubectl delete -f namespace.yml
```

IMPORTANT: Do not use the command above if you have any other resources present
in `echoserver` namespace as the above command will also delete them.
