---
title: "A/B testing with VPC Lattice"
sidebar_position: 20
---

In this section we will show how to use Amazon VPC Lattice for advanced traffic management with weighted routing for blue/green and canary-style deployments.

Let's deploy a modified version of the `checkout` microservice with an added prefix *"Lattice"* in the shipping options. Let's deploy this new version in a new namespace (`checkoutv2`) using Kustomize.

```bash
$ kubectl apply -k /workspace/modules/networking/vpc-lattice/abtesting/
```

The checkout namespace now contains two versions of the application:

```bash
$ kubectl get pods -n checkout
NAME                              READY   STATUS    RESTARTS   AGE
TODO
```

# Set up Lattice Service Network

The following YAML will create a Kubernetes gateway resource which is associated with a VPC Lattice **Service Network.

```file
networking/vpc-lattice/controller/eks-workshop-gw.yaml
```

Apply it with the following command:

```bash
$ kubectl apply -f /workspace/modules/networking/vpc-lattice/controller/eks-workshop-gw.yaml
```

Verify that `eks-workshop-gw` gateway is created:

```bash
$ kubectl get gateway  
NAME              CLASS         ADDRESS   READY   AGE
eks-workshop-gw   aws-lattice                     12min
```

Once the gateway is created, find the VPC Lattice service network. Wait until the status is `Reconciled` (this could take about five minutes).

```bash
$ kubectl describe gateway eks-workshop-gw
apiVersion: gateway.networking.k8s.io/v1alpha2
kind: Gateway
status:
   conditions:
      message: 'aws-gateway-arn: arn:aws:vpc-lattice:us-west-2:<YOUR_ACCOUNT>:servicenetwork/sn-03015ffef38fdc005'
      reason: Reconciled
      status: "True"

$ kubectl wait --for=jsonpath='{.status.conditions.reason}'=Reconciled gateway/eks-workshop-gw
```

 Now you can see the associated **Service Network** created in the VPC console under the Lattice resources in the [AWS console](https://console.aws.amazon.com/vpc/home#ServiceNetworks).
![Checkout Service Network](assets/servicenetwork.png)

# Create Routes to targets
Let's demonstrate how weighted routing works by creating  `HTTPRoutes`.

At the time of writing (Apr 2023), the controller requires a port number for `targetPort`, rather than simply specifying `http`. We are working on a better solution, progress is tracked [here](https://github.com/aws/aws-application-networking-k8s/issues/86).

```bash
$ kubectl patch svc checkout -n checkout --patch '{"spec": { "type": "ClusterIP", "ports": [ { "name": "http", "port": 80, "protocol": "TCP", "targetPort": 8080 } ] } }'
```

Create the Kubernetes `HTTPRoute` route that evenly distributes the traffic between `checkout` and `checkoutv2`:

```bash
$ kubectl apply -f /workspace/modules/networking/vpc-lattice/routes/checkout-route.yaml
```

```file
/networking/vpc-lattice/routes/checkout-route.yaml
```

This step may take 2-3 minutes, but once completed you will find the `HTTPRoute`'s DNS name from `HTTPRoute` status (highlighted here on the `message` line):

```bash
$ kubectl describe httproute checkoutroute -n checkout
apiVersion: gateway.networking.k8s.io/v1alpha2
kind: HTTPRoute
...
status:
   parents:
   - conditions:
      - lastTransitionTime: "2023-02-27T14:36:26Z"
         message: 'DNS Name: checkout-checkouroute-05bcd5fb087c79394.7d67968.vpc-lattice-svcs.us-west-2.on.aws'
         reason: Reconciled
      status: "True"
      type: httproute
      controllerName: application-networking.k8s.aws/gateway-api-controller
      parentRef:
         group: gateway.networking.k8s.io
         kind: Gateway
         name: eks-workshop-gw
```

 Now you can see the associated Service created in the [VPC Lattice console](https://console.aws.amazon.com/vpc/home#Services) under the Lattice resources.
![CheckoutRoute Service](assets/checkoutroute.png)

Patch the `configmap` to point to the new endpoint.

```bash
$ export CHECKOUT_ROUTE_DNS='http://'$(kubectl get httproute checkoutroute -n checkout -o json | jq -r '.status.parents[0].conditions[0].message' | cut  -c 11-)
$ kubectl patch configmap/ui -n ui --type merge -p '{"data":{"ENDPOINTS_CHECKOUT": "'${CHECKOUT_ROUTE_DNS}'"}}'
```

:::tip Traffic is now handled by Amazon VPC Lattice
Amazon VPC Lattice can now automatically redirect traffic to this service from any source, including different VPCs! You can also take full advantage of other VPC Lattice [features](https://aws.amazon.com/vpc/lattice/features/).
:::

# Check weighted routing is working

In the real world, canary deployments are regularly used to release a feature to a subset of users. In this scenario, we are artifically routing 50% of users to the new version of the checkout service. Completing the checkout procedure multiple times with different objects in the cart should present the users with the 2 version of the applications. 

Let's ensure that the UI pods are restarted and then port-forward to the preview of your application with Cloud9.

```bash
$ kubectl delete --all po -n ui
$ kubectl port-forward svc/ui 8080:80 -n ui
```

Click on the **Preview** button on the top bar and select **Preview Running Application** to preview the UI application on the right:

![Preview your application](assets/preview-app.png)

Now, try to checkout multiple times: you will notice how the new feature will be available around 50% of the times: this is because Amazon VPC Lattice automatically redirects traffic to different versions of `checkout` microservice. This is because now the UI pod points to the Amazon VPC Lattice endpoint we created earlier whith the `HttpRoute`.

:::danger Don't forget to clean-up

This module is currently in beta, so you must manually run the [Cleanup](cleanup.md) steps before proceeding to the next module.
:::
