# OSM the Very First Run
* TOC
{:toc}
## OSM with AKS
[OSM](https://github.com/openservicemesh/osm) which claims to be lightweight and extensible, now can be one-click integrated into AKS as add-on.
It sounds promising but I do spend quite sometime running just a demo app, OSM is kind of new at the moment and documentation is not covering all scenarios. I'm using the latest [v1.0.0](https://github.com/openservicemesh/osm/releases/tag/v1.0.0) version, and I do hope there will be more wiki and trouble shooting guide coming out before next version.

>Prerequisites
- AKS cluster with 
    [OSM add-on installed](https://docs.microsoft.com/en-us/azure/aks/open-service-mesh-deploy-addon-az-cli)
    [Ingress Nginx installed](https://docs.microsoft.com/en-us/azure/aks/ingress-basic?tabs=azure-cli#basic-configuration)  
- OSM [CLI](https://release-v1-0.docs.openservicemesh.io/docs/guides/cli/)
- A bit understanding of how ingress nginx works  
- A bit understanding of **permissive traffic policy mode** versus **SMI traffic access policies**

## Exploring OSM with demo whoami
Why [whoami](https://hub.docker.com/r/containous/whoami)? Simply because it accepts all paths/prefix so you don't need to spend time struggling with path regex annotations. 

### whoami as a Loadbalancer Service
1. `kubectl create ns osm`
2. `kubectl create deployment whoami --image=containous/whoami -n osm`
3. `kubectl expose deployment whoami --port=80 --type=LoadBalancer  -n osm` 
4. [Optional, only required if you want whoami expose within virtual network] `kubectl patch services whoami -n osm -p '{"metadata":{"annotations":{"service.beta.kubernetes.io/azure-load-balancer-internal": "true"}},"spec":{"type":"LoadBalancer"}}'`
5. Check the accessibility, e.g. `curl whoami_service_ip`, and you will get something like:
```
Hostname: whoami-84f56668f5-xr7ln
IP: 127.0.0.1
IP: ::1
IP: 10.240.0.57
IP: fe80::7860:3fff:fef7:9909
RemoteAddr: 127.0.0.1:43006
GET / HTTP/1.1
Host: 20.200.91.235
User-Agent: curl/7.58.0
Accept: */*
X-Envoy-Internal: true
X-Forwarded-For: 10.240.0.4
X-Forwarded-Host: 20.200.91.235
X-Forwarded-Port: 80
X-Forwarded-Proto: http
X-Forwarded-Scheme: http
X-Real-Ip: 10.240.0.4
X-Request-Id: e48425ac977c250d36c352bb68680fda
X-Scheme: http
```

### add osm namespace to service mesh
> Things start to break here with osm mesh enabled:
- Add the osm namespace to the mesh and verify if envoy side-car is injected
`osm namespace add osm`
- If you curl the same IP again you will find the whoami is no longer accessible, because any service within OSM must use an ingress gateway/controller to be exposed outside the cluster. and you could use:
  - [Nginx-Ingress](https://release-v1-0.docs.openservicemesh.io/docs/demos/ingress_k8s_nginx/) In the next section.
  - [Contour](https://release-v1-0.docs.openservicemesh.io/docs/demos/ingress_contour/) I didn't try myself because nginx is good(~tough~) enough for me.
## Expose your service with Ingress Nginx
Suppose we have ingress-controller installed in **ingress-basic** namespece(or anywhere else just don't mess up with **kube-system** and **osm**)

1. Ask Ingress controller if it is ready by
   `curl Ingress_Controller_Service_IP` and expect a happy 404 from the nginx:
    ```
    <html>
    <head><title>404 Not Found</title></head>
    <body>
    <center><h1>404 Not Found</h1></center>
    <hr><center>nginx</center>
    </body>
    </html>
    ```
2. Let OSM monitor the ingress namespace while not hurting it
    
    `osm namespace add "$INGRESS_NAMESPACE"  --mesh-name "$OSM_MESH_NAME"  --disable-sidecar-injection`

    for our example, it is 
    
    `osm namespace add ingress-basic  --mesh-name osm --disable-sidecar-injection`
3. Have Ingress rules defined for whoami:

    ```
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: whoami
      namespace: osm
      annotations:
        nginx.ingress.kubernetes.io/use-regex: "true" # optional
        nginx.ingress.kubernetes.io/rewrite-target: / # no need for whoami actually, it eats everything
    spec:
      ingressClassName: nginx
      rules:
      - http:
          paths:
          - path: /whoami
            pathType: Prefix
            backend:
              service:
                name: whoami
                port:
                  number: 80
    ```
At this stage, if you`curl Ingress_Controller_Service_IP/whoami`, you will get **502 Bad Gateway**, because whoami service disallows reqeust from our ingress controller.
I love ingress nginx logs because is it always helps to understand what's going on:

```
[error] 102#102: *179260 recv() failed (104: Connection reset by peer) while reading response header from upstream, 
client: 10.240.0.36, server: _, request: "GET /whoami HTTP/1.1", upstream: "http://10.240.0.57:80/", host: "20.200.91.235"
10.240.0.36 - - [02/Apr/2022:10:40:03 +0000] "GET /whoami HTTP/1.1" 502 150 "-" 
"curl/7.58.0" 83 0.001 [osm-whoami-80] [] 
10.240.0.57:80, 10.240.0.57:80, 10.240.0.57:80 0, 0, 0 0.000, 0.000, 0.000 502, 502, 502 4c2590c86069efbe0584960568480bfc
```
It is saying my whoami pod 10.240.0.57 refused the traffic from ingress controller, we should be happy to see the log because this is what OSM is supposed to do right?

4. To allow the traffic, need to define IngressBackend:
```
kind: IngressBackend
apiVersion: policy.openservicemesh.io/v1alpha1
metadata:
  name: whoami
  namespace: osm
spec:  
  backends:  
  - name: whoami  
    port:
      number: 80 
      protocol: http
  sources:
  - kind: Service
    namespace: ingress-basic
    name: ingress-nginx-controller
```

apply it then it's done,  `curl Ingress_Controller_Service_IP/whoami` you will see whoami responding again.

## About Backend Application Protocol
I've been successful with almost every single applications, but failed with the [azure vote app](https://docs.microsoft.com/en-us/azure/aks/kubernetes-walkthrough#run-the-application).
However I tried it just return with 500 Internal Server Error:
```
<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 3.2 Final//EN">
<title>500 Internal Server Error</title>
<h1>Internal Server Error</h1>
<p>The server encountered an internal error and was unable to complete your request.  Either the server is overloaded or there is an error in the application.</p>
```

The azure-vote-front log is complaining something about **Protocol Error: H, b'TTP/1.1 400 Bad Request**
```
[pid: 15|app: 0|req: 2/2] 127.0.0.1 () {50 vars in 637 bytes} [Sun Apr 10 10:39:31 2022] GET / => generated 950 bytes in 1 msecs (HTTP/1.1 200) 2 headers in 80 bytes (1 switches on core 0)
[2022-04-10 10:39:34,382] ERROR in app: Exception on / [GET]
Traceback (most recent call last):
  File "/usr/local/lib/python3.6/site-packages/flask/app.py", line 1982, in wsgi_app
    response = self.full_dispatch_request()
...
  File "/usr/local/lib/python3.6/site-packages/redis/connection.py", line 292, in read_response
    (str(byte), str(response)))
redis.exceptions.InvalidResponse: Protocol Error: H, b'TTP/1.1 400 Bad Request'
```
This means when azure-vote-front sending the request to the backend redis(azure-vote-back), the protocol was changed by OSM. And turns out the current OSM only supports HTTP traffic and [TCP is not yet supported](https://github.com/openservicemesh/osm/issues/1521).   
In order to send TCP protocol to backend redis, we need to do some trick(adding **appProtocol: TCP** ) on the azure-vote-back service telling OSM to send him TCP:
```
apiVersion: v1
kind: Service
metadata:
  name: azure-vote-back
  ...
  ports:
  - port: 6379
    appProtocol: TCP
```
Remember to delete the azure-vote-front to force reconnection, it is not designed to retry itself.
Sadly I cannot find any explanation about appProtocol except for this one [application protocol selection](https://release-v1-0.docs.openservicemesh.io/docs/guides/app_onboarding/app_protocol_selection/).

## What's coming next
observability with dashboard

egress exploration

mTLS
