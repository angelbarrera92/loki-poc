



kubectl create secret generic reverse-proxy-auth-config --from-file=authn.yaml=reverse-proxy-conf.yaml --dry-run -o yaml | kubectl apply -f -

## Grafana configuration

As an admin in the main organization i can created as many loki datasources as we need (debuging tasks).

Once a new user appears, the admin has to create its organization, then create the grafana user. Removes from the main org and set to the org created before as `editor`.

Then, as admin create the loki datasource into the users organization (dont forget to check the current admin organization).

Name: Loki
Url: http://loki:11811 (the reverse proxy container endpoint). The port 3100 shouldnt work without auth. (Data source connected, but no labels received. Verify that Loki and Promtail is configured properly.
)basic-auth: With the user credentials (the reverse proxy credentials).
