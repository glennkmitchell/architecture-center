# Gateway

Problems with client directly calling backend services

- May be chatty to call multiple services
- Client has to keep track of multiple end points
- Hard to change the backend endpoints or refactor services
- Each backend service must handle everything necessary to accept client requests, such as authentication, throttling, SSL
- Backend services must expose client-friendly protocols like HTTP or WebSocket. 


Reasons to use a gateway

- Terminate SSL
- Route requests to services / aggregrate results
- Reduce chattiness (aggregation)
- User / client authentication
- IP whitelisting
- Client throttling (rate limits)
- Log client requests
- Protocol translation
- Load balancing. In this case, the Azure Load Balancer already 
- Logging, monitoring


Options
- Run nginx, haproxy, or other solution in a container. 
    - Easy to configure and deploy as part of your cluster
    - Run in pods for redundancy
    - Rich feature set
- Use a service mesh (Istio) - linkerd?
    - Layer-7 routing
    - Advantage: Integrated into service mesh, nothing else to deploy (if you're using service mesh already)
- Azure App Gateway - Note ACS does not support App Gateway 
    - Layer-7 routing, SSL termination, WAF
    - Easy to provision via RM templates
    - Set instance count > 1 for HA
    - Logging and monitoring
- API Management
    - Deploy to a cluster VNET
    - Expose  k8s service as NodePort
    - API routing
    - Throttling, whitelisting
    - Monitoring
    - AAD access tokens or API key
    - Mutual authentication between the gateway and the back end services


Considerations:
- Availability of the gateway - failure point
- Process for updating gateway for new services or API changes
- Scalability. Avoid gateway becoming performance bottleneck.


## Nginx, HA Proxy, or other reverse proxies

Run nginx or HAProxy in a container, use config map for configuration file, and mount the configmap as a volume. (Example for nginx mount at `/etc/nginx/nginx.conf`)

Another option is to use an ingress controller. This lets you deploy a reverse proxy (nginx, HAProxy) and specify TLS keys and certs, configure layer-7 routing, name-based virtual hosting (e.g., deliveries.controso.com goes to one backend service, account.contoso.com goes to another backend service)

The way it works is

- Ingress resource defines the configuration (routing rules, etc)
- Ingress controller is the resource that performs the reverse proxying. Several implementations exists, including nginx and HAProxy. It watches the Ingress resource and updates the reverse proxy as needed.

Ingress = rules

Deployment with container iamge set to ingress controller
 
Service (type = Load Balancer) to expose the ingress controller

Backend services

This is a cleaner approach as it abstracts the ingress rules from the implementation details of the reverse proxy. However this is beta at the time of this writing, and will continue to evolve.

## Connect API Management to Kubernetes

1. Create an empty subnet in the cluster VNET
2. Deploy API Management to that subnet.
3. Create a Kubernetes service of type LoadBalancer. Use the [internal load balancer](https://kubernetes.io/docs/concepts/services-networking/service/#internal-load-balancer) annotation (see example below)
4. Find the private IP of the internal load balancer, using kubectl or the Azure Portal, CLI, etc.
5. In API Management, create an API that directs to the private IP address of the load balancer.


## Connect Application Gateway to Kubernetes

1. Create an empty subnet in the cluster VNET.
2. Deploy Application Gateway.
3. Create Kubernetes service with type=[NodePort](https://kubernetes.io/docs/concepts/services-networking/service/#type-nodeport). This exposes the service on each node so that it can be reached frmo outside the cluster. It does not create a load balancer. 
5. Get the assigned port number for the service.
6. Add a Application Gateway rule where:
    - The backend pool contains the agent VMs.
    - The HTTP setting specifies the service port number.
    - The gateway listener listens on port 80
    
App GW will route from 80 to the service port number.

