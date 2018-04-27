# Mutating webhook example

As part of understanding admission controllers for the purposes of replacing `conduit inject`,
I've put together an example mutating webhook
[admission controller](https://kubernetes.io/docs/admin/admission-controllers).

Currently, this example simply injects a `conduit.io=im-injected` annotation into any pods you deploy.

This is a toy example. It is based on the examples in [caesarxuchao/example-webhook-admission-controller](https://github.com/caesarxuchao/example-webhook-admission-controller) and
[Istio's webhook](https://github.com/istio/istio/blob/master/pilot/pkg/kube/inject/webhook.go).
(Here's an simpler example of a controller from [Kelsey](https://github.com/kelseyhightower/denyenv-validating-admission-webhook)).

## What this example webhook does

In `mutating-webhook/main.go` we define a mutating wehbook server that accepts requests from the k8s apiserver.
It processes the request and always accepts the request for admission.
In addition, it modifies any pod specs to include the annotation `conduit.io=im-injected`.

In `selfRegistration` in `config.go`, we specify that we want all pod/deployment creates
to be sent to the webhook for approval.
`config.go` also contains code for setting up the right certs for the HTTPS server.

## Running the webhook in minikube

Make sure minikube is started with the admission controllers available:
```
ADMISSION_CONTROLLERS=NamespaceLifecycle,LimitRanger,ServiceAccount,PersistentVolumeLabel,DefaultStorageClass,DefaultTolerationSeconds,MutatingAdmissionWebhook,ValidatingAdmissionWebhook,ResourceQuota
minikube start --kubernetes-version v1.9.0 --extra-config=apiserver.Admission.PluginNames=$ADMISSION_CONTROLLERS
```

```
cd mutating-webhook

# build the webhook server image
eval $(minikube docker-env)
./build-container

# deploy the webhook
kubectl apply -f k8s/webhook-server.yaml

# ensure the webhook was created properly (wait 10 seconds)
kubectl describe MutatingWebhookConfiguration

# optionally tail the webhook logs
kubectl logs -f $(kubectl get po -o jsonpath='{.items[0].metadata.name}')

# attempt to create a deployment (should work)
kubectl run nginx --image=nginx

# look for our injected annotation
kubectl describe po
kubectl get po $(kubectl get po -o jsonpath='{.items[1].metadata.name}') -o jsonpath='{.metadata.annotations}'
```

## Cleanup
Delete the webhook deployment as well as the webhook config:
```
./cleanup
kubectl delete deployment nginx
```

## Running with conduit
This is intended as a basis for replacing `conduit inject` with an admission controller.
But until installing the webhook is actually packaged into `conduit install`, you'll
need to install conduit *before* deploying the webhook, otherwise conduit's pods will also
pass through the webhook (as is, this is fine since the webhook admits all pods and simply
adds an annotiation, but the goal is that the webhook injects conduit, which we don't want
to do on conduit's pods).

TLDR: If you'd like to run this in order to modify pods before they are deployed in conduit,
be sure to install conduit *first*, before deploying the webhook:
```
# install conduit
conduit install | kubectl apply -f -

# deploy our webhook
kubectl apply -f mutating-webhook/k8s/webhook-server.yaml

# deploy an example application
curl https://raw.githubusercontent.com/runconduit/conduit-examples/master/emojivoto/emojivoto.yml | conduit inject - | kubectl apply -f -

#inspect the example app's pods for our injected annotation
kubectl describe po -n emojivoto
```
