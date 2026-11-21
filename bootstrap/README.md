# Bootstrap (one-time, manual)

Prereqs: Talos cluster, Longhorn (`longhorn` StorageClass), Traefik with Gateway API
enabled (GatewayClass `traefik`), and a Forgejo repo mirroring this one.

```sh
kubectl apply -f bootstrap/namespace.yaml

# Secrets are never committed; create them out-of-band (or add SOPS later).
kubectl -n flux-system create secret generic forgejo-auth \
  --from-literal=username=<forgejo-user> --from-literal=password=<forgejo-token>
kubectl -n flux-system create secret generic zitadel-oidc \
  --from-literal=clientID=<client-id> --from-literal=clientSecret=<client-secret>
kubectl -n flux-system create secret generic cloudflared-token \
  --from-literal=token=<tunnel-token>

helm install flux-operator oci://ghcr.io/controlplaneio-fluxcd/charts/flux-operator \
  -n flux-system
kubectl apply -f clusters/homelab/flux-system/flux-instance.yaml
```

After the first sync Flux manages itself (operator HelmRelease, Gateway, tunnel, UI).

Placeholders to edit: Forgejo URL (`flux-instance.yaml`), Zitadel issuer
(`REPLACE_ME` in `flux-operator.yaml`).

- Zitadel redirect URI: `https://flux.joeckr.dev/oauth2/callback`
- Cloudflare tunnel public hostname `flux.joeckr.dev` -> Traefik web entrypoint
  service (e.g. `http://traefik.traefik.svc:80`).
- The Flux Operator web UI OIDC value schema should be verified against the
  operator version you install.
