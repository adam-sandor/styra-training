
# Envoy Tutorial

[Envoy](https://www.envoyproxy.io/docs/envoy/v1.10.0/intro/what_is_envoy) is a L7 proxy and communication bus designed for large modern service oriented architectures. Envoy (v1.7.0+) supports an [External Authorization filter](https://www.envoyproxy.io/docs/envoy/v1.10.0/intro/arch_overview/ext_authz_filter.html) which calls an authorization service to check if the incoming request is authorized or not.

This feature makes it possible to delegate authorization decisions to an external service and also makes the request context available to the service which can then be used to make an informed decision about the fate of the incoming request received by Envoy.

## Goals

The tutorial shows how Envoy’s External authorization filter can be used with OPA and Styra as an authorization service to enforce security policies over API requests received by Envoy. The tutorial also covers examples of authoring custom policies over the HTTP request body.

## Prerequisites

This tutorial requires Kubernetes 1.14 or later. To run the tutorial locally, we recommend using [minikube](https://kubernetes.io/docs/getting-started-guides/minikube) in version `v1.0+` with Kubernetes 1.14 (which is the default).


## Steps


### 1. Start Minikube

```bash
minikube start
```

### 2. Set up policies in TENANT.styra.com

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

allow_envoy = response {
  input.attributes.request.http.method == "GET"
  input.parsed_path[0] == "cars"
  response := {
    "allowed": true,
    "headers": {"x-ext-auth-allow": "yes"}
  }
}
```

Once we start running Envoy and OPA, it will pull down this policy and start enforcing it.

### 3. Configure the Decision Mapping

Because you are using a custom system, you need to describe a little about the decision format so TENANT.styra.com knows what an allow decision looks like.  This also lets you control the format of the decision log entries.

1. Go to TENANT.styra.com > SYSTEM > Settings > Decisions
1. Under `Path to decision`, enter `result.allowed`
1. Under `Columns` near the bottom, enter the following
   * Search key: `method`, Path to value: `input.attributes.request.http.method`
   * Search key: `path`, Path to value: `input.attributes.request.http.path`
   * Search key: `sourceip`, Path to value: `input.attributes.source.address.Address.SocketAddress.address`
1. Click `Save Changes` at the bottom

### 4. Launch Envoy, OPA and App

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
          image: openpolicyagent/demo-car-store:v1
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
          - "--plugin-dir=/app"
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

### 5. Create a Service to expose HTTP server

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

### 6. Check that policy is being enforced

Check that `anyone` can list cars.

```bash
$ curl -i http://$SERVICE_URL/cars
```

Check that no one can update cars.
```
curl -i -d '{"id":"11111111-1111-1111-1111-111111111111","model":"Bravo","registeredVehicleId":"Omiya-500-ka-1111","ownerId":"aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa"}' -H "Content-Type: application/json" -X POST http://$SERVICE_URL/cars
```

### 7. Codify your tests

As you make changes in your policy you won't want to need to rerun the tests you just did manually, so write some proper tests.

Go to your System > Tests and enter the following rules.  Tests are written in the same language as policies.

```
package test
import data.rules


test_allow_get {
  in := {"attributes": {"request": {"http": {"method": "GET"}}},
         "parsed_path": ["cars"]}
  actual := rules.allow_envoy with input as in   
  actual.allowed
}
```

Now go back to your `Rules` file and click Validate.  The left-hand pane in the dialog that appears shows your test results, which should succeed.  

### 8. See the decisions in the Decision Log

To see the decisions being recorded, click on your system name in the left-hand navigation and then click on the Decisions tab in the right-hand pane.

OPA batches decisions and sends them periodically, so you may need to wait a bit before the latest decision you made shows up.  You can repeat the previous step to run additional requests and fill up OPA's buffer more quickly if you like.


### 9. Change the policy

Change the policy by going back to your system's `Rules` file and adding the following rule that allows a `POST` TO `/cars`

```
allow_envoy = response {
  input.attributes.request.http.method == "POST"
  input.parsed_path[0] == "cars"
  response := {
    "allowed": true,
    "headers": {"x-ext-auth-allow": "yes"}
  }
}
```

Before publishing, click the `Validate` button.  The right-hand pane of the dialog that appears replays previous decisions using whatever draft policy you have open in your editor and shows you which decisions changed so that you have a sense of the impact of your policy change.  You should see all the past decisions on `POST` that were denied change to allowed.

Publish the policy and verify that it is allowing the `POST /cars`.  OPA downloads policy every minute, so you may need to wait a bit to see the change take effect.  This is the same command that was previously rejected; it will now succeed.

```bash
curl -i -d '{"id":"11111111-1111-1111-1111-111111111111","model":"Bravo","registeredVehicleId":"Omiya-500-ka-1111","ownerId":"aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa"}' -H "Content-Type: application/json" -X POST http://$SERVICE_URL/cars
```


## Wrap Up

Congratulations for finishing the tutorial!  You learned a number of things about Envoy with OPA and Styra:

* The OPA-Envoy integration gives you fine-grained access control over microservice API authorization
* You can use Styra to write policies, test policies, distribute policies to OPA, perform impact analysis on proposed policy changes, and manage the decisions that OPA makes

Look here for additional information on the OPA-Envoy integration
* https://github.com/open-policy-agent/opa-istio-plugin
* https://github.com/tsandall/minimal-opa-envoy-example
* https://blog.openpolicyagent.org/envoy-external-authorization-with-opa-578213ed567c

