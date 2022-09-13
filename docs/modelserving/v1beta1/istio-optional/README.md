# Deploy model without istio top level virtual service

Istio is a default networking layer in Knative. You can able change various network layers. The following tabs guide you to migrate or set
kourier network layer.

## Delete Istio networking layer

Skip this command if you are not install istio previously.

```bash
kubectl delete --filename https://github.com/knative/net-istio/releases/download/${KNATIVE_VERSION}/release.yaml
```

## Create Kourier networking layer

Install the Knative Kourier controller by running the command: