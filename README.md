# Notes for Edge Service

#### BOOTING UP THIS PROJECT
First, we need both Catalog Service and Order Service 
up and running. From each project’s root folder, 
run `./gradlew bootBuildImage` to package them as container images. 
Then start them via Docker Compose. 
Open a Terminal window, navigate to the folder where your 
`docker-compose.yml` file is located `(polar-deployment/docker)`, 
and run the following command:
```bash
docker-compose up -d catalog-service order-service
```

Start the redis container:
```bash
docker-compose up -d polar-redis
```

Since both applications depend on PostgreSQL, Docker Compose 
will also run the PostgreSQL container.
When the downstream services are all up and running, 
it’s time to start Edge Service. 
From a Terminal window, navigate to the project’s root folder 
`(edge-service)`, and run the following command:
```bash
./gradlew bootRun
```
Terminate all the containers by giving the path to 
`docker-compose.yml` file:
```bash
docker-compose -f <path_to_docker-compose.yml> down
```

<br>

---

## Circuit breakers for Spring with resilience4j
There is **closed**, **open** and **half-open** states when talking
about circuit breakers.
**Closed** state allows normal operation, and requests are allowed
to pass through.
In the **open** state, the circuit breaker stops requests from being 
passed through, providing a kind of "short-circuit" behavior.
The **half-open** state is an intermediary state between open and closed.
After a certain time period in the open state, the circuit breaker 
transitions to the **half-open** state. 
In this state, a limited number of requests are allowed to 
pass through.

When you combine multiple resilience patterns, 
the sequence in which they are applied is fundamental. 
Spring Cloud Gateway takes care of applying the TimeLimiter 
first (or the timeout on the HTTP client), 
then the CircuitBreaker filter, and finally Retry. 
Figure 9.5 shows how these patterns work together to increase 
the application’s resilience.
![](img/applicationsResilience.png)
The result can be verified by using the tool `Apache Benchmark`.


## Request rate limiter
The implementation of RequestRateLimiter on Redis is based 
on the token bucket algorithm. 
Each user is assigned a bucket inside which tokens are dripped 
over time at a specific rate (the replenish rate). 
Each bucket has a maximum capacity (the burst capacity). 
When a user makes a request, a token is removed from its bucket. 
When there are no more tokens left, the request is not permitted, 
and the user will have to wait until more tokens have dripped 
into its bucket.
To know more about the token bucket algorithm, 
recommend reading Paul Tarjan’s “Scaling your API with 
Rate Limiters” article about how they use it to implement rate 
limiters at Stripe 
[rate-limiters](https://stripe.com/blog/rate-limiters).

## Redis
What happens if Redis becomes unavailable? 
Spring Cloud Gateway has been built with resilience in mind, 
so it will keep its service level, but the rate limiters 
would be disabled until Redis is up and running again.

## Ingress
An Ingress is an object that “manages external access to the 
services in a cluster, typically HTTP. Ingress may provide 
load balancing, SSL termination and name-based 
virtual hosting” (https://kubernetes.io/docs). 
An Ingress object acts as an entry point into a 
Kubernetes cluster and is capable of routing traffic 
from a single external IP address to multiple 
services running inside the cluster. 
We can use an Ingress object to perform load balancing, 
accept external traffic directed to a specific URL, 
nd manage the TLS termination to expose the 
application services via HTTPS.
Ingress objects don’t accomplish anything by themselves. 
We use an Ingress object to declare the desired state in 
terms of routing and TLS termination. 
The actual component that enforces those rules and routes 
traffic from outside the cluster to the applications 
inside is the ingress controller. 
Since multiple implementations are available, 
there’s no default ingress controller included in the 
core Kubernetes distribution— it’s up to you to install one. 
Ingress controllers are applications that are usually 
uilt using reverse proxies like NGINX, HAProxy, or Envoy. 
Some examples are Ambassador Emissary, Contour, and Ingress NGINX.
In production, the cloud platform or dedicated tools would be used
to configure an ingress controller. 
In our local environment, 
we’ll need some additional configuration to make the routing work. 
For the Polar Bookshop example, we’ll use Ingress 
NGINX (https://github.com/kubernetes/ingress-nginx) in 
both environments.

Start the polar minikube from scratch or execute it if already
create profile polar:
```bash
minikube start --cpus --memory 4g --driver docker --pofile polar
minikube start --profile polar
```

Enable the ingress add-on:
```bash
minikube addons enable ingress --profile polar
```

Get information about the different components deployed with
ingress NGINX:
```bash
kubectl get all -n ingress nginx
```
The flag -n fetches all objects created in the nginx namespace.

Retrieve the assigned IP-address for the minikube cluster
run:
```bash
minikube ip --profile polar
```
The ingress add-on doesn't yet support the use of cluster's IP
addresses for mac or windows when running on Docker.
Therefor use the minikube tunnel to expose cluster to local
environment:
```bash
minikube tunnnel --profile polar
```
That is similar to `kubectl port-forward` but applies to whole
cluster not just specific service.

