# Anchore Kubernetes Admission Controller Guide for Kubernetes v1.15

**Requirements:**
- Anchore Engine running
- Account and user(s) added to your Anchore deployment
- Access to the Anchore Engine api container (default port is 8228) from the cluster where you will be installing the Anchore Kubernetes Admission Controller
- Kubernetes cluster running
- kubectl installed on your local machine
- helm installed on your local machine
- jq installed on your local machine

**If you have a non-working version of the Anchore Kubernetes Admission Controller, perform these 3 steps first:**
1. Locate the old controller's release
```
helm -n <your-namespace> list
```
2. Delete the old controller's release
```
helm -n <your-namespace> delete <release-name>
```
3. Use _cleanup.sh_ (provided in this repo) to remove the Kubernetes objects that were not removed by the helm delete
```
chmod +x cleanup.sh
./cleanup.sh <release-name> <your-namespace>
```

## Steps to install the Anchore Kubernetes Admission Controller:
1. Create _credentials.json_ (provided in this repo) and add the users you'd like to configure the controller for. These users must already exist in your Anchore deployment
2. Create a secret for the Anchore credentials that the controller will use to make api calls to Anchore
```
kubectl -n <your-namespace> create secret generic anchore-credentials --from-file=credentials.json
```
3. Create _values.yaml_ (provided in this repo) and add your anchoreEndpoint with desired policy configuration
4. Install the controller
```
helm -n <your-namespace> install <release-name> --repo https://charts.anchore.io/stable anchore-admission-controller -f <path-to-values.yaml>
```
5. Copy the validating webhook config (provided in this repo) from the output of step 4 and apply it
```
kubectl -n <your-namespace> apply -f validating-webhook.yaml
```

## Testing with Mode: policy
- **Example 1**
```
kubectl -n <your-namespace> run -it debian-latest --restart=Never --image debian:latest /bin/sh
```
_Output if network access to the Anchore Engine api container is not setup:_
```
Error from server (Timeout): Timeout: request did not complete within requested timeout 30s
```
_Output if image has not been analyzed:_
``` 
Error from server: admission webhook "my-anchore-admission-controller.admission.anchore.io" denied the request: Image debian:latest is not analyzed. Cannot evaluate policy
```
_Output if image has been analyzed, but didn't pass policy evaluation:_
``` 
Error from server: admission webhook "my-anchore-admission-controller.admission.anchore.io" denied the request: Image debian:latest with digest sha256:1e8d7127072cdbaae1935656444c3ec2bef8882c8c14d459e3a92ca1dd313c28 failed policy checks for policy bundle 2c53a13c-1765-11e8-82ef-23527761d060
```
- **Example 2**
```
kubectl -n <your-namespace> apply -f pod.yaml 
```
_Output if not analyzed:_
``` 
Error from server: error when creating "pod.yaml": admission webhook "my-anchore-admission-controller.admission.anchore.io" denied the request: Image busybox is not analyzed. Cannot evaluate policy
```
_Output if image has been analyzed and passed policy evaluation:_
``` 
pod/busybox-sleep created
```
```
kubectl -n <your-namespace> get pods
NAME                                               READY   STATUS    RESTARTS   AGE
busybox-sleep                                      1/1     Running   0          33s
my-anchore-admission-controller-597f48bc87-7tfcs   1/1     Running   0          54m
```

### Other Helpful Resources
- [Anchore Admission Controller repo](https://github.com/anchore/anchore-charts/tree/master/stable/anchore-admission-controller)
- [Dynamic Policy Mappings & Modes in the Anchore Kubernetes Admission Controller](https://anchore.com/blog/dynamic-policy-mappings-and-modes-in-the-anchore-kubernetes-admission-controller/)