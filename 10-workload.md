# Deploy the Workload (ASP.NET Core Docker web app)

The cluster now has an [Traefik configured with a TLS certificate](./08-secret-management-and-ingress-controller.md). The last step in the process is to deploy the workload, which will demonstrate the system's functions.

## Steps

> :book: The Contoso app team is about to conclude this journey, but they need an app to test their new infrastructure. For this task they've picked out the venerable [ASP.NET Core Docker sample web app](https://github.com/dotnet/dotnet-docker/tree/master/samples/aspnetapp).

1. Deploy the ASP.NET Core Docker sample web app

   > The workload definition demonstrates the inclusion of a Pod Disruption Budget rule, ingress configuration, and pod (anti-)affinity rules for your reference.

   ```bash
   kubectl apply -k workload/
   ```

1. Wait until is ready to process requests running

   ```bash
   kubectl wait -n a0008 --for=condition=ready pod --selector=app.kubernetes.io/name=aspnetapp --timeout=90s
   ```

1. Check your Ingress resource status as a way to confirm the AKS-managed Internal Load Balancer is functioning

   > In this moment your Ingress Controller (Traefik) is reading your ingress resource object configuration, updating its status, and creating a router to fulfill the new exposed workloads route. Please take a look at this and notice that the address is set with the Internal Load Balancer IP from the configured subnet.

   ```bash
   kubectl get ingress aspnetapp-ingress -n a0008
   ```

   > At this point, the route to the workload is established, SSL offloading configured, a network policy is in place to only allow Traefik to connect to your workload, and Traefik is configured to only accept requests from App Gateway.

### Next step

:arrow_forward: [End-to-End Validation](./11-validation.md)
