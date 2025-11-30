# security-with-gitops

## Committing everything to Git? What about Secrets?

1. Sealed Secret

### Install Sealed Secret Controller

```
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm install sealed-secrets sealed-secrets/sealed-secrets
```

### Install sealed secret cli

```
choco install kubeseal

curl -LO https://github.com/bitnami-labs/sealed-secrets/releases/latest/download/kubeseal-linux-amd64
chmod +x kubeseal-linux-amd64
sudo mv kubeseal-linux-amd64 /usr/local/bin/kubeseal
```

### Generate a key-pair

```
kubectl -n sealed-secrets get secret sealed-secrets-… -o json -o=jsonpath="{.data.tls\.crt}" | base64 -d > sealed-secret.crt
```

### Use the kubeseal CLI to encrypt your secret

```
kubectl create secret generic my-secret --from-literal=password='myStrongPassword' --dry-run=client -o json | kubeseal --cert sealed-secret.crt > mysealedsecret.yaml
```

### Applying Sealed Secrets

```
Commit mysealedsecret.yaml to the Git repository.
```

### Automation with GitOps

Integrate this process into your GitOps workflows. Whenever you update your sealed secrets in Git, your CI/CD pipeline can automatically apply them to your cluster.

2. External Secrets

### Deploy the External Secrets Operator in your Kubernetes cluster

```
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets
```

### ine an ExternalSecret resource that specifies the external secret store and the secret key.

```
apiVersion: external-secrets.io/v1beta1
   kind: ExternalSecret
   metadata:
     name: my-external-secret
   spec:
     secretStoreRef:
       name: my-secret-store
       kind: SecretStore
     target:
       name: my-kubernetes-secret
     data:
     - secretKey: external-secret-key
       remoteRef:
         key: name-of-the-secret-in-external-store
```

### Applying External Secrets

Commit the ExternalSecret resource to your Git repository. The External Secrets Operator will automatically create or update the Kubernetes secret in your cluster based on the external source.

### Integration with GitOps

Incorporate External Secrets into your GitOps pipelines. Changes to the ExternalSecret definitions in your Git repo trigger the operator to sync the secrets, ensuring your cluster’s secrets are always up-to-date.

## Policy Engine for policy-as-code practices (Integrating Kyverno and OPA)

```
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-latest-tag-example
  annotations:
    policies.kyverno.io/title: Disallow Latest Tag Example
    policies.kyverno.io/category: Best Practices
    policies.kyverno.io/minversion: 1.6.0
    policies.kyverno.io/severity: medium
    policies.kyverno.io/subject: Pod
    policies.kyverno.io/description: >-
      The ':latest' tag is mutable and can lead to unexpected errors if the
      image changes.. This policy validates that the image
      specifies a tag and that it is not called `latest`.
spec:
  validationFailureAction: Audit
  background: true
  rules:
    - name: require-image-tag
      match:
        any:
          - resources:
              kinds:
                - Pod
      validate:
        message: "An image tag is required."
        pattern:
          spec:
            containers:
              - image: "*:*"
    - name: validate-image-tag
      match:
        any:
          - resources:
              kinds:
                - Pod
      validate:
        message: "Using a mutable image tag e.g. 'latest' is not allowed."
        pattern:
          spec:
            containers:
              - image: "!*:latest"
```

I refer to this setup as a gatekeeper because it allows you to dictate, through simple rules, what can and cannot be deployed into the cluster.
When combined with the GitOps approach, it opens up new possibilities that transcend team boundaries, enhancing the security of projects.

## Automating security scanning and compliance

### KubeClarity

By integrating KubeClarity within a GitOps framework, an organization can significantly enhance its security and compliance posture, ensuring that its Kubernetes clusters are fortified against evolving threats.

### Falco

Falco operates at the system level (Figure 13.5), monitoring the underlying Linux kernel functionality, or more precisely, the system-level activities of container orchestration platforms. It utilizes Linux kernel capabilities, particularly extended BPF (Berkeley Packet Filter) or traditional system calls (syscalls) monitoring through a kernel module, to observe and analyze system-wide events in real time. Falco can capture and evaluate system calls from applications running inside containers, identifying unusual or undesirable behavior.
