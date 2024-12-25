# Description
We deploy and configure a TLS network to use a subdomain called **k8s.oferto.io** to be redirect to a minikube cluster running inside an Ubuntu 24.04 LTS ARM64 Virtual Machine running in our mac m1 host. Also we will deploy several services to test this remote access from internet:

- A simple nginx service deployed in minikube without any security
- A Identity and Access Manager service implemented by Keycloak 26.0.0. We will use the last Bitnami Helm Chart.
- A simple angular application without any security.
- A simple Angular application integrated with Keycloak configured to use a particula realm, cliente, user and roles configured in Keycloak. This application will show these data after login.

## Prerequisites
Before start these tutorial you must prepared these resources:

- A real domain to be tested in out case oferto.io
- A real node (host) to deploy our Virtual Machine in our case a mac m1 node.
- A Virtual Machine Ubuntu 24.04 running inside the mac node
- A Proxy running inside the Virtual Machine in our case HAProxy 3.0
- A Kubernetes Cluster with ingress and optional Dashboard deployed inside. In our case we will user minikube 0.0.45

## Configure your subdomain
First we must to go to our [DNS manager Admin Console](https://account.squarespace.com/domains/managed/oferto.io/dns/dns-settings) and register an **A register** for our subdomain **k8s.oferto.io** redirected to the **public IP of our home router**. You can discover this public ip going to google an search this world: 'what is my ip'.

## Configure Virtual Machine
As we said we used a Virtual Machine Ubuntu 24.04 to deploy the proxy and the minikube cluster inside, but we must set the network connection of this VM as **Bridge** to have a private IP in the same network as the host (Mac node) where the home router is attached too.

## Configure your router
After this, you must configure your home router and add some **NAT rules** to redirect the traffic from internet to your node where the proxy is running. In our case we will add two NAT rules for the ports 80 and 443 to be redirect to our Virtual Machine private IP. Check it inside the VM executing thid command:

```
$ ifconfig
```

## Create a certificate for your subdomain
To have a TLS netowrk we need a real certificat to be use for the proxy. To obtain this real certificate(not auto-signed) we will use the service called **cerbot**. Install certbot service:
```
$ sudo apt install certbot
```

To use the certbot command we must stop the HAProxy service first if is running, and be sure that any service is running in the 80 port right now We can execute this command in our host to check it:
```
$ netstat -na | grep ':80.*LISTEN'
```

Now we can execute certbot to create our subdomain certificate. We must set our email and the subdomain to be certificated an wait some seconds to generate the private key and the certificates as you see:
```
$ sudo certbot certonly --standalone
Saving debug log to /var/log/letsencrypt/letsencrypt.log
Please enter the domain name(s) you would like on your certificate (comma and/or
space separated) (Enter 'c' to cancel): k8s.oferto.io
Requesting a certificate for k8s.oferto.io

Successfully received certificate.
Certificate is saved at: /etc/letsencrypt/live/k8s.oferto.io/fullchain.pem
Key is saved at:         /etc/letsencrypt/live/k8s.oferto.io/privkey.pem
This certificate expires on 2025-01-17.
These files will be updated when the certificate renews.
Certbot has set up a scheduled task to automatically renew this certificate in the background.
We were unable to subscribe you the EFF mailing list. You can try again later by visiting https://act.eff.org.

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
If you like Certbot, please consider supporting our work by:
 * Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
 * Donating to EFF:                    https://eff.org/donate-le
```

## Create the final certificate for your proxy
Now we will combine the two certificates **fullchain.pem** and **privkey.pem** created by certbot for our subdomain **k8.oferto.io** in only one file with prefix pem, and save in the subfolder used by our HAProxy container executing this command:

```
$ sudo DOMAIN='k8s.oferto.io' sudo -E bash -c 'cat /etc/letsencrypt/live/$DOMAIN/fullchain.pem /etc/letsencrypt/live/$DOMAIN/privkey.pem > ${PWD}/certs/$DOMAIN.pem'
```

A file called **k8s.oferto.io.pem** with the private key and certificate conbined will be saved.

## Configure your proxy
Now with the certificates created and located in the certs subfolder, we configure the HAProxy using the **haproxy.cfg** file to redirect the traffic to our minikube cluster. Execute this command to deploy HAProxy using the configuration and the cert just created. This service is a docker container deployed in the host network to share the same network as the host and the home router. Also we will use a volume to reconfigure any rules inside the proxy and reload fast. We must to start the container as root because we will bind the priviledges 80 and 443 ports.

```
$ docker run -d --name k8s-proxy -u root -p 80:80 --volume ${PWD}/haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg --volume ${PWD}/certs/k8s.oferto.io.pem:/etc/ssl/certs/ssl.pem --network host haproxy:3.0-alpine
```

## Deploy the kubernetes services:
As we said we will deploy 4 services:

- The nginx service is a simple nsginx service with default configuration using the 80 port inside the cluster. 
For this service as in the others we deploy a ingress rule to redirect the traffic from HAPrpxy to the nginx service running inside the cluster. Check the folder poc-keycloak-k8s/nginx to check the pod, service and ingress deployed

- Deploy Keycloak using the Bitmani Helm Chart. 
We only configured three attributes for this chart: the admin username and password to have access to the Keycloak Admin Console and the **proxyHeaders** attribute with the value **forwarded**, this value is very important to se set because Keycloak will inspect the forwarded headers seted by the proxy to know the host, protocol, etc. The forwarded protocol is very important because we start our connection from internet using a terminated TLS connection to the proxy. The connection from proxy to kubernetes ingres and later from ingress to kubernetes services is open, but keycloak must to know the real protocol used in orgin because some redurections and requests send by the eframe used by the keycloak login must send this request also using the https protocol because if not a policy created in keycloak to control this conenctions from the eframe will triggered and the conenction will stop. We can see the **values.yaml** in the repo poc-keycloak-k8s/keycloak. Also the ingress for this service is there.

- Deploy a simple angular aplication without any security
In that case the angular app is the default angular scaffolded. We must only add a new build option in the package to build our app in production where defined the subpath used by the ingrees later:

Inb this build option for minikube we define the **base-href** and **deploy-url** parameters with **angular**, becasuse this value is the same used in the ingress fos this service:

```
"scripts": {
    ...
    "build-minikube": "ng build --configuration production --base-href '/angular/' --deploy-url '/angular/'",
    ...
}
```

This is part of the ingress where the value **angular** is repeated
```
spec:
  rules:
    - host: k8s.oferto.io
      http:
        paths:
          - path: /angular(/|$)(.*)
```

The rest of the resources: pod, service and ingress we can check under the repo folder poc-keycloak-k8s/poc-angular

- Deploy a simple angular aplication with security integreated with Keycloak
Finally this sample is an angular app integrated with keycloak, in this case 

The angular app must be configured with:
- **realm**: poc
- **clientId**:  portal-ui
- **url**: https://k8s.oferto.io 

From KeycloakThe Root, Hole, Valid redirect, Valid post logout and Web origin urls are:

**Root url**: https://k8s.oferto.io
**Home url**: https://k8s.oferto.io
**Valid redired Urls**: https://k8s.oferto.io/*
**Valid post logout redirect**: https://k8s.oferto.io/*
**Web origin**: https://k8s.oferto.io

Also define some **roles** and **users** to test angular app.

In that case similar to the other angular sample we must define a new build task in the package with the path used in the ingress similar to the previous sample. Also to check the pod, service and ingress files we can check them in the repo folder of poc-keycloak-k8s/poc-keycloak-angular

## Links to test
These are thw two services deplotes in kubernetes to test:

To access nginx sample
```
https://k8s.oferto.io/web
https://k8s.oferto.io/web/greetings.html
```

To access  Keycloak Admin console
```
https://k8s.oferto.io
```

To access angular without security
```
https://k8s.oferto.io/angular/
```

To access angular with security. User are: admin/password and user/password
```
https://k8s.oferto.io/keycloak-angular/

```

## Some Links

- [Terminación SSL en HAProxy con Let's Encrypt en Ubuntu 20.04](https://es.linkedin.com/pulse/ssl-en-haproxy-con-lets-encrypt-cert-bot-ubuntu-2004-jojoa-yand%C3%BAn)

- [Redirect HTTP to HTTPS in a Few Easy Steps With HAProxy](https://www.haproxy.com/blog/howto-write-apache-proxypass-rules-in-haproxy)

- [How to Write Apache ProxyPass* Rules in HAProxy](https://www.haproxy.com/blog/howto-write-apache-proxypass-rules-in-haproxy)

- [Add a Forwarded header](https://www.haproxy.com/documentation/haproxy-configuration-tutorials/client-ip-preservation/add-forwarded-header)

- [nginx annotations](https://github.com/kubernetes/ingress-nginx/blob/main/docs/user-guide/nginx-configuration/annotations.md)