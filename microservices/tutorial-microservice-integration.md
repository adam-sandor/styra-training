# Tutorial: Integrating Apps with OPA

OPA provides simple HTTP APIs that enable quick integration with apps written
in any language. This tutorial shows how you can offload authorization
decisions from your application using OPA.  Even if you are using a service mesh,
there are authorization decisions that require additional information that the microservice
itself needs to provide to make a policy decision.  For example, the policy that everyone
may read the details of a resource, but only the owner may update the details
of that resource requires knowledge of who the owner of a resource is, which the service
mesh will not know.

## Goals

This tutorial will take you through the steps of modifying the
[CarStore](../../opa/microservices/car-store-code/) application that was used in the [Envoy Tutorial](./tutorial-envoy.md) to use OPA to enforce authorization policies that require external data. At the end of this tutorial, you will be familiar with integrating apps with OPA's HTTP enforcement API.

## Prerequisties

This tutorial requires Kubernetes 1.14 or later. To run the tutorial locally, we recommend using [minikube](https://kubernetes.io/docs/getting-started-guides/minikube) in version `v1.0+` with Kubernetes 1.14 (which is the default).

## Steps

### 1. Start minikube

```bash
minikube start
```

### 2. Use Minikube docker daemon on host

```bash
eval $(minikube docker-env)
```

### 3. Set up policies in TENANT.styra.com

To manage the policies enforced by OPA now running on each host:

1. Go to TENANT.styra.com
1. Create a new system by clicking the plus sign next to `Systems`
   * **Type**: Custom
   * **Name**: something unique that you can remember
1. Open up the system and click on `Rules`
1. Replace what is there with the following.
1. Publish the policy using the left-arrow next to the trashcan


```ruby
package rules

import data.dataset

# Envoy allows all requests to pass
allow_envoy = response {
  response := {
    "allowed": true,
    "headers": {"x-ext-auth-allow": "yes"}
  }
}

default allow_app = false

allow_app_result = {"allowed": allow_app}

# /cars
allow_app {
    # everyone in the world can see the car listing
    input.method == "GET"
    input.path == ["cars"]
}

allow_app {
    # only managers can create a new car
    input.method == "POST"
    input.path == ["cars"]
    user_is_manager
}

# /cars/{carid}
allow_app {
    # only employees can see car details
    input.method == "GET"
    input.path = ["cars", carid]
    user_is_employee
}

allow_app {
    # only managers can update an existing car
    input.method == "PUT"
    input.path = ["cars", carid]
    user_is_manager
}

allow_app {
    # only managers can delete a car
    input.method == "DELETE"
    input.path = ["cars", carid]
    user_is_manager
}

# /cars/{carid}/status
allow_app {
    input.method == "GET"
    input.path = ["cars", carid, "status"]
    user_is_employee
}

allow_app {
    input.method == "POST"
    input.path = ["cars", carid, "status"]
    user_is_employee
}


######################
# Helpers

user_is_manager {
    input.users[input.auth.principal.username].title != "salesperson"
}

user_is_employee {
    input.users[input.auth.principal.username]
}
```

Once we start running Envoy and OPA, it will pull down this policy and start enforcing it. Envoy's external authorization filter will query `allow_envoy` for a policy decision while the CarStore app will query `allow_app_result` for the same.

### 4. Configure the Decision Mapping

Because you are using a custom system, you need to describe a little about the decision format so TENANT.styra.com knows what an allow decision looks like.  This also lets you control the format of the decision log entries.

1. Go to TENANT.styra.com > SYSTEM > Settings > Decisions
1. Under `Path to decision`, enter `result.allowed`
2. Click `Save Changes` at the bottom

### 5. Integrate CarStore with OPA

While some policies can be implemented in the Envoy check-authz filter,
the information available to the policy decision is limited to the information
Envoy has at its disposal.  Sometimes the policy requires additional information.
For example the policy "a resource can only be modified by its owner" cannot
be enforced via Envoy if neither Envoy nor OPA know who a resource's owner
is.  In these cases the application itself may need to ask for authorization from
OPA directly and supply the needed information.

In this step we modify the [CarStore app](../../opa/microservices/car-store-code/), which
is written in Python, to ask OPA for authorization on all of the HTTP endpoints in
the app using Python decorators.

Alternatively, you could use roughly the same logic as shown below
and call out to OPA only in those code locations that you know the Envoy policy does
not suffice.  The benefit of one-off callouts is improved performance; the drawback
is tighte3r coupling between the policy and the integration with
OPA since only those locations that explicitly call-out to OPA support policies that
depend on the external data provided by those callouts.

To get started, use Flask's `@app.before_request` Python decorator to execute
an authorization query against OPA for all requests.

```python
from flask import request
from flask import abort

@app.before_request
def check_authorization():
    # Perform authorization check against request. Call abort if
    # request should be denied. The request value includes all
    # of the attributes that describe the current HTTP request
    # (e.g., the method, path, etc.)
```

The `check_authorization` function is called every time the CarStore app
receives an HTTP request. If `check_authorization` returns, the request will
be allowed. On the other hand, if the request should be denied,
`check_authorization` can call Flask's `abort` helper to raise an exception
and reject the request.

Integrating OPA into enforcement points such as HTTP endpoints typically
involves three steps.

1. Gathering and constructing the input to pass to OPA.
2. Executing the query against OPA.
3. Processing the response from OPA.

In this case, the `check_authorization` function can build the input to pass
to OPA from the incoming request. For simplicity, the HTTP Authorization
header is used to obtain the user information. In real-world scenarios, the
HTTP request would be authenticated using a separate system that would return
similar user information.

A list of sample users along with their roles is also passed to OPA as input using the `users` field.

```python
input = json.dumps({
    "input": {
        "method": request.method,
        "path": request.path.strip().split("/")[1:],
        "auth": get_authentication(request),
        "users": users,
    }
})
```

To query OPA, `check_authorization` only has to execute a single HTTP POST
request that passes the input along.

```python
response = requests.post(OPA_URL + POLICY_PATH, data=input)
```

Finally, when `check_authorization` receives the response from OPA it can
interpret the result and either allow (`true`) or deny (`false`) the request.

```python
allowed = response.json()['result'].get('allowed', False)

if not allowed:
    abort(403)
else:
    pass  # Nothing to do.
```

Now that you have seen how you can integrate OPA into an enforcement point,
go to the CarStore app directory and update the **server.py** file to include the sample code below.

```python
# Sample Users
users = {
    "alice":   {"manager": "charlie", "title": "salesperson"},
    "bob":     {"manager": "charlie", "title": "salesperson"},
    "charlie": {"manager": "dave",    "title": "manager"},
    "dave":    {"manager": None,      "title": "ceo"}
}

# Authorization Handler

OPA_URL = os.environ.get("OPA_URL", "http://localhost:8181")
POLICY_PATH = os.environ.get("POLICY_PATH", "/v1/data/rules/allow_app_result")


@app.before_request
def check_authorization():
    try:
        input = json.dumps({
            "input": {
                "method": request.method,
                "path": request.path.strip().split("/")[1:],
                "auth": get_authentication(request),
                "users": users,
            }
        }, indent=2)
        url = OPA_URL + POLICY_PATH
        app.logger.debug("OPA query: %s. Body: %s", url, input)
        response = requests.post(url, data=input)
    except Exception as e:
        app.logger.exception("Unexpected error querying OPA.")
        abort(500)

    if response.status_code != 200:
        app.logger.error("OPA status code: %s. Body: %s",
                         response.status_code, response.json())
        abort(500)

    allowed = response.json()['result'].get('allowed', False)
    app.logger.debug("OPA result: %s", allowed)
    if not allowed:
        abort(403)


def get_authentication(request):
    user_encoded = request.headers.get('Authorization', "Anonymous:none")
    _, _, user_encoded = user_encoded.partition("Basic ")
    user, _, _ = base64.b64decode(user_encoded).decode("utf-8").partition(":")
    return {
        "principal": {
            "username": user,
        },
    }
```

Build the server image

```bash
docker build -t openpolicyagent/demo-car-store:v2 -f Dockerfile .
```

### 6. Launch Envoy, OPA and App

Create a file that launches a sample App (just a python process) with Envoy and OPA as sidecars.  There are comments before each of the k8s resources describing what they do. 

Be sure to fill in the SYSTEMID (2 instances), TOKEN (1 instance), and TENANT (1 instance) in the OPA configuration.  
* **SYSTEMID**: The numeric ID for the system.  To find this, click on your system and the number will be shown in the upper part of the right-hand pane.
* **TOKEN**: An API token for TENANT.styra.com.  Generate an API token by going to TENANT.styra.com > Workspace > Settings > API Tokens.  Name it something unique that you will remember so you can delete it once you are finished.
* **TENANT.styra.com**: Replace TENANT with your tenant name.


**all.yaml**:

```yaml
# https://github.com/tsandall/minimal-opa-envoy-example

# Namespace for deployment
kind: Namespace
apiVersion: v1
metadata:
  name: cars
  labels:
    openpolicyagent.org/webhook: ignore
---
# OPA + Envoy + App deployment
kind: Deployment
apiVersion: apps/v1
metadata:
  name: example-app
  namespace: cars
  labels:
    app: example-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: example-app
  template:
    metadata:
      labels:
        app: example-app
    spec:
      initContainers:
        - name: proxy-init
          image: openpolicyagent/proxy_init:v2
          args: ["-p", "8000", "-u", "1111"]
          securityContext:
            capabilities:
              add:
              - NET_ADMIN
            runAsNonRoot: false
            runAsUser: 0
      containers:
        - name: app
          image: openpolicyagent/demo-car-store:v2
          imagePullPolicy: Never
          ports:
            - containerPort: 5000
        - name: envoy
          image: envoyproxy/envoy:v1.10.0
          securityContext:
            runAsUser: 1111
          volumeMounts:
          - readOnly: true
            mountPath: /config
            name: proxy-config
          args:
          - "envoy"
          - "--config-path"
          - "/config/envoy.yaml"
        - name: opa
          image: openpolicyagent/opa:0.15.0-istio-6
          securityContext:
            runAsUser: 1111
          volumeMounts:
          - readOnly: true
            mountPath: /config
            name: opa-config-vol
          args:
          - "run"
          - "--server"
          - "--config-file=/config/conf.yaml"
      volumes:
      - name: opa-config-vol
        configMap:
          name: opa-envoy-config
      - name: proxy-config
        configMap:
          name: proxy-config

---
# OPA configuration
kind: ConfigMap
apiVersion: v1
metadata:
  name: opa-envoy-config
  namespace: cars
data:
  conf.yaml: |
    bundle:
      name: systems/SYSTEMID/rules
      service: styra
    labels:
      policy-type: custom/rules
      system-id: SYSTEMID
      system-type: custom
    plugins:
      envoy_ext_authz_grpc:
        addr:    :9191
        query:   data.rules.allow_envoy
        dry-run: false
    decision_logs:
      service: styra
      reporting:
        min_delay_seconds: 3
        max_delay_seconds: 10
    status:
      service: styra
    services:
    - credentials:
        bearer:
          token: TOKEN
      name: styra
      url: https://TENANT.styra.com/v1
---
# Envoy configuration
kind: ConfigMap
apiVersion: v1
metadata:
  name: proxy-config
  namespace: cars
data:
  envoy.yaml: |
    static_resources:
      listeners:
        - address:
            socket_address:
              address: 0.0.0.0
              port_value: 8000
          filter_chains:
            - filters:
                - name: envoy.http_connection_manager
                  typed_config:
                    "@type": type.googleapis.com/envoy.config.filter.network.http_connection_manager.v2.HttpConnectionManager
                    codec_type: auto
                    stat_prefix: ingress_http
                    route_config:
                      name: local_route
                      virtual_hosts:
                        - name: backend
                          domains:
                            - "*"
                          routes:
                            - match:
                                prefix: "/"
                              route:
                                cluster: service
                    http_filters:
                      - name: envoy.ext_authz
                        config:
                          failure_mode_allow: false
                          grpc_service:
                            google_grpc:
                              target_uri: 127.0.0.1:9191
                              stat_prefix: ext_authz
                            timeout: 0.5s
                      - name: envoy.router
                        typed_config: {}
      clusters:
        - name: service
          connect_timeout: 0.25s
          type: strict_dns
          lb_policy: round_robin
          load_assignment:
            cluster_name: service
            endpoints:
              - lb_endpoints:
                  - endpoint:
                      address:
                        socket_address:
                          address: 127.0.0.1
                          port_value: 5000
    admin:
      access_log_path: "/dev/null"
      address:
        socket_address:
          address: 0.0.0.0
          port_value: 8001
---
```

```shell
$ kubectl apply -f all.yaml
```

### 7. Create a Service to expose HTTP server

So that you can access the sample application from outside of minikube, open up a local port and send traffic to your application.

```bash
kubectl expose deployment example-app -n cars --type=NodePort --name=example-app-service --port=5000
```

Set the `SERVICE_URL` environment variable to the service's IP/port.

```bash
export SERVICE_PORT=$(kubectl get service example-app-service -n cars -o jsonpath='{.spec.ports[?(@.port==5000)].nodePort}')
export SERVICE_HOST=$(minikube ip)
export SERVICE_URL=$SERVICE_HOST:$SERVICE_PORT
echo $SERVICE_URL
```

Here is some sample output:

```bash
192.168.99.113:31056
```

### 8. Check that policy is being enforced

In this step you will test policy through a combination of HTTP requests.

Using the CarStore API, exercise the policy. Try each of the commands below and verify the output is what you expect.

```bash
# Managers can create cars.
curl -i -u charlie:password -X POST $SERVICE_URL/cars -H 'Content-Type: application/json' -d '{"id":"11111111-1111-1111-1111-111111111111","model":"Bravo","registeredVehicleId":"Omiya-500-ka-1111","ownerId":"aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa"}'

# Employees can list cars.
curl -i -u alice:password -s $SERVICE_URL/cars

# Employees can set car status.
curl -i -u alice:password -s -X POST -H 'Content-Type: application/json' $SERVICE_URL/cars/11111111-1111-1111-1111-111111111111/status -d '{"sensedAt":"2017-07-04T07:31:28.121589463Z","position":{"latitude":35.720345298234555,"longitude":139.5619397088644},"mileage":1000.0468836604865,"speed":7.671673190917932,"fuel":60.99531163395134,"tyrePressure":249.99953116339512}'

# Employees can get car status.
curl -i -u alice:password -s $SERVICE_URL/cars/11111111-1111-1111-1111-111111111111/status

# Employees are not allowed to create cars.
curl -i -u alice:password -X POST $SERVICE_URL/cars -H 'Content-Type: application/json' -d '{"id":"66666666-6666-6666-6666-666666666666","model":"Bravo","registeredVehicleId":"Omiya-500-ka-6666","ownerId":"kkkkkkkk-kkkk-kkkk-kkkk-kkkkkkkkkkkk"}'

# Employees are not allowed to delete cars.
curl -i -u alice:password -X DELETE $SERVICE_URL/cars/11111111-1111-1111-1111-111111111111

# But managers can delete cars.
curl -i -u charlie:password -X DELETE $SERVICE_URL/cars/11111111-1111-1111-1111-111111111111
```

### 9. Verify remaining functionality

Ensure that everything is working as expected.

* **Decision log**.  Check that both the Envoy decisions and your Python decisions are appearing in the decision log.  In the left-hand navigation, click on your system, and then click on `Decisions` in the right-hand pane.
* **Unit tests**.  Write unit tests in the `Tests` file and validate that they pass from the `Rules` file by running `Validate`.
* **Policy changes**.  Modify your `Rules` policy, `Validate` that the results are what you expect, publish the results, and check that OPA is properly enforcing your new policy.


## Wrap Up

Congratulations for finishing the tutorial!

At this point you have integrated an application with OPA and learned:

* How to write code that enforces policy decisions using OPA
* How to load policy and data into OPA
* How to execute policy queries against OPA's HTTP API