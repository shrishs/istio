# istio mTLS
- This example is based upon https://istio.io/docs/tasks/traffic-management/secure-ingress/

- Install the following example
https://docs.openshift.com/container-platform/3.11/servicemesh-install/servicemesh-install.html#installing-bookinfo-application

```
 git clone https://github.com/nicholasjackson/mtls-go-example

 cd mtls-go-example

 ./generate.sh istio-singressgateway-istio-system.apps.82c0.example.opentlc.com xxxxxxxx

 mkdir istio-singressgateway-istio-system.apps.82c0.example.opentlc.com

 mv 1_root 2_intermediate 3_application 4_client istio-singressgateway-istio-system.apps.82c0.example.opentlc.com
```
- Create the passhrough route on the existing openshift-istio service(istio-ingressgateway).
```
oc create route passthrough istio-singressgateway --hostname=istio-singressgateway-istio-system.apps.82c0.example.opentlc.com --service=istio-ingressgateway --port=443 -n istio-system
```
## Configure a TLS ingress gateway

```
oc create -n istio-system secret tls istio-ingressgateway-certs --key 3_application/private/istio-singressgateway-istio-system.apps.82c0.example.opentlc.com.key.pem --cert 3_application/certs/istio-singressgateway-istio-system.apps.82c0.example.opentlc.com.cert.pem
```
-- Gateway definition is as follows.Make sure tls information is added for the corresponding host in server section.
```
apiVersion: v1
items:
- apiVersion: networking.istio.io/v1alpha3
  kind: Gateway
  metadata:
    annotations:
    generation: 1
    name: bookinfo-gateway
    namespace: myproject
  spec:
    selector:
      istio: ingressgateway
    servers:
    - hosts:
      - '*'
      port:
        name: http
        number: 80
        protocol: HTTP
    - hosts:
      - istio-singressgateway-istio-system.apps.82c0.example.opentlc.com
      port:
        name: https
        number: 443
        protocol: HTTPS
      tls:
        mode: SIMPLE
        privateKey: /etc/istio/ingressgateway-certs/tls.key
        serverCertificate: /etc/istio/ingressgateway-certs/tls.crt
kind: List
metadata:
  resourceVersion: ""
  selfLink: ""
```

- Check the browser-it will display the product page
   https://istio-singressgateway-istio-system.apps.82c0.example.opentlc.com/productpage

``` 
export INGRESS_HOST=$(oc -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
```
```
curl -v -HHost:istio-singressgateway-istio-system.apps.82c0.example.opentlc.com --resolve istio-singressgateway-istio-system.apps.82c0.example.opentlc.com:443:$INGRESS_HOST --cacert 2_intermediate/certs/ca-chain.cert.pem https://istio-singressgateway-istio-system.apps.82c0.example.opentlc.com/productpage
```
``` 
curl -v -HHost:istio-singressgateway-istio-system.apps.82c0.example.opentlc.com --resolve istio-singressgateway-istio-system.apps.82c0.example.opentlc.com:443:$INGRESS_HOST --cacert 2_intermediate/certs/ca-chain.cert.pem https://istio-singressgateway-istio-system.apps.82c0.example.opentlc.com/productpage
* Added istio-singressgateway-istio-system.apps.82c0.example.opentlc.com:443:172.29.162.78 to DNS cache
* About to connect() to istio-singressgateway-istio-system.apps.82c0.example.opentlc.com port 443 (#0)
*   Trying 172.29.162.78...
* Connected to istio-singressgateway-istio-system.apps.82c0.example.opentlc.com (172.29.162.78) port 443 (#0)
* Initializing NSS with certpath: sql:/etc/pki/nssdb
*   CAfile: 2_intermediate/certs/ca-chain.cert.pem
  CApath: none
* SSL connection using TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
* Server certificate:
* 	subject: CN=istio-singressgateway-istio-system.apps.82c0.example.opentlc.com,O=Dis,L=Springfield,ST=Denial,C=US
* 	start date: Nov 19 05:10:09 2018 GMT
* 	expire date: Nov 29 05:10:09 2019 GMT
* 	common name: istio-singressgateway-istio-system.apps.82c0.example.opentlc.com
* 	issuer: CN=istio-singressgateway-istio-system.apps.82c0.example.opentlc.com,O=Dis,ST=Denial,C=US
> GET /productpage HTTP/1.1
> User-Agent: curl/7.29.0
> Accept: */*
> Host:istio-singressgateway-istio-system.apps.82c0.example.opentlc.com
>
< HTTP/1.1 200 OK
< content-type: text/html; charset=utf-8
< content-length: 5719
< server: envoy
< date: Mon, 19 Nov 2018 05:58:58 GMT
< x-envoy-upstream-service-time: 38
```
## Configure a mutual TLS ingress gateway
- Create secret with ca cert.
oc create -n istio-system secret generic istio-ingressgateway-ca-certs --from-file=2_intermediate/certs/ca-chain.cert.pem


- Change the gateway definition to mutualTLS
```
apiVersion: v1
items:
- apiVersion: networking.istio.io/v1alpha3
  kind: Gateway
  metadata:
    name: bookinfo-gateway
    namespace: myproject
  spec:
    selector:
      istio: ingressgateway
    servers:
    - hosts:
      - '*'
      port:
        name: http
        number: 80
        protocol: HTTP
    - hosts:
      - istio-singressgateway-istio-system.apps.82c0.example.opentlc.com
      port:
        name: https
        number: 443
        protocol: HTTPS
      tls:
        caCertificates: /etc/istio/ingressgateway-ca-certs/ca-chain.cert.pem
        mode: MUTUAL
        privateKey: /etc/istio/ingressgateway-certs/tls.key
        serverCertificate: /etc/istio/ingressgateway-certs/tls.crt
kind: List
```

- Try the same command above and it will fail as it require client certificate.

``` curl -v -HHost:istio-singressgateway-istio-system.apps.82c0.example.opentlc.com --resolve istio-singressgateway-istio-system.apps.82c0.example.opentlc.com:443:$INGRESS_HOST --cacert 2_intermediate/certs/ca-chain.cert.pem https://istio-singressgateway-istio-system.apps.82c0.example.opentlc.com/productpage
* Added istio-singressgateway-istio-system.apps.82c0.example.opentlc.com:443:172.29.162.78 to DNS cache
* About to connect() to istio-singressgateway-istio-system.apps.82c0.example.opentlc.com port 443 (#0)
*   Trying 172.29.162.78...
* Connected to istio-singressgateway-istio-system.apps.82c0.example.opentlc.com (172.29.162.78) port 443 (#0)
* Initializing NSS with certpath: sql:/etc/pki/nssdb
*   CAfile: 2_intermediate/certs/ca-chain.cert.pem
  CApath: none
* NSS: client certificate not found (nickname not specified)
* NSS error -12227 (SSL_ERROR_HANDSHAKE_FAILURE_ALERT)
* SSL peer was unable to negotiate an acceptable set of security parameters.
* Closing connection 0
curl: (35) NSS: client certificate not found (nickname not specified)
```

- Resend the previous request by curl, this time passing as parameters your client certificate (the --cert option) and your private key (the --key option):
```
curl -v -HHost:istio-singressgateway-istio-system.apps.82c0.example.opentlc.com --resolve istio-singressgateway-istio-system.apps.82c0.example.opentlc.com:443:$INGRESS_HOST --cacert 2_intermediate/certs/ca-chain.cert.pem --cert 4_client/certs/istio-singressgateway-istio-system.apps.82c0.example.opentlc.com.cert.pem --key 4_client/private/istio-singressgateway-istio-system.apps.82c0.example.opentlc.com.key.pem https://istio-singressgateway-istio-system.apps.82c0.example.opentlc.com/productpage

- curl -v -HHost:istio-singressgateway-istio-system.apps.82c0.example.opentlc.com --resolve istio-singressgateway-istio-system.apps.82c0.example.opentlc.com:443:$INGRESS_HOST --cacert 2_intermediate/certs/ca-chain.cert.pem --cert 4_client/certs/istio-singressgateway-istio-system.apps.82c0.example.opentlc.com.cert.pem --key 4_client/private/istio-singressgateway-istio-system.apps.82c0.example.opentlc.com.key.pem https://istio-singressgateway-istio-system.apps.82c0.example.opentlc.com/productpage
* Added istio-singressgateway-istio-system.apps.82c0.example.opentlc.com:443:172.29.162.78 to DNS cache
* About to connect() to istio-singressgateway-istio-system.apps.82c0.example.opentlc.com port 443 (#0)
*   Trying 172.29.162.78...
* Connected to istio-singressgateway-istio-system.apps.82c0.example.opentlc.com (172.29.162.78) port 443 (#0)
* Initializing NSS with certpath: sql:/etc/pki/nssdb
*   CAfile: 2_intermediate/certs/ca-chain.cert.pem
  CApath: none
* NSS: client certificate from file
* 	subject: CN=istio-singressgateway-istio-system.apps.82c0.example.opentlc.com,O=Dis,L=Springfield,ST=Denial,C=US
* 	start date: Nov 19 05:10:12 2018 GMT
* 	expire date: Nov 29 05:10:12 2019 GMT
* 	common name: istio-singressgateway-istio-system.apps.82c0.example.opentlc.com
* 	issuer: CN=istio-singressgateway-istio-system.apps.82c0.example.opentlc.com,O=Dis,ST=Denial,C=US
* SSL connection using TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
* Server certificate:
* 	subject: CN=istio-singressgateway-istio-system.apps.82c0.example.opentlc.com,O=Dis,L=Springfield,ST=Denial,C=US
* 	start date: Nov 19 05:10:09 2018 GMT
* 	expire date: Nov 29 05:10:09 2019 GMT
* 	common name: istio-singressgateway-istio-system.apps.82c0.example.opentlc.com
* 	issuer: CN=istio-singressgateway-istio-system.apps.82c0.example.opentlc.com,O=Dis,ST=Denial,C=US
> GET /productpage HTTP/1.1
> User-Agent: curl/7.29.0
> Accept: */*
> Host:istio-singressgateway-istio-system.apps.82c0.example.opentlc.com
>
< HTTP/1.1 200 OK
< content-type: text/html; charset=utf-8
< content-length: 5723
< server: envoy
< date: Mon, 19 Nov 2018 06:32:09 GMT
< x-envoy-upstream-service-time: 54
```
