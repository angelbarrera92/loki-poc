# POC: Multi tenant Grafana Loki Server

## Requirements

It's needed to have a Kubernetes cluster *(k3s inside)* with an ingress controller installed and the helm's tiller deployed.
This POC starts with a `kubectl` client configured.

## Scope

This POC demonstrate how to use Grafana, Grafana loki and Promtail to push logs from a multi-tenant Kubernetes cluster having no access to the underlay host. *Example use case: `pod sandboxes`*.

## Out of scope

- `NetworkPolicies` isolating *tenants/namespaces*: Consider deploy each pod in a separate namespace. The loki server has to be in another namespace. Just open connectivity between pods with promtail to the Grafana Loki server (Proxy).

### Start

#### Configure users/tenants for Grafana Loki server

In this example, you can see the following file: [authn.yaml](authn.yaml)

```yaml
users:
  - username: Grafana
    password: L0k1
    orgid: tenant-0
  - username: Tenant1
    password: 1tnaneT
    orgid: tenant-1
```

This file is required by the [`loki-multi-tenant-proxy`](https://github.com/k8spin/loki-multi-tenant-proxy). This will be used by the proxy to forward requests to the grafana loki server authenticating every request and injecting the required loki server headers.

Create the secret in kubernetes:

```bash
$ kubectl create secret generic loki-multi-tenant-proxy-auth-config --from-file=authn.yaml=authn.yaml --dry-run -o yaml | kubectl apply -f -
secret/loki-multi-tenant-proxy-auth-config created
```

#### Deploy Grafana

This POC modifies the default [values.yaml](https://raw.githubusercontent.com/helm/charts/master/stable/grafana/values.yaml) file to create an ingress object using the awesome [`nip.io`](https://nip.io/) service. The rest configuration properties has the default values to make it simple.

```bash
$ helm upgrade --install loki-grafana --values grafana.values.yaml stable/grafana
# Truncated output
NOTES:
1. Get your 'admin' user password by running:

   kubectl get secret --namespace default loki-grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo

2. The Grafana server can be accessed via port 80 on the following DNS name from within your cluster:

   loki-grafana.default.svc.cluster.local

   From outside the cluster, the server URL(s) are:
     http://grafana.35.246.140.217.nip.io


3. Login with the password from step 1 and the username: admin
# Truncated output
```

You can get the admin password with the following command:

```bash
$ kubectl get secret --namespace default loki-grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
oD2a5r-----TRUNCATED-----8tEY2jd
```

Access from outside the cluster, the server URL(s) are: [http://grafana.35.246.140.217.nip.io](http://grafana.35.246.140.217.nip.io).

![Grafana Installed](assets/grafana-installed.png)

#### Deploy Grafana Loki

This POC uses the pull request [#909](https://github.com/grafana/loki/pull/909) to add a sidecar container to the Grafana Loki Server. This sidecar adds multi-tenant authorization to the Grafana Loki Server.

The default [values.yaml](https://raw.githubusercontent.com/grafana/loki/master/production/helm/loki/values.yaml) file has been modified to add the [grafana-loki-multitenant-proxy](https://github.com/k8spin/loki-multi-tenant-proxy) sidecar container and activated the `auth_enabled: true` configuration property.

*Partial [loki.values.yaml](loki.values.yaml):*

```yaml
config:
  auth_enabled: true

extraContainers: 
## Additional containers to be added to the loki pod.
- name: loki-multi-tenant-proxy
  image: angelbarrera92/loki-multi-tenant-proxy:dev
  args:
    - "run"
    - "--port=3101"
    - "--loki-server=http://localhost:3100"
    - "--auth-config=/etc/loki-multi-tenant-proxy/authn.yaml"
  ports:
    - name: http
      containerPort: 3101
      protocol: TCP
  volumeMounts:
    - name: loki-multi-tenant-proxy-auth-config
      mountPath: /etc/loki-multi-tenant-proxy

extraVolumes:
# Additional volumes to the loki pod.
- name: loki-multi-tenant-proxy-auth-config
  secret:
    secretName: loki-multi-tenant-proxy-auth-config

extraPorts:
# Additional ports to the loki services. Useful to expose extra container ports.
- port: 3101
  protocol: TCP
  name: http
  targetPort: http
```

```bash
$ helm upgrade --install loki --values loki.values.yaml loki/loki
$ # Check there are two containers *(loki server and the proxy)* in the loki-0 pod.
$ kubectl get pods loki-0
NAME     READY   STATUS    RESTARTS   AGE
loki-0   2/2     Running   0          9m32s
```

#### Tenant 1

You can find in [tenant-1.yaml](tenant-1.yaml) file all required resources to configure the log-recolector inside a new tenant. Those resources inclues:

- Namespace definition
- Log Recolector service Account
  - Log Recolector role bindings
- Log Recolector deployment
- Log Recolector configuration *(promtail configuration)*

The log recolector is a kubernetes `deployment` with a multi-container pod definition, it includes: [promtail](https://github.com/grafana/loki/tree/master/cmd/promtail) + [kail](https://github.com/boz/kail).

Don't worry, all is preconfigured to just apply the configuration and get it up and running and a few seconds.

The authentication and the log parser are configured in the `promtail.yaml` file *(inside a kubernetes secret)*:

```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0
client:
  url: http://loki.default.svc.cluster.local:3101/api/prom/push
  basic_auth:
    username: Tenant1
    password: 1tnaneT
scrape_configs:
  - job_name: containers
    static_configs:
      - targets:
          - localhost
        labels:
          job: containers
          __path__: /loki/logs/containers
    pipeline_stages:
    - regex:
        expression: '^(?P<namespace>.*)\/(?P<pod>.*)\[(?P<container>.*)\]: (?P<content>.*)'
    - labels:
        namespace:
        pod:
        container:
    - output:
        source: content
  - job_name: kail
    static_configs:
      - targets:
          - localhost
        labels:
          job: kail
          __path__: /loki/logs/kail
    pipeline_stages:
    - regex:
        expression: '^time="(?P<time>.*)" level=(?P<level>.*) msg="(?P<content>.*)" cmp=(?P<component>.*)'
    - labels:
        time:
        level:
        component:
    - timestamp:
        source: time
        format: RFC3339
    - output:
        source: content
```

Important notes about this configuration:

- `client.url` parameter is the proxy server, not the loki server.
- `client.basic_auth` object is the proxy server authentication configured in previous steps.
- `scrape_configs` configurations are configured to parse namespaced logs forwarding it's metadata as labels to loki server. It also parse `kail` logs.

Deploy it:

```bash
$ kubectl apply -f tenant-1.yaml 
namespace/tenant-1 created
serviceaccount/log-recolector created
rolebinding.rbac.authorization.k8s.io/log-recolector created
deployment.apps/log-recolector created
secret/tenant-1-log-recolector-config created
```

```bash
$ kubectl get pods -n tenant-1
NAME                             READY   STATUS    RESTARTS   AGE
log-recolector-5cf8d5889-jvtnf   2/2     Running   0          15s
```

Deploy an example application that writes logs to the standard output *(following best practices)*.

```bash
$ kubectl apply -f counter.yaml -n tenant-1
```

You can see it logs using `kubectl logs` command:

```bash
$ kubectl logs -f counter -n tenant-1
0: Sat Nov 23 13:51:17 UTC 2019
1: Sat Nov 23 13:51:19 UTC 2019
2: Sat Nov 23 13:51:20 UTC 2019
3: Sat Nov 23 13:51:21 UTC 2019
4: Sat Nov 23 13:51:22 UTC 2019
5: Sat Nov 23 13:51:23 UTC 2019
```

It prints only the current date every second.

## Grafana configuration

Enter into the Grafana Server as admin. Create a new Organization: `tenant-0`.

![Grafana Create Org](assets/grafana-create-org.png)

Create a new user *(could be different of the grafana loki proxy)*:

![Grafana Create User](assets/grafana-create-user.png)

Modify the created user to delete from the main organization adding to the created organization as **editor**.

![Grafana Modify User](assets/grafana-modify-user.png)

Then move to the recently created organization:

![Grafana Switch User](assets/grafana-switch-org.png)

Create the loki datasource into the created organization:

![Grafana Create Datasource](assets/grafana-create-loki-datasource.png)

Now, logout as admin and login as the new user created. Enter explore tab:

![Grafana Explore](assets/grafana-loki-explore.png)

And you've got the container logs pod.
