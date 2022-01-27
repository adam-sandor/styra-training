# Tutorial: Basic Rules with K8s Config Authorization and Styra DAS

[Kubernetes](https://kubernetes.io/) is a popular, open-source system for automating deployment, scaling, and management of containerized applications.

## Goals

This tutorial shows how Kubernetes's admission controllers can be used with Styra DAS to enforce policies that dictate which resources can be deployed to the cluster and how those resources should be configured.  You will learn how to use the DAS's pre-built rules as well as write one of your own custom rules and test that it is doing what you think it should.


## Prerequisites

This tutorial requires Kubernetes 1.14 or later. To run the tutorial locally, we recommend using [minikube](https://kubernetes.io/docs/getting-started-guides/minikube) in version `v1.0+` with Kubernetes 1.14 (which is the default).


## Steps


### 1. Start Minikube

```bash
minikube start
```

### 2. Install OPA/Styra onto the k8s cluster

Install OPA onto your cluster as an admission controller, configured to download policies and upload decision logs to TENANT.styra.com.  This will let you manage the policies enforced via admisison control through TENANT.styra.com.

1. Go to TENANT.styra.com
1. Create a new System by clicking on the plus icon
   * **System type**: Kubernetes
   * **System name**: something unique so you can recognize it in the list (e.g. include your name)
1. Follow the `kubectl` install instructions on TENANT.styra.com that are shown after creating your system.  (If you clicked away from the screen and need to find the install instructions, click on your system in the left-hand side, then on `Settings` and then `Install`.)


### 3. Put your first policy in place

To test that everything is working, put a policy in place that prohibits using the `latest` version of any image.

1. Go to TENANT.styra.com
1. On the left-hand-side navigation, open the system you created
1. Click to open Validating
1. Click on `Rules`
1. Click the `Add Rule` button and type in `latest` to filter down to the proper rule `Prohibit :latest image tag`
1. Click on the rule to add it to your policy
1. Click on the `Enforce` button in the middle gutter to switch the rule from Monitor mode into Enforce mode.
1. Click the `Publish` button (the left-arrow near the trashcan)


### 4. Exercise your policy

Check that your policy is working by trying to launch a Pod that has images with the `latest` version.

Put the following YAML into say file `podlatest.yaml`.

**podlatest.yaml**

```yaml
kind: Pod
apiVersion: v1
metadata:
  name: myapp
spec:
  containers:
  - image: nginx
    name: nginx-frontend
  - image: mysql
    name: mysql-backend
```

Now try to run it on the cluster with `kubectl` and see that it is rejected because of the latest tags.

``` shell
$ kubectl apply -f podlatest.yaml
Error from server: error when creating "podlatest.yaml": admission webhook "validating-webhook.openpolicyagent.org" denied the request: Enforced: Resource Pod/default/myapp should not use the 'latest' tag on container image nginx., Resource Pod/default/myapp should not use the 'latest' tag on container image mysql.
```
If you don't see this error message, make sure that step (2) successfully installed OPA in the `styra-system` namespace.  Another possibility is that OPA has not yet downloaded the latest policy, so `kubectl delete -f podlatest.yaml` and try again after a minute or so.


Once kubernetes has rejected the pod above, change `podlatest.yaml` to use versions on both the images and rerun the `kubectl` command to see that it succeeds.

**podlatest.yaml**

```yaml
kind: Pod
apiVersion: v1
metadata:
  name: myapp
spec:
  containers:
  - image: nginx:1.2
    name: nginx-frontend
  - image: mysql:7.9
    name: mysql-backend
```

```shell
kubectl apply -f podlatest.yaml
```

### 5. Get familiar with the DAS editor (if you haven't already)

1. Go to TENANT.styra.com
1. On the left-hand-side navigation, open the system you created
1. Click to open Validating
1. Click on `Rules`

In the text editor on the right hand side, find some space outside the Latest image rule and add in the following statements.

```ruby
default hello = false

hello {
  input.message == "world"
}
```

Click the `Preview` button.  It will open 2 panes: an Input pane, where you can control what input is provided to the policy, and an Output pane, which shows the result of evaluating policy against that input.  (By default Preview will populate the Input pane with an entry from the decision log, if it can.)  Replace the JSON that is in the Input pane with the following JSON:

```json
{
  "message": "world"
}
```

Click the `Preview` button to see the result of evaluating the rules in the Output pane.

```json
{
  "enforce": [],
  "hello": true
}
```

Add another statement to the policy.

```ruby
at_least_world {
  startswith(input.message, "wor")
  endswith(input.message, "ld")
}
```

Evaluate everything again by clicking the `Preview` button again.  (Make sure to de-select what you copied in before re-evaluating.)  The Output window displays values for all of the variables in the policy.

```json
{
  "at_least_world": true,
  "enforce": [],
  "hello": true
}
```

Now change the Input to the following JSON and click `Preview` again.

```json
{
  "message": "world is not enough"
}
```

Neither of the new rules evaluated to true.

```json
{
  "enforce": [],
  "hello": false
}
```

The reason `hello` is false is obvious.  But if we pretend we don't know why `at_least_world` failed, we need to discover which of the two conditions failed.  To do that, first highlight `startswith(input.message, "wor")`.  Notice that `Preview` has changed to `Preview selection`; click it and see the result in the output pane.

```json
true
```


Now highlight both lines in the body and evaluate again.  This time you see the output pane is empty (i.e., line 1 shows `# No results`), which means the result is `undefined`.  You can highlight anything OPA treats as a query and ask for an evaluation of it.  That includes global variables/rule names, individual expressions like `startswith(input.message, "wor")`, and blocks of expressions like the body of a rule.  This ability to evaluate parts of a rule is invaluable for debugging.



### 6. Write a custom policy that requires a costcenter label

Now we will write a custom rule that requires every pod on a Kubernetes cluster to have a `costcenter` label.  Imagine the user tries to create the Pod shown below.

```yaml
kind: Pod
apiVersion: v1
metadata:
  name: myapp
  labels:
    foo: bar
spec:
  containers:
  - image: nginx:1.2
    name: nginx-frontend
  - image: mysql:7.9
    name: mysql-backend
```

By the time Kubernetes hands that resource off to OPA running as an admission controller,
it looks like the resource that follows (some details omitted for brevity).

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
            "foo": "bar"
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

Now that you have an example input, you can write the policy that says all objects must have a metadata label whose key is `costcenter`.

The rules you write for the Styra DAS construct an `enforce` set that contains objects of the form `{"allowed": <boolean>, "message": <string>}`.  The `allowed` field dictates whether the rule allows or denies the request, and the `message` field is a string that includes an error message.  The `message` is used even when the request is allowed as a way of returning monitoring results.

```ruby
enforce[decision] {
  ...
  decision := {
    "allowed": false,
    "message": msg
  }
}
```

Your task is to fill in the `...` above to implement the following conditions.

* The `kind` is a `Pod`
* The `metadata` fails to include a `costcenter` label
* The variable `msg` gets assigned to a string that tells the user about the mistake


### 7. Test that your solution is correct

First manually test that your rule is correct.
1. Click `Preview`
1. Put the AdmissionReview object from the last step into the Input pane
1. Click `Preview` again to evaluate the result
1. Check that the result includes the following object inside of the `enforce` set.  (It may include additional messages from the other rule you have prohibiting `latest`.)


```yaml
{
  "enforce": [
    {
      "allowed": false,
      "message": "your error message"
    }
  ]
}
```

When you write custom rules for Kubernetes, it's good practice to write unit tests for them, so that as you or someone else change those rules or write new ones, the rules can be tested as a whole for correctness before being enforced.  Unit tests are written in Rego, just like policy.

1. From the left-hand navigation pane, click on the Test file: System > Validating > Test
1. Enter the following test code

```rego
import data.policy["com.styra.kubernetes.validating"].rules.rules as rules

test_labels_positive {
  # assign 'actual' to the result of evaluating the enforce rule using as input the value of variable input_label_without_costcenter
  actual := rules.enforce with input as input_label_without_costcenter
  # the answer you expect to be included
  correct := {"allowed": false, "message": "YOUR STRING"}
  # check that the correct answer is a member of the returned set
  actual[correct]
}

input_label_without_costcenter := {
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
            "foo": "bar"
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

The `import` line says to use `rules` as shorthand for the package in which the policy you just wrote is located.

The `test_labels_positive` rule is a unit test.  All unit tests are boolean rules that start with `test`.  If `test_labels_positive` evaluates to true, then the test passes; otherwise, it fails.

The `input_label_without_costcenter` is a variable assigned the same AdmissionReview object you've seen earlier and is used as the input to the `test_labels_positive`.

Be sure to replace YOUR STRING in the test with whatever message your `enforce` rule returns.

Now run the test by clicking on the `Validate` button in the upper right hand corner.  You should see your test passing in the left-hand-corner.

Now write your own test.  This time write a test for the case when your rule should NOT produce any values.  That is, fill in the following rule.  Hint: the [builtin function](https://www.openpolicyagent.org/docs/latest/policy-reference/#aggregates) `count()` will return the number of elements in a set.

```rego
test_labels_negative {
...
}
```


### 8. Do impact analysis for your policy
Notice that when you click the `Validate` button it not only runs your unit tests but also shows you two other panels.

**Compliance panel (middle)**
The compliance panel in the middle shows all resources currently on the cluster that violate whatever draft policy you have open in the editor.  It is a fantastic way to help you understand whether your cluster is ready to put your rule in place (and to help you fine-tune your rules).  If there are 1 or 2 resources that violate your policy, it may be safe to put that policy into your cluster, but if there are 100s of resources that violate your policy, your cluster may not be ready.

**Decision panel (right)**
The decision panel on the right replays previous decisions that OPA made using your draft policy and lists out the decisions that were made differently by your new policy.  This gives you more insight into how impactful your changes really are.

Change your draft policy a bit, and see how the validate results change. For example:

* Modify the policy to apply where the `kind` is "Lease" instead of "Pod".
* Modify the policy to apply only for certain namespaces.

### 9. Exercise your policy using Kubernetes

Now that you are convinced your new rule is working as expected, click the Publish button to deploy the rule to OPA (the next time OPA phones home).

In the meantime delete the pod you created in the last step.

```shell
kubectl delete -f podlatest.yaml
```

Once you've given OPA a minute or two to download your new policy, try to create your pod again and see that k8s rejects it because while it has versions for each image, it fails to have a costcenter label.

```yaml
kind: Pod
apiVersion: v1
metadata:
  name: myapp
spec:
  containers:
  - image: nginx:1.2
    name: nginx-frontend
  - image: mysql:7.9
    name: mysql-backend
```

```shell
$ kubectl apply -f podlatest.yaml
Error from server: error when creating "podlatest.yaml": admission webhook "validating-webhook.openpolicyagent.org" denied the request: Enforced: YOUR STRING HERE
```


### 10. See the decisions that OPA is making

Take a look at the decisions that OPA is making on the cluster.

1. Click on your system in the left-hand navigation
1. Click on the `Decisions` tab
1. Search for all the pods by entering `kind: pod` in the filter text area.

OPA batches the decisions and sends them up at similar frequency to when it downloads policy.  So if you don't see the decision you expect, wait a little bit and check back again.

If you find a decision that you'd like to test against your policy, click the `Reply Decision` button.  That will load that input up into the policy editor and show you which of the rules in the current policy would allow or deny that decision.



## Wrap Up

Congratulations for finishing the tutorial!  You learned a number of things about Kubernetes admission control with OPA and Styra's DAS:

* The OPA-Kubernetes integration gives you fine-grained control over the resources deployed on your cluster
* You can use Styra to write policies, distribute policies to OPA, log OPA's decisions, understand those decisions

