# Demo

## OPA Gatekeeper

Deploy: 

```bash
$ kind create cluster
$ kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/release-3.2/deploy/gatekeeper.yaml
```


Example: Labels required in all namespaces:

Template:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: templates.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
  name: k8srequiredlabels
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredLabels
      validation:
        # Schema for the `parameters` field
        openAPIV3Schema:
          properties:
            labels:
              type: array
              items: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredlabels

        violation[{"msg": msg, "details": {"missing_labels": missing}}] {
          provided := {label | input.review.object.metadata.labels[label]}
          required := {label | label := input.parameters.labels[_]}
          missing := required - provided
          count(missing) > 0
          msg := sprintf("you must provide labels: %v", [missing])
        }
EOF
```

Constraint:


```bash
cat <<EOF | kubectl apply -f -
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: ns-must-have-gk
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Namespace"]
  parameters:
    labels: ["gatekeeper"]
EOF
```

```bash
$ kubectl create ns demo
Error from server ([denied by ns-must-have-gk] you must provide labels: {"gatekeeper"}): admission webhook "validation.gatekeeper.sh" denied the request: [denied by ns-must-have-gk] you must provide labels: {"gatekeeper"}
````

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: demo
  labels:
    gatekeeper: ok
EOF
```
