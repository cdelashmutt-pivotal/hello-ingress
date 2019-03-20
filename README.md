# Exposing Apps Outside of Kubernetes
A small deployment to showcase a few options for exposing apps to the outside world.

## Usage
Deploy the app itself with the following:

```shell
$ kubectl apply -f deployment.yml
```

---
**Note**

The deployment references a container image stored in Google's Container Registry on the public internet.  If your cluster can't reach the public internet, the pods won't be able to start due to being unable to pull the referenced image.  You'll need to load that image to a repository that your cluster can reach, and then modify the deployment.yml file to reference the image from your private repository.

---

If you want to test the app, you can port-forward the app's listening port to your local machine to quickly test.

```shell
$ kubectl port-forward deployment/hello-node 8080:8080
```

You can then make a request to http://localhost:8080 to see the result "Hello world!".  After you are finished testing, you can press CTRL-C in the terminal that you execute the above `kubectl port-forward` command to end port forwarding.

After you have deployed and tested the app, you can choose to expose the app via a standard loadbalancer, via HTTP Ingress, or via HTTP(S) Ingress.

### LoadBalancer
You can deploy a LoadBalancer service with the following:

```shell
$ kubectl apply -f loadbalancer-service.yml
```

You can get the External IP address of your LoadBalancer with the following command (your IP address will be different from below):

```shell
$ kubectl get service hello-node-service
NAME                 TYPE           CLUSTER-IP       EXTERNAL-IP        PORT(S)        AGE
hello-node-service   LoadBalancer   50.50.50.50      1.2.3.4,...        80:31840/TCP   36s
```

You can then make a request to your app at http://<your-ip> to get a response of "Hello world!".

### HTTP Only Ingress
Instead of a specific LoadBalancer for your app, you can save resources by using a platform provided Ingress controller to route requests to your application by using this following:

```shell
$ kubectl apply -f .\ingress.yml
```

After the service is updated (we don't need a LoadBalancer, so that manifest turns the service into a "ClusterIP" service), you can then look at the Ingress details to see how to make requests to your app:

```shell
$ kubectl describe ingress hello-node-ingress
Name:             hello-node-ingress
Namespace:        cdelashmutt
Address:          10.35.120.38,100.64.64.5
Default backend:  default-http-backend:80 (<none>)
Rules:
  Host           Path  Backends
  ----           ----  --------
  foobar.domain
                    hello-node-service:8080 (<none>)
Annotations:
  kubectl.kubernetes.io/last-applied-configuration:  {"apiVersion":"extensions/v1beta1","kind":"Ingress","metadata":{"annotations":{},"name":"hello-node-ingress","namespace":"cdelashmutt"},"spec":{"rules":[{"host":"foobar.domain","http":{"paths":[{"backend":{"serviceName":"hello-node-service","servicePort":8080}}]}}]}}

  ncp/internal_ip_for_policy:  100.64.64.5
Events:                        <none>
```

If you look at this Ingress definition, we can see that the external IP for the Ingress server is 10.35.120.38.  Below the address we can see all the rules defined for this Ingress definition.  In this example, the rules say that any requests to `http://foobar.domain` for all paths will be sent to the pods behind the `hello-node-service`.

---
**Note**
It is important to understand that Kubernetes doesn't necessarily automatically register the domain name that you specified for your Ingress.  You need to have an A record created in DNS for that host name that points to the address of the Ingress server (10.35.120.38 in this example).

Alternatively edit your `hosts` file (/etc/hosts on *nix systems, c:\windows\system32\drivers\etc\hosts on Windows) and add your own entry.  An additional option would be to deploy the Ingress as is, find the IP for the Ingress server, and then edit the Ingress to use a hostname from a service like nip.io to get to your service.  In our example, we could edit the Ingress rule to use the host name `hello.10.35.120.38.nip.io` to prevent having to register the name in DNS.

---

### HTTPS Ingress

To enable HTTPS for your Ingress, you will need to provide a TLS certificate and private key for use with your server.  You can use the following to create a secret with a key and cert, and assign it to your Ingress:

```shell
$ kubectl apply -f .\ingress-tls.yml
```

Now if you try to make a request to https://foobar.domain, you should get an encrypted connection to your app.