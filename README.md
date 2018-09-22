# kube-plex

kube-plex is a scalable Plex Media Server solution for Kubernetes. It
distributes transcode jobs by creating pods in a Kubernetes cluster to perform
transcodes, instead of running transcodes on the Plex Media Server instance
itself.

## How it works

kube-plex works by replacing the Plex Transcoder program on the main PMS
instance with our own little shim. This shim intercepts calls to Plex
Transcoder, and creates Kubernetes pods to perform the work instead. These
pods use shared persistent volumes to store the results of the transcode (and
read your media!).

## Prerequisites

* A persistent volume type that supports ReadWriteMany volumes (e.g. NFS,
Amazon EFS)
* Your Plex Media Server *must* be configured to allow connections from
unauthorized users for your pod network, else the transcode job is unable to
report information back to Plex about the state of the transcode job. At some
point in the future this may change, but it is a required step in order to make
transcodes work right now.

## Prerequisites Setup

1) Define Storage Class

```bash
    kubectl apply -f StorageClasses/vsan-thin-disk-sc.yaml
```

2) Define Persistent Volume Claims (as per declaration in [charts/kube-plex/values.yaml](https://github.com/DooDleWilk/kube-plex/blob/master/charts/kube-plex/values.yaml))

```bash
    kubectl apply \
      --namespace plex \
      -f PersistentVolumeClaims/plex-config-pvc.yaml \
      -f PersistentVolumeClaims/plex-data-pvc.yaml \
      -f PersistentVolumeClaims/plex-transcode-pvc.yaml \
      -f PersistentVolumeClaims/plex-master-pvc.yaml
```


## Setup

This guide will go through setting up a Plex Media Server instance on a
Kubernetes cluster, configured to launch transcode jobs on the same cluster
in pods created in the same 'plex' namespace.

1) Obtain a Plex Claim Token by visiting [plex.tv/claim](https://plex.tv/claim).
This will be used to bind your new PMS instance to your own user account
automatically.

2) Deploy the Helm chart included in this repository using the claim token
obtained in step 1.

```bash
    helm install ./charts/kube-plex --name plex \
      --namespace plex \
      --set claimToken=[insert claim token here] \
      --set ingress.enabled=true
```

This will deploy a scalable Plex Media Server instance that uses Kubernetes as
a backend for executing transcode jobs.

3) Access the Plex dashboard, either using `kubectl port-forward`, or using
the services LoadBalancer IP (via `kubectl get service`), or alternatively use
the ingress provisioned in the previous step (with `--set ingress.enabled=true`).

4) Visit Settings->Server->Network and add your pod network subnet to the
`List of IP addresses and networks that are allowed without auth` (near the
bottom). For example, `10.100.0.0/16` is the subnet that pods in my cluster are
assigned IPs from, so I enter `10.100.0.0/16` in the box.

You should now be able to play media from your PMS instance - pods will be
created to handle transcodes, and data automatically mounted in appropriately:

```bash
➜  kubectl get po -n plex
NAME                              READY     STATUS    RESTARTS   AGE
plex-kube-plex-75b96cdcb4-skrxr   1/1       Running   0          14m
pms-elastic-transcoder-7wnqk      1/1       Running   0          8m
```
