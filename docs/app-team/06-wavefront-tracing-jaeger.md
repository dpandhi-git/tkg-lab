# Wavefront Tracing with Jaeger

For this section, you will perform the following actions in order to get data flowing through Wavefront proxy, which is already installed on this cluster:

- Modify wavefront deployment to accept jaeger traces
- Deploy jaeger-agent as daemonset into acme-fitness namespace
- Modify YAMLs for acme-fitness and redeploy acme
- Generate traffic with application
- View with Wavefront

## Modify Wavefront

Assuming that hte wavefront proxy is as installed per the instructions earlier in the lab, make the following modifications:

```bash
export VMWARE_ID=<your id>
export TO_API_KEY=<wavefront api key>
export WF_APP_NAME=<something to identify your app - ie "AcmeFitness" + your initials>

helm upgrade --install wavefront wavefront/wavefront \
--set wavefront.url=https://demo.wavefront.com \
--set wavefront.token=$TO_API_KEY \  
--set clusterName=$VMWARE_ID-wlc-1 \
--set proxy.jaegerPort=30001 \
--set proxy.args="--traceJaegerApplicationName ${WF_APP_NAME}" \
--namespace wavefront
```

After a moment, the wavefront proxy should be redeployed.  You can check it with:
```bash
kubectl get all -n wavefront
```

## Add Jaaeger Daemonset

By deploying a Jaeger daemonset, the Jaeger agent is running on each node in the cluster.  Applications can point their instrumentation to the IP of the host, which is why we will later modify the acme-fitness deployments to consume the IPs of the host.  For now, install the daemonset in the acme-fitness namespace:

```bash
kubectl apply -f scripts/kubernetes-manifests/jaeger-daemonset.yaml -n acme-fitness
```

This will create a pod per worker node that is sending trace data to the wavefront proxy over in its own namespace.  It uses the DNS name of wavefront-proxy.wavefront as the hostname and the port we picked for jaeger traces (30001).

## Modify ACME Fitness deployments

In order to send data from Jaeger to the jaeger-agent pods, we need to modify 6 of the deployments:
- cart-total.yaml
- catalog-total.yaml
- users-total.yaml
- payment-total.yaml
- frontend-total.yaml
- order-total.yaml

All of these are in acme-fitness-demo/kubernetes-manifests.  Perform the following change:

From:
```
        - name: JAEGER_AGENT_HOST
          value: 'localhost'
```
To:
```bash
        - name: JAEGER_AGENT_HOST
          valueFrom:
            fieldRef:
              fieldPath: status.hostIP
``` 

Redeploy like this:
```bash
kubectl apply -f acme-fitness-demo/kubernetes-manifests/
```
Now hit the application in your browser, with this suggested order:
1) Log in as eric/vmware1!
2) Browse the catalog
3) Put something in the cart
4) Modify the cart (quantity)
5) Checkout (fake data)

Within a few minutes, you should see data flowing into the Wavefront UI that you configured in the Applications -> Application Status area.  The name you selected for WF_APP_NAME should be a top level item.  Select that and look for front-end as the service.  All distributed traces go through that service.

NOTE - cluster and shard do not come through as tags without advanced modification.  That is another lesson.

