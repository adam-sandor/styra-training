# Tutorial: Accessing External Services

In the [Envoy Tutorial](./tutorial-envoy.md), we leveraged Envoy's [External Authorization filter](https://www.envoyproxy.io/docs/envoy/v1.10.0/intro/arch_overview/ext_authz_filter.html) to check if an incoming request is authorized or not. We can use the same filter to authorize all outbound requests from a Kubernetes pod.

## Goals

This tutorial will take you through the steps of modifying the
[CarStore](../../opa/microservices/car-store-code/) application that was used in the [Envoy Tutorial](./tutorial-envoy.md) to make calls to external services and then show how Envoy’s External authorization filter can be used with OPA and Styra as an authorization service to control access to those external services.

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

default allow_envoy = {
  "allowed": false,
  "headers": {"x-ext-auth-allow": "no"},
  "body": "Unauthorized Request",
  "http_status": 403
}

# incoming request
allow_envoy = response {
  input.attributes.request.http.method == "GET"
  input.parsed_path[0] == "httpbin"
  response := {
    "allowed": true,
    "headers": {"x-ext-auth-allow": "yes"}
  }
}

# incoming request
allow_envoy = response {
  input.attributes.request.http.method == "GET"
  input.parsed_path[0] == "random"
  response := {
    "allowed": true,
    "headers": {"x-ext-auth-allow": "yes"}
  }
}

# allow outbound calls to httpbin.org
allow_envoy = response {
  input.attributes.request.http.method == "GET"
  input.attributes.request.http.host == "httpbin.org"
  response := {
    "allowed": true,
    "headers": {"x-ext-auth-allow": "yes"}
  }
}
```

Once we start running Envoy and OPA, it will pull down this policy and start enforcing it. The rule `allow_envoy` allows incoming requests for the `/httpbin` and `/random` endpoints. Additionally the rule blocks all outbound calls except to `httpbin.org`.

### 4. Configure the Decision Mapping

Because you are using a custom system, you need to describe a little about the decision format so TENANT.styra.com knows what an allow decision looks like.  This also lets you control the format of the decision log entries.

1. Go to TENANT.styra.com > SYSTEM > Settings > Decisions
1. Under `Path to decision`, enter `result.allowed`
1. Under `Columns` near the bottom, enter the following
   * Search key: `method`, Path to value: `input.attributes.request.http.method`
   * Search key: `path`, Path to value: `input.attributes.request.http.path`
   * Search key: `host`, Path to value: `input.attributes.request.http.host`
1. Click `Save Changes` at the bottom

### 5. Modify CarStore App

We will now modify the [CarStore](../../opa/microservices/car-store-code/) app such that it makes outbound calls to publicly accessible services.

We will add two new APIs to the app that will make the external calls. The first one `/httpbin` will choose a random endpoint served by `httpbin.org` and then call it. The second API `/random` will call a random external website from a given list of websites. 
 
```python
import random

# External Endpoints

# endpoint to make an external call to httpbin.org
@app.route("/httpbin", methods=["GET"])
def httpbin_get():
    try:
        endpoints = ["headers", "ip", "user-agent", "json", "uuid", "anything"]
        # choose a random endpoint
        url = 'http://httpbin.org/' + random.choice(endpoints)
        response = requests.get(url)
    except Exception as e:
        app.logger.exception("Unexpected error querying httpbin.org.")
        abort(500)
    if response.status_code != 200:
        abort(response.status_code)
    return jsonify({'result': response.json()})


# endpoint to make an external call to a random service
@app.route("/random", methods=["GET"])
def random_get():
    try:
        urls = ["http://ip.jsontest.com/", "http://iczn.org", "http://open-up.eu", "http://zootaxa.info/"]
        # choose a random url
        response = requests.get(random.choice(urls))
    except Exception as e:
        app.logger.exception("Unexpected error querying external service.")
        abort(500)

    if response.status_code != 403:
        app.logger.exception("Expected OPA to deny request but got non-403 status code.")
        abort(500)
    abort(403)
```

Build the server image

```bash
docker build -t openpolicyagent/demo-car-store:v3 -f Dockerfile .
```

### 6. Launch Envoy, OPA and App

Create a file that launches a sample App (just a python process) with Envoy and OPA as sidecars.  There are comments before each of the k8s resources describing what they do. 

Be sure to fill in the SYSTEMID (2 instances), TOKEN (1 instance), and TENANT (1 instance) in the OPA configuration.  
* **SYSTEMID**: The numeric ID for the system.  To find this, click on your system and the number will be shown in the upper part of the right-hand pane.
* **TOKEN**: An API token for TENANT.styra.com.  Generate an API token by going to TENANT.styra.com > Workspace > Settings > API Tokens.  Name it something unique that you will remember so you can delete it once you are finished.
* **TENANT.styra.com**: Replace TENANT with your tenant name.

The `proxy-init` container installs iptables rules to redirect all container traffic through the Envoy proxy sidecar. In this case, Envoy will query OPA to check whether both an incoming and an outgoing request is allowed or not.

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
          image: openpolicyagent/proxy_init:proxy-in-and-out
          args: ["-p", "8000", "-u", "1111"]
          securityContext:
            capabilities:
              add:
              - NET_ADMIN
            runAsNonRoot: false
            runAsUser: 0
      containers:
        - name: app
          image: openpolicyagent/demo-car-store:v3
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
                                prefix: "/headers"
                              route:
                                cluster: serviceExt
                            - match:
                                prefix: "/ip"
                              route:
                                cluster: serviceExt
                            - match:
                                prefix: "/user-agent"
                              route:
                                cluster: serviceExt
                            - match:
                                prefix: "/json"
                              route:
                                cluster: serviceExt
                            - match:
                                prefix: "/uuid"
                              route:
                                cluster: serviceExt
                            - match:
                                prefix: "/anything"
                              route:
                                cluster: serviceExt
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
          listener_filters:
          - name: envoy.listener.original_dst
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
        - name: serviceExt
          type: ORIGINAL_DST
          connect_timeout: 0.25s
          lb_policy: ORIGINAL_DST_LB
          dns_lookup_family: V4_ONLY
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

Check that outbound calls to different endpoints of `httpbin.org` are allowed. Run the below command multiple times to see different outputs.

```bash
curl -i $SERVICE_URL/httpbin
```

Check that outbound calls to **ANY** other external service are blocked. Run the below command multiple times.

```
curl -i $SERVICE_URL/random
```

### 9. Verify remaining functionality

Ensure that everything is working as expected.

* **Decision log**.  Check that both the Envoy decisions and your Python decisions are appearing in the decision log.  In the left-hand navigation, click on your system, and then click on `Decisions` in the right-hand pane.
* **Unit tests**.  Write unit tests in the `Tests` file and validate that they pass from the `Rules` file by running `Validate`.
* **Policy changes**.  Modify your `Rules` policy, `Validate` that the results are what you expect, publish the results, and check that OPA is properly enforcing your new policy.



## Wrap Up

Congratulations for finishing the tutorial! 

In this tutorial we saw how we can use OPA and Styra to enforce fine-grained access control policies over outbound calls made by a service to external services outside the cluster.

We also saw how the same Envoy external authorization filter can be leveraged for both incoming and outgoing calls thereby allowing stricter control over traffic entering and leaving the pod.