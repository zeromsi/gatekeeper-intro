Let's learn about opa and gatekeeper by asking some questions!

## What is opa (Open Policy Agent)?
Opa is a general purpose policy engine, a tool to enforce policies across stacks (from infrastructure to application serve). It supports [Rego](https://www.openpolicyagent.org/docs/latest/policy-language/); a policy language to write policies.


## What is Gatekeeper?
Gatekeeper is one implementation of OPA; a tool to enforce policies to govern a k8s cluster declaratively. It's a CNCF graduated project.

## Why we need a policy engine in our k8s cluster; We have authentication and authorization features?

Yes, kubernetes let's us authenticate and authorize user while creating objects using RBAC. But to govern a cluster, we need more power.
For example, we may need to ensure that every k8s object must have certain levels or annotations or workloads must have certain number of replicas. These can be done by adopting policies, here comes OPA and gatekeeper.

## How k8s let's us adopt a policy engine?

In order to understand this, we need to understand how k8s persists objects.

When we request to create/update/delete an object, request goes to the ```api-server```, ```api-server``` authenticates and authorizes the request, calls some interceptors(webhooks) named ```admission controller webhooks``` to do object ```validation``` or ```mutation``` or both, if everyting is okay, it persists the object into ```ETCD```.

A ```mutating admission controller``` modifies object before persisting. For example, it may add certain annotations or labels with the object. 


A ```validating admission controller``` validates object against certain rules, like object must have certain labels or annotations etc. 

K8s let's us write and adopt webhook in our cluster according to our need. Gatekeeper is basically a validating webhook that let's us apply k8s style policies. 


## What are the key components of gatekeeper from a users perspective?

- ```Constraint Templates```: Constraint template is a custom resource definition (CRD) or type that holds the policy logic and regarding properties of arbitrary objects that can be passed by a ```Constraint```.

    Example: 

    ```yaml
    apiVersion: templates.gatekeeper.sh/v1beta1
    kind: ConstraintTemplate
    metadata:
    name: k8srequiredlabels
    spec:
    crd:
        spec:
        names:
            kind: K8sRequiredLabels
            listKind: K8sRequiredLabelsList
            plural: k8srequiredlabels
            singular: k8srequiredlabels
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

            deny[{"msg": msg, "details": {"missing_labels": missing}}] {
            provided := {label | input.review.object.metadata.labels[label]}
            required := {label | label := input.parameters.labels[_]}
            missing := required - provided
            count(missing) > 0
            msg := sprintf("you must provide labels: %v", [missing])
            } 

    ```

    Here, ```spec.crd.spec.validation.openAPIV3Schema.properties``` property holds the argument schema that can be passed from a ```Constraint```.

    Then, ```spec.targets[_].rego``` is the portion that holds the logic of a policy. 

    In this example, we are validating arbitrary objects against certain labels. 

    By appliyng this, we will create a ```Constraint Template``` type ```K8sRequiredLabels``` in our cluster.

- ```Constraint``` is an object (CR) of ```Constraint template``` that applies policies for specific k8s objects based on the argument passed. 

    Example: 

    ```yaml
    apiVersion: constraints.gatekeeper.sh/v1beta1
    kind: K8sRequiredLabels
    metadata:
    name: ns-must-have-database
    spec:
    match:
        kinds:
        - apiGroups: [""]
            kinds: ["Namespace"]
    parameters:
        labels: ["database"]

    ```

    In this example, we are creating a ``` Custom Resource``` or object of ```K8sRequiredLabels``` ```Constraint Template``` where we have defined the attributes ```labels``` for all namespaces. 

    By applying this, we have initiated a policy for namespace to require a label ```database```.




## How to Install?

Please follow [this](https://open-policy-agent.github.io/gatekeeper/website/docs/install/) guide.


## References
https://kubernetes.io/blog/2019/08/06/opa-gatekeeper-policy-and-governance-for-kubernetes/
https://github.com/open-policy-agent/gatekeeper




