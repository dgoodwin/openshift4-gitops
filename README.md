# openshift4-gitops
Testing OpenShift 4 Cluster Configuration With Gitops

## Installing ArgoCD on OpenShift 4

These manual steps will hopefully be replaced by an ArgoCD operator on OperatorHub in the near future.

```bash
oc new-project argocd
oc apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
oc create route passthrough --service=argocd-server

# Patch ArgoCD Server so no TLS is configured on the server (--insecure)
PATCH='{"spec":{"template":{"spec":{"$setElementOrder/containers":[{"name":"argocd-server"}],"containers":[{"command":["argocd-server","--insecure","--staticassets","/shared/app"],"name":"argocd-server"}]}}}}'
oc -n argocd patch deployment argocd-server -p $PATCH
# Expose the ArgoCD Server using an Edge OpenShift Route so TLS is used for incoming connections
oc -n argocd create route edge argocd-server --service=argocd-server --port=http --insecure-policy=Redirect


# Get the argoCD 'admin' password:
ARGO_ADMIN_PASS=`kubectl get pods -n argocd -l app.kubernetes.io/name=argocd-server -o name | cut -d'/' -f 2`

# Login:
argocd login argocd-server-argocd.apps.dgoodwin-dev.new-installer.openshift.com:443 --username admin --password $ARGO_ADMIN_PASS --insecure

# Change the ArgoCD password:
argocd account update-password
```

# Configuring OpenShift 4

## General Guidelines

 1. ArgoCD "Applications" (despite the name) can be used to deliver global custom resources such as those which configure OpenShift v4 clusters.
 1. When creating an application you will be required to provide a namespace. In the case of an application delivering global custom resources this doesn't make a lot of sense, but you can provide the name of any namespace to get past this issue.
 1. By default Argo will look to prune resources, should you ever delete your application that delivered them. In the case of OpenShift v4 global configuration custom resources, these often are blocked from being deleted, which can cause Argo to become stuck. If however in your configuration git repository you add the `argocd.argoproj.io/sync-options: Prune=false` annotation to your custom resources, this problem can be avoided. If you do run into this problem, you will need to manually "kubectl edit" the Argo Application and remove the finalizer which blocks until resources are pruned.

## Examples

The following section demonstrates the use of ArgoCD to delivery some of the available [OpenShift v4 Cluster Customizations](https://docs.openshift.com/container-platform/4.1/installing/install_config/customizations.html).

### Identity Provider

The [identity-providers](./identity-providers) directory contains an example for deploying a HTPasswd OAuth provider, and the associated secret. Deploying this as an ArgoCD application should allow you to login to your cluster as *user1 / MyPassword!*. For information on how this secret was created, see the [OpenShift 4 Documentation](https://docs.openshift.com/container-platform/4.1/authentication/identity_providers/configuring-htpasswd-identity-provider.html#configuring-htpasswd-identity-provider).

```bash
argocd app create htpasswd-oauth --repo https://github.com/dgoodwin/openshift4-gitops.git --path=identity-providers --dest-server=https://kubernetes.default.svc --dest-namespace=openshift-config
argocd app sync htpasswd-oauth
```

This example includes both a global OAuth config resource, and a namespaced secret.

WARNING: The openshift-oauth operator copies your specified secrets to the openshift-authentication, including their labels. One of these labels in added by ArgoCD to indicate the secret is owned by the htpasswd-oauth application. When this is copied, it causes ArgoCD to now see the copied secret as a resource it doesn't know about, is owned by this app, thus should be pruned. You can disable pruning with the normal annotation but will still see this secret as out of sync in the UI.

### Builds

The [builds](./builds) directory contains an example global Build configuration.

```bash
argocd app create builds-config --repo https://github.com/dgoodwin/openshift4-gitops.git --path=builds --dest-server=https://kubernetes.default.svc --dest-namespace=openshift-config
argocd app sync builds-config
```

### Console

The [console](./console) directory contains a simple configuration for the OpenShift console which simply changes the logout behavior to redirect to Google.

```bash
argocd app create console-config --repo https://github.com/dgoodwin/openshift4-gitops.git --path=console --dest-server=https://kubernetes.default.svc --dest-namespace=openshift-config
argocd app sync console-config
```

TODO: The --dest-namespace here is odd as this example contains only a global resource.

### Machine Sets

The machine-sets sub-dir contains an example MachineSet being deployed as an application via ArgoCD:

```bash
argocd app create machineset --repo https://github.com/dgoodwin/openshift4-gitops.git --path=machine-sets --dest-server=https://kubernetes.default.svc --dest-namespace=openshift-machine-api
argocd app sync machineset
```

However there is a problem here, if you [view the
yaml](./machine-sets/machinesets.yaml) you will see the cluster's generated
InfraID referenced multiple times. This value is generated by the OpenShift installer and used in the naming of many cloud objects. Committing cluster config will be problematic as this value is not known before install, and not consistent across clusters.

A standard OpenShift 4 cluster with 3 compute nodes in us-east-1 comes with 6 MachineSets, one per AZ (in my account), with only three of them scaled to 1 replicas. Each MachineSet references the generated InfraID roughly 9 times:

 - MachineSet Name
 - Selector
 - IAM Instance Profile
 - Security Group Name
 - Subnet
 - AWS Tags

TODO: Should we recommend against using MachineSets with gitops and Argo? Or is there a templating solution we should explore? In this case the value we want to template is a fact about the individual cluster it's being deployed to.

