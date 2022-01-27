# Tutorial: Iterative Rules with K8s Config Authorization and Styra DAS

[Kubernetes](https://kubernetes.io/) is a popular, open-source system for automating deployment, scaling, and management of containerized applications.

## Goals

This tutorial shows how Kubernetes's admission controllers can be used with Styra DAS to enforce policies that dictate which resources can be deployed to the cluster and how those resources should be configured.  You will learn how to write custom rules that utilize Rego iteration so that your policies can apply to resources of arbitrary size.


<!-- [Envoy](https://www.envoyproxy.io/docs/envoy/v1.10.0/intro/what_is_envoy) is a L7 proxy and communication bus designed for large modern service oriented architectures. Envoy (v1.7.0+) supports an [External Authorization filter](https://www.envoyproxy.io/docs/envoy/v1.10.0/intro/arch_overview/ext_authz_filter.html) which calls an authorization service to check if the incoming request is authorized or not.  This External Authorization feature enables policies to be enforced on every one of the application's APIs, but everything must be configured correctly upon deployment.  Thus, policies ensuring that applications are properly deployed with Envoy are vital to ensure that application-level API policies actually get enforced.    -->

## Prerequisites

This tutorial requires Kubernetes 1.14 or later. To run the tutorial locally, we recommend using [minikube](https://kubernetes.io/docs/getting-started-guides/minikube) in version `v1.0+` with Kubernetes 1.14 (which is the default).


## Steps


### 1. Preparation

In Part 1, you started minikube, installed OPA/Styra DAS onto your cluster, and created a DAS System where your policies live.  If you have not already done that, follow the instructions below.

First start minikube:

```bash
minikube start
```

Next install OPA onto your cluster as an admission controller, configured to download policies and upload decision logs to TENANT.styra.com.  This will let you manage the policies enforced via admission control through TENANT.styra.com.

1. Go to TENANT.styra.com
1. Create a new System by clicking on the plus icon
   * **System type**: Kubernetes
   * **System name**: something unique so you can recognize it in the list (e.g. include your name)
1. Follow the `kubectl` install instructions on TENANT.styra.com that are shown after creating your system.  (If you clicked away from the screen and need to find the install instructions, click on your system in the left-hand side, then on `Settings` and then `Install`.)


### 2.Prepare to write a policy that requires a sidecar

Sidecars are a common pattern in Kubernetes--they pull functionality out of say a microservice into a separate container but run that container as part of the same Pod.  For example, it is common to run a network proxy like [Envoy](https://www.envoyproxy.io/docs/envoy/v1.10.0/intro/what_is_envoy) as a sidecar to handle network connectivity for the app.

Here you will write a Rego policy that requires an Envoy sidecar to exist in every Pod.  (In reality you would not require Envoy in EVERY pod; you would only require it in pods that have certain characteristics.)  A sidecar is simply a container whose image name is `"envoyproxy/envoy"`.

For example, the following Pod violates te policy because none of the containers contain the envoy image.  Verify that this Pod is accepted on your Kubernetes cluster.

Put the following YAML into say file `pod.yaml`.

**pod.yaml**

```yaml
kind: Pod
apiVersion: v1
metadata:
  name: myapp
  labels:
    costcenter: abc123
spec:
  containers:
  - image: nginx:1.2
    name: nginx-frontend
  - image: mysql:7.9
    name: mysql-backend
```

```shell
kubectl apply -f pod.yaml
```


### 3. Write a policy that requires Envoy to run as a sidecar

To add the rule that every pod requires Envoy to run as a sidecar, open up the System > Validating > Rules file and write a rule.  Remember that when Kubernetes hands a request over to an admission controller like OPA, it wraps some additional information around the resource, and you end up with a JSON object like the one shown below.

```
{
  "kind": "AdmissionReview",
  "request": {
    "kind": {
      "kind": "Pod",
      "version": "v1"
    },
    "object": {
      "metadata": {
        "name": "myapp",
        "labels": {
            "costcenter": "abc123"
        }
      },
      "spec": {
        "containers": [
          {
            "image": "nginx:1.2",
            "name": "nginx-frontend"
          },
          {
            "image": "mysql:7.9",
            "name": "mysql-backend"
          }
        ]
      }
    }
  }
}
```

Remember that the rules you write for the Styra DAS construct an `enforce` set that contains objects of the form `{"allowed": <boolean>, "message": <string>}`.  The `allowed` field dictates whether the rule allows or denies the request, and the `message` field is a string that includes an error message.  The `message` is used even when the request is allowed as a way of returning monitoring results.


```ruby
enforce[decision] {
  not envoy_is_sidecar
  ...
  decision := {
    "allowed": false,
    "message": msg
  }
}

envoy_is_sidecar {
  ...
}

```
<!-- Untested answers -->
<!-- input.request.kind.kind == "Pod"; msg := "Pods must contain Envoy as a sidecar" >
<!-- some i
    startswith(input.request.object.spec.containers[i].image, "envoyproxy/envoy")
-->

Your first task is to fill in the `enforce` rule's `...` above with the following conditions.

* The `kind` is a `Pod`
* The variable `msg` gets assigned to a string that tells the user about the mistake

Your second task is to fill in the helper definition for `envoy_is_sidecar`.  To do that, you need to check if there is some container whose `image` starts with `envoyproxy/envoy`.


### 4. Validate your Envoy-sidecar policy

Once your policy is written, perform the following tasks to verify that it does what you think.  Part 1 of this tutorial series provides additional details.

* Manual testing using the `Preview` button and the JSON input shown above.
* Unit testing by writing tests in the `System > Validating > Test` file and the `Validate` button.
* Impact analysis via the `Validate` button.

Once your testing is done, publish your policy and verify that it is working properly.  First delete the pod you created earlier.

```shell
kubectl delete -f pod.yaml
```

Once you've given OPA a minute or two to download, try to create your pod again and see that k8s rejects it.

```shell
$ k apply -f pod.yaml
Error from server: error when creating "pod.yaml": admission webhook "validating-webhook.openpolicyagent.org" denied the request: Enforced: YOUR ERROR MESSAGE HERE.
```


### 5. Check out the decision log

Take a look at the decisions that OPA is making on the cluster.

1. Click on your system in the left-hand navigation
1. Click on the `Decisions` tab
1. Search for all the pods by entering `kind: pod` in the filter text area.

OPA batches the decisions and sends them up at similar frequency to when it downloads policy.  So if you don't see the decision you expect, wait a little bit and check back again.

If you find a decision that you'd like to test against your policy, click the `Reply Decision` button.  That will load that input up into the policy editor and show you which of the rules in the current policy would allow or deny that decision.


### 6. Write a policy that prohibits network ports

Another interesting class of OPA policies restricts what networking is safe for the cluster.  Here is a sample NetworkPolicy object as seen by OPA at admission control time.  Notice that it specifies what ingress traffic is allowed on the cluster, including which ports are open.  In this step, you will write a policy that limits the ports to a predefined set.

```json
{
  "kind": "AdmissionReview",
  "request": {
    "kind": {
      "kind": "NetworkPolicy",
      "version": "networking.k8s.io/v1"
    },
    "object": {
      "metadata": {
        "name": "netpolicy1",
        "namespace": "dev",
        "labels": {
          "application": "app123"
        }
      },
      "spec": {
        "podSelector": {},
        "ingress": [
          {
            "from": [
              {
                "ipBlock": {
                  "cidr": "192.168.164.8/32"
                }
              }
            ],
            "ports": [
              {
                "port": 443,
                "protocol": "TCP"
              }
            ]
          }
        ],
        "egress": [
          {
            "to": [
              {
                "ipBlock": {
                  "cidr": "192.168.165.50/32"
                }
              }
            ],
            "ports": [
              {
                "port": 21,
                "protocol": "TCP"
              }
            ]
          }
        ]
      }
    }
  }
}
```

To do that your task is again to write the following Rego rule by filling in `...`.

```rego
permitted_ports := ...

enforce[decision] {
  ...
  decision := {
    "allowed": false,
    "message": msg
  }
}
```
<!-- Untested answer
  input.request.kind.kind == "NetworkPolicy"
  some i,j
  port := input.request.object.spec.ingress[i].ports[j].port
  not permitted_ports[port]
  msg := sprintf("port %v is prohibited", [port])

-->

The `permitted_ports` variable should be assigned the set of port numbers `443`, `80`, `8080`.

The `enforce` rule should check the following conditions.
* The `kind` is a NetworkPolicy
* There is some ingress port that is not in the set of permitted_ports
* The variable `msg` is assigned an appropriate error message for the user


### 7. Verify that your rule behaves as you expected

To verify that your rule does what you expect, try

* Manual testing via the `Preview` button (or if you're feeling confident just skip this and go straight to unit testing)
* Unit testing via the file `System > Validating > Tests` and the `Validate` button
* Impact analysis via the `Validate` button
* Real-world verification by Publishing your policy and trying to create a violating resource via `kubectl`

Here is a sample NetworkPolicy that you can use with `kubectl`.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: test-network-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - ipBlock:
        cidr: 172.17.0.0/16
        except:
        - 172.17.1.0/24
    - namespaceSelector:
        matchLabels:
          project: myproject
    - podSelector:
        matchLabels:
          role: frontend
    ports:
    - protocol: TCP
      port: 6379
  egress:
  - to:
    - ipBlock:
        cidr: 10.0.0.0/24
    ports:
    - protocol: TCP
      port: 5978
```

### 8. Comprehensions (optional)
Here we get a bit of experience with comprehensions.

First write a comprehension that finds the set of all ports open on ingress.  Use the AdmissionReview NetworkPolicy object shown earlier as the input.

```rego
open_on_ingress := { <missing logic> }
```

<!-- Untested answer
open_on_ingress := {p | some i,j; p := input.request.object.spec.ingress[i].ports[j].port} -->

Now write the rego for an object that maps each protocol (e.g. TCP or UDP) to the list of ports that are open for that protocol on ingress.

```rego
protocol_to_port := <missing logic>
```

<!-- Untested answer
protocol_to_port[prot] = ports {
    some i, j
    protocol := input.request.object.spec.ingress[i].ports[j].protocol
    ports := {p.port | p := input.request.object.spec.ingress[_].ports[_]
                       p.protocol == protocol}
} -->

## Wrap Up

Congratulations for finishing the tutorial!  You learned a number of things about Kubernetes admission control with OPA and Styra:

* The OPA-Kubernetes integration gives you fine-grained control over the resources deployed on your cluster
* You can use Styra to write policies, distribute policies to OPA, log OPA's decisions, understand those decisions

