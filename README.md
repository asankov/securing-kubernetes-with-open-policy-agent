# Securing Kubernetes with Open Policy Agent (OPA)

Build-in Kubernetes security is not enough for most organizations to enforce granular rules and policies to the workloads running in their clusters.

That is why, Kubernetes has pluggable mechanism for deploying additional validation for your resources.
These are the [admission controllers/admission webhooks](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/) which allow you to deploy a webhook which will called by Kubernetes to check whether a given resource can be created/updated.

You can write your own admission webhook from scratch or use [Gatekeeper](https://github.com/open-policy-agent/gatekeeper) which allows you to deploy custom policies, written in [Rego](https://www.openpolicyagent.org/docs/latest/policy-language/) and evaluated by [Open Policy Agent](https://www.openpolicyagent.org/).

## Open Policy Agent

[Open Policy Agent](https://www.openpolicyagent.org/) is a general-purpose policy agent that evaluates JSON input against rules, written in Rego, and returns a JSON output based on the evaluation.

## Rego

Rego is a declarative query language that can be used to write policies about the data coming into the system.

A simple Rego rule that checks if the conference name of the input document is "BSides":

```rego
package bsides

default allow = false

allow = true {
    input.conference.name = "BSides"
}
```

If you want to play with the examples in the Rego playground, use the links in the `Link` column.

| Input                                      | Output             | Playground link                                       |
| ------------------------------------------ | ------------------ | ----------------------------------------------------- |
| `{"conference":{"name": "BSides"}}`        | `{"allow": true}`  | [Link](https://play.openpolicyagent.org/p/OqvsvG5BU7) |
| `{"conference":{"name": "SomethingElse"}}` | `{"allow": false}` | [Link](https://play.openpolicyagent.org/p/SzGf67ckQg) |

A bit more complex rule that checks if the conference name is "BSides" and conference venue is "UNWE" and if not outputs a message:

```rego
package bsides

violations[{"msg": msg}] {
    input.conference.name != "BSides"
    input.conference.venue != "UNWE"
    msg = sprintf("name and venue are wrong - [%s, %s]", [input.conference.name, input.conference.venue])
}
```

However, the message is only shown if both checks are true, e.g. if will not output anything if only one of the values is wrong:

| Input                                                                | Output                                                                                   | Correct | Playground link                                       |
| -------------------------------------------------------------------- | ---------------------------------------------------------------------------------------- | ------- | ----------------------------------------------------- |
| `{"conference":{"name": "BSides", "venue": "UNWE"}}`                 | `{"violations": []}`                                                                     | ✅      | [Link](https://play.openpolicyagent.org/p/wY3h6EN6Dj) |
| `{"conference":{"name": "SomethingElse", "venue": "SomethingElse"}}` | `{"violations": [{"msg": "name and venue are wrong - [SomethingElse, SomethingElse]"}]}` | ✅      | [Link](https://play.openpolicyagent.org/p/JeV5Yg2mlw) |
| `{"conference":{"name": "SomethingElse", "venue": "UNWE"}}`          | `{"violations": []}`                                                                     | ❌      | [Link](https://play.openpolicyagent.org/p/ZPS6QPZZD7) |
| `{"conference":{"name": "BSides", "venue": "SomethingElse"}}`        | `{"violations": []}`                                                                     | ❌      | [Link](https://play.openpolicyagent.org/p/Xo4jlfyFii) |

In this case, this is not what we want, we want to have an error message if at least one of the two values is wrong.

However, Rego rules work by just chaining the expressions in an `AND` statement.

That is why, if we want to achieve this result, we need to tweek the rule a bit:

```rego
package bsides

violations[{"msg": msg}] {
    input.conference.name != "BSides"
    msg := "name is wrong"
}

violations[{"msg": msg}] {
    input.conference.venue != "UNWE"
    msg := "venue is wrong"
}
```

| Input                                                              | Output                                                                  | Correct | Playground link                                       |
| ------------------------------------------------------------------ | ----------------------------------------------------------------------- | ------- | ----------------------------------------------------- |
| `{"conference":{"name": "BSides", "venue": "UNWE"}}`               | `{"violations": []}`                                                    | ✅      | [Link](https://play.openpolicyagent.org/p/WMxPOAnMqR) |
| `{"conference":{"name": "SomethingElse", venue: "SomethingElse"}}` | `{"violations": [{"msg": "name is wrong"}, {"msg": "venue is wrong"}]}` | ✅      | [Link](https://play.openpolicyagent.org/p/bOYYXRjkhY) |
| `{"conference":{"name": "SomethingElse", venue: "UNWE"}}`          | `{"violations": [{"msg": "name is wrong"}]}`                            | ✅      | [Link](https://play.openpolicyagent.org/p/iVHjvE6C4c) |
| `{"conference":{"name": "BSides", venue: "SomethingElse"}}`        | `{"violations": [{"msg": "venue is wrong"}]}`                           | ✅      | [Link](https://play.openpolicyagent.org/p/UB8NwmJ5e4) |

This covers rules all possibilities and we have at least one error message if some of the values is wrong.

**NOTE:** This is not a bug or a deficiency in Rego, this is just how Rego rules work and something we should be aware of when using the language.

## Gatekeeper

As said, Open Policy Agent is general-purpose policy engine that has nothing to do with Kubernetes.

In order to use it with Kubernetes, we need an adapter that will be the bridge between OPA and Kubernetes.

That adapter is called [Gatekeeper](https://github.com/open-policy-agent/gatekeeper).
It serves as the bridge between Kubernetes and OPA.
It implements an Admission controller, allowing it to be called by Kubernetes in order to determine whether a resource can be created/updated.
Also, it provided the object data to the Rego policy so that when we are writing our policies we can expect to get the whole object data provided by Gatekeeper.

It also allows to store the policies as Kubernetes objects (CRDs).
That way, our policies and rules become first-class citizens of our Kubernetes cluster.

### ConstraintTemplates

`ConstraintTemplates` are Kubernetes objects - Custom Resource Definitions (CRDs) that are installed with Gatekeeper.

They wrap a Rego policy and define input parameters for the policy.
This allows for a `ContstraintTemplate` to be reused with different parameters.

A sample `ConstraintTemplate` looks like this:

```yaml
apiVersion: templates.gatekeeper.sh/v1
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
          type: object
          properties:
            labels:
              type: array
              items:
                type: string
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
```

This `ConstraintTemplate` wraps a Rego rules that checks whether a given Kubernetes resource has a set of required labels.

The exact set is contained in the `input.parameters.labels` field.

We will see where this comes from soon.

### Constraint

After you apply the `ConstraintTemplate` from above, Gatekeeper will register a new CRD into your cluster.

The name of that CRD is in the `spec.crd.spec.names.kind` field of the `ConstraintTemplate`.
In our case it's `K8sRequiredLabels`.

In order to start enforcing this policy we will need to create a instance of this CRD, in which we will specify when to invoke the rule and what are the required labels.

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: deployments-must-have-gk
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: [“Deployments"]
  parameters:
    labels: ["gatekeeper"]
```

This resource specifies that this policy must be invoked when `Deployments` are being interacted with.

It also specifies that the required labels are `["gatekeeper"]`.

Now, if we try to create a deployment, Gatekeeper will use OPA to check whether the deployment has the required labels.
If not, Gatekeeper will deny the creation of this resource, and Kubernetes will respect this decision, returning an error to the user.

### Getting our hands dirty

Let's see all of this in practice.

#### Installing Gatekeeper

First, we need to install Gatekeeper into our cluster.

To do that apply this Kubernetes resource:

```sh
kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/release-3.7/deploy/gatekeeper.yaml
```

**NOTE:** This link is taken for the official docs at <https://open-policy-agent.github.io/gatekeeper/website/docs/install/>.
If it gets outdated and it does not work, consult the actual docs.

#### Creating a ConstraintTemplates

Apply the `ConstraintTemplate` YAML:

```sh
kubectl apply -f k8s/constraint-template.yaml
```

or if you haven't cloned the repo:

```sh
kubectl apply -f https://raw.githubusercontent.com/asankov/securing-kubernetes-with-open-policy-agent/main/k8s/constraint-template.yaml
```

This should create the `ConstraintTemplate` and also register the `K8sRequiredLabels` CRD.

Check that by running:

```sh
$ kubectl get constrainttemplates.templates.gatekeeper.sh
NAME                AGE
k8srequiredlabels   119s
```

and

```sh
$ kubectl get crds
NAME                                                    CREATED AT
...
k8srequiredlabels.constraints.gatekeeper.sh             2022-04-09T20:49:22Z
...
```

You should see the `k8srequiredlabels` constraint template and CRD in the output of these commands.

#### Creating a Constraint

To start enforcing this `ConstraintTemplate` create the actual `Constraint`:

```sh
kubectl apply -f k8s/constraint.yaml
```

or if you haven't cloned the repo:

```sh
kubectl apply -f https://raw.githubusercontent.com/asankov/securing-kubernetes-with-open-policy-agent/main/k8s/constraint.yaml
```

This should create the `K8sRequiredLabels` resource.

Check that by running:

```sh
$ kubectl get k8srequiredlabels.constraints.gatekeeper.sh
NAME                       AGE
deployments-must-have-gk   8s
```

You should see the `deployments-must-have-gk` constraint output.

#### Creating a non-compliant resource

The moment of truth!
We have created our `ConstraintTemplate` and our `Constraint` so our security should be in place.
Let's try to create a Deployment that does not comply with this rule and see what happens.

```sh
kubectl apply -f k8s/non-compliant-deployment.yaml
```

or

```sh
kubectl apply -f https://raw.githubusercontent.com/asankov/securing-kubernetes-with-open-policy-agent/main/k8s/non-compliant-deployment.yaml
```

In both cases the result should be:

```sh
$ kubectl apply -f k8s/non-compliant-deployment.yaml
Error from server ([deployments-must-have-gk] you must provide labels: {"gatekeeper"}): error when creating "non-compliant-deployment.yaml": admission webhook "validation.gatekeeper.sh" denied the request: [deployments-must-have-gk] you must provide labels: {"gatekeeper"}
```

Our deployment cannot be created, because it does not comply with our rule.

What happened in more detail was:

- we invoked `kubectl` to try to create the deployment
- `kubectl` parsed the `non-compliant-deployment.yaml` file, serialized it into JSON and send a POST request to the Kube API server
- the Kube API server parsed the request
- it checked whether we have the necessary RBAC permission to perform this operation (we do)
- it called all admission controllers registered for this type of operation (action:CREATE resource:Deployment)
- one of these admission controllers was `gatekeeper-validating-webhook-configuration` (the one coming from Gatekeeper)
- once Kubernetes called Gatekeeper, Gatekeeper checked what constraint it had for this operation (action:CREATE resource:Deployment)
- the only and only constraint was `deployments-must-have-gk` (which has `k8srequiredlabels` for constraint template)
- Gatekeeper called OPA with the policy from the policy from `k8srequiredlabels` constraint template, passing as input the deployment being created and the parameters from the `deployments-must-have-gk` constraint
- OPA evaluated the policy and the input, giving as output `violations` array with one item
- Gatekeeper got the output from OPA, determined that this resource cannot be created and returned a response to the Kube API server that this operation is not allowed
- the Kube API server aborted the operation and returned an error
- `kubectl` outputs the error on the screen

#### Fixing the violation

The output is clear on where we are wrong and what we need to correct in order to create this resource.
Let's do that.

We have the same deployment with the additional labels in [`compliant-deployment.yaml`](k8s/compliant-deployment.yaml).

Let's apply this file:

```sh
kubectl apply -f k8s/compliant-deployment.yaml
```

or

```sh
kubectl apply -f https://raw.githubusercontent.com/asankov/securing-kubernetes-with-open-policy-agent/main/k8s/compliant-deployment.yaml
```

In both cases the result should be:

```sh
$ kubectl apply -f k8s/compliant-deployment.yaml
deployment.apps/non-compliant created
```
