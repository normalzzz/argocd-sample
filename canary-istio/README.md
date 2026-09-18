# Canary deployment with Istio

This chart uses Argo Rollouts and an Istio `VirtualService` for exact weighted
canary traffic. The stable and canary images may contain identical application
content: Istio adds `X-Rollout-Track: stable` or `X-Rollout-Track: canary` to
each response after choosing the destination.

## Prerequisites

- Argo Rollouts is installed.
- Istio and its ingress gateway are installed. The default gateway selector is
  `istio: ingressgateway`; override `istio.gateway.selector` if the installation
  uses a different label.

## Deploy and start a rollout

Create the application (change the repository URL as needed):

```sh
argocd app create canary-istio \
  --repo https://github.com/argoproj/argocd-example-apps \
  --path canary-istio \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default
argocd app sync canary-istio
```

Change the tag from `v5` to `v6` to create a new ReplicaSet:

```sh
argocd app set canary-istio -p image.tag=v6
argocd app sync canary-istio
```

The configured steps set canary traffic to 20%, run the smoke test, move it to
50% and wait for manual promotion, then move it to 80%.

## Observe the real request distribution

Obtain the ingress gateway address. The following example assumes the standard
Istio service name and namespace:

```sh
export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
```

If the load balancer reports a hostname, use
`{.status.loadBalancer.ingress[0].hostname}` instead. Send independent requests
and count the response marker:

```sh
for request in $(seq 1 100); do
  curl -sSI -H 'Host: canary.example.com' "http://${INGRESS_HOST}/" \
    | awk -F ': ' 'tolower($1) == "x-rollout-track" {print tolower($2)}' \
    | tr -d '\r'
done | sort | uniq -c
```

At `setWeight: 20`, a sufficiently large sample should be close to:

```text
80 stable
20 canary
```

The exact count varies because each request is randomly routed. Inspect the
desired weights at any time with:

```sh
kubectl get virtualservice canary-istio-helm-guestbook \
  -o jsonpath='{range .spec.http[?(@.name=="primary")].route[*]}{.destination.host}{"="}{.weight}{"%\n"}{end}'
```
