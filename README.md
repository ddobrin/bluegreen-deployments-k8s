# Demo for Blue-Green deployments in Kubernetes

The demo applications in this repo are leveraging Spring Cloud Kubernetes and *DO NOT* use a service discovery mechanism (Consul, Eureka, etc)

## The Kubernetes model for connecting containers
Once you have a continuously running, replicated application you can expose it on a network. Before discussing the Kubernetes approach to networking, it is worthwhile to contrast it with the “normal” way networking works with Docker.

By default, Docker uses host-private networking, so containers can talk to other containers only if they are on the same machine. In order for Docker containers to communicate across nodes, there must be allocated ports on the machine’s own IP address, which are then forwarded or proxied to the containers. This obviously means that containers must either coordinate which ports they use very carefully or ports must be allocated dynamically.

Coordinating port allocations across multiple developers or teams that provide containers is very difficult to do at scale, and exposes users to cluster-level issues outside of their control. Kubernetes assumes that pods can communicate with other pods, regardless of which host they land on. Kubernetes gives every pod its own cluster-private IP address, so you do not need to explicitly create links between pods or map container ports to host ports. This means that containers within a Pod can all reach each other’s ports on localhost, and all pods in a cluster can see each other without NAT. The rest of this document elaborates on how you can run reliable services on such a networking model.

## Why Blue-Green deployments
Blue-green deployment is a technique that reduces downtime and risk by running two identical production environments, called for example Blue and Green.

At this time, only one of the environments is live, with the live environment serving all production traffic. For the example in this repo, Blue is considered the live versio of the service and Green the idle one.

As you prepare a new version of your service, deployment and the final stage of testing takes place in the environment which is not live: in this example, Green. 
Smoke tests can be run on the new version to verify its functionality, then have traffic move from the old version to the new, with zero downtime.

The Green environment serves for business acceptance testing of the new release of the service.

Please note that once you have deployed and fully tested the service in Green, you can start the switch from the (old) Blue version top the (new) Green version. 
This technique eliminates downtime due to app deployment. 

Finally, blue-green deployment reduces risk: if something unexpected happens with your new version on Green, you can immediately roll back to the last version by switching back to Blue.

## Blue Green deployment process

The process is broken down into four distinct processes
1. The Blue service version is deployed and exposed using a Kubernetes service, which matches the selector of the Blue service version
2. The Green service version is deployed but NOT exposed using a service
3. The service is being updated to match the selector of the Green service version. Clients are thus redirected to the pods of the Green service version, and becomes the active deployment.
4. The Blue service version can be decommissioned once business acceptance is complete. In case of an error, step 3 is being reverted to the Blue version on short notice with zero-downtime

**The Blue service version deployment:**

![Blue Version Deployment](https://github.com/ddobrin/bluegreen-deployments-k8s/blob/master/images/BG-K8s1.png)  

The deployment of the blue service version can be created using the command:
```shell
> kubectl apply -f deployment.yaml
```

Once we have a deployment created we can provide a way to access the instances of the deployment by creating a Service. Services are decoupled from deployments so that means that you don't explicitly point a service at a deployment. What you do instead is specify a label selector which is used to list the pods that make up the service. When using deployments, this is typically set up so that it matches the pods for a deployment.

The service can be created using the command:
```shell
> kubectl apply –f service.yaml
```

The selector of the service matches the Blue service version:
```yaml
  selector:
    app: message-service
    version: blue
```

**The Green service version deployment:**

![Green Version Deployment](https://github.com/ddobrin/bluegreen-deployments-k8s/blob/master/images/BG-K8s2.png)  

The Green service version will be deployed in parallel with the Blue service version.
```shell
> kubectl apply -f deployment-green.yaml
```

**The Blue - Green switch:**

![Blue-Green Switch](https://github.com/ddobrin/bluegreen-deployments-k8s/blob/master/images/BG-K8s3.png)

To "switch over" to the Green service version, we are updating the selector of the Service and applying it in Kubernetes:
```yaml
  selector:
    app: message-service
    version: green
```
The command to update the service with zero-downtime for the consumer is the same:
```shell
> kubectl apply –f service.yaml
```