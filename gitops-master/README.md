# Optty GitOps

[![pipeline status](https://gitlab.com/deimosdev/client-project/optty/gitops/badges/master/pipeline.svg)](https://gitlab.com/deimosdev/client-project/optty/gitops/-/commits/master)

This project contains scripts and instructions to manage kubernetes resource dependencies for the Optty Project.

### Requirements
- Argocd > 1.7.10 (installed on the kubernetes cluster)
- kustomize
- helm


## Secret Management

Secrets on the cluster are managed using Sealed Secret Controller. 
Visit https://github.com/bitnami-labs/sealed-secrets to install kubeseal, a cli used to interact with the sealedSecret operator.

To encrypt a secret,  you need to be connected to the cluster. 

- First create a kubernetes secret manifest containing the secret for example.
```
apiVersion: v1
data:
  secret: BASE64_ENCODED_SECRET
kind: Secret
metadata:
  name: image-updater-ssh-git-creds
  namespace: argocd
type: Opaque
```
- Encrypt the secret with sealed secret 
```bash
kubeseal < secret.yaml > output.yaml --controller-name sealed-secrets -o yaml
```
- move the output.yaml to the repository where it can then be committed to Argocd

### Decrypt an existing secret
To decrypt an existing sealedSecret you need access to the cluster and the private keys.
- Get the private keys
```
kubectl get secret -n kube-system -l sealedsecrets.bitnami.com/sealed-secrets-key -o yaml >master.key
```
- Decrypt the sealed secret
```bash
kubeseal < sealed-secret.yaml --controller-name sealed-secrets -o yaml --recovery-unseal --recovery-private-key master.key
```

> :warning: NOTE
>
> Do not commit the master key or the raw secret yaml


### Environments
- [staging](./staging/)
- [production](./production)

To apply, `kubectl apply argocd/argocd.yaml` to create the resources for each environment

## Applications

### Namespace
- Creates namespaces for various applications: `nats-io`: for nats, `internal-system` for running resources not related to any service, `apps-staging` for staging applications and `apps` for the dev applications
- Includes annotations for linkerd injection and skip-outbound-ports to prevent linkerd interfering with connections to such applications. Ports to be skipped by linkerd:
    - sql (3306)
    - nats (4222)
    - redis 6379
### [Cert Manager](https://cert-manager.io/)
- Install Cert-Manager from Helm charts
- Create lets-encrypt and self-signed issuer

### [Linkerd](https://github.com/linkerd/linkerd2/)
- Automate the Linkerd control plane install and upgrade lifecycle using Argo CD
- Incorporate Linkerd auto proxy injection feature into the GitOps workflow to auto mesh applications
- Securely store the mTLS trust anchor key/cert with offline encryption and runtime auto-decryption using sealed-secrets
- Use cert-manager to manage the mTLS issuer key/cert resources
- Utilize Argo CD projects to manage bootstrap dependencies and limit access to servers, namespaces and resources
- Uses Argo CD app of apps pattern to declaratively manage a group of application

### [Nats Operator](https://github.com/nats-io/nats-operator)
- Setup Nats Operator to create Nats server
- Automate Nats User Management using `ServiceAccount` and `NatsServiceRole` which creates `bound-token` secrets that can be used as passwords