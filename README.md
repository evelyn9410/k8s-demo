# k8s-demo

I wanted to see if `mirrord` actually saves time when debugging stuff running in Kubernetes. Spoiler: it does. Here's a tiny Express app I used to test it out on GKE.

## What's the pain

Every time I change something and want to see it in the cluster, the loop is:

code change -> rebuild image -> push -> deploy -> wait for rollout -> check logs -> repeat

Even for this hello-world app that takes ~2 min per cycle. For anything real (bigger images, more replicas, multiple services) it's way worse and more expensive. 

mirrord lets you skip all that by running your code locally but with the cluster's network, env vars, and DNS. So you just... edit and re-run.

## What's in here

- `app/` - Express service with `/`, `/healthz`, `/readyz`, `/downstream-check`
- `k8s/` - Deployment + Service manifests
- `k8s/networkpolicy-deny-egress.yaml` - NetworkPolicy I used to simulate a cluster-only failure
- `.mirrord/mirrord.json` - mirrord config that worked for me

## Setup

### What you need installed

- `gcloud`, `kubectl`, `gke-gcloud-auth-plugin`
- `docker` with `buildx`
- `node`
- `mirrord`

```bash
gcloud auth login
gcloud config set project YOUR_GCP_PROJECT_ID
export USE_GKE_GCLOUD_AUTH_PLUGIN=True
```

### Cluster (has to be Standard, not Autopilot)

I burned time on this: Autopilot blocks the Linux capabilities mirrord's agent needs. Use Standard.

```bash
gcloud services enable container.googleapis.com

gcloud container clusters create k8s-demo-standard \
  --region us-central1 \
  --num-nodes 1 \
  --disk-type=pd-standard \
  --disk-size=50

gcloud container clusters get-credentials k8s-demo-standard \
  --region us-central1
```

### Build and push the image

I used Artifact Registry. `gcr.io` gave me trouble in a restricted org, so heads up on that.

```bash
export PROJECT_ID=YOUR_GCP_PROJECT_ID
export REGION=us-central1
export REPO=k8s-demo

# one-time repo setup
gcloud services enable artifactregistry.googleapis.com
gcloud artifacts repositories create $REPO \
  --repository-format=docker \
  --location=$REGION || true

gcloud auth configure-docker ${REGION}-docker.pkg.dev --quiet

export IMAGE=${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPO}/hello-world:$(date +%Y%m%d-%H%M%S)
docker buildx build --platform linux/amd64 -t "$IMAGE" --push ./app
```

### Deploy

```bash
kubectl create namespace k8s-demo || true
sed "s|IMAGE_PLACEHOLDER|$IMAGE|g" k8s/deployment.yaml | kubectl -n k8s-demo apply -f -
kubectl -n k8s-demo apply -f k8s/service.yaml
kubectl -n k8s-demo rollout status deployment/hello-world
```

Sanity check:

```bash
kubectl -n k8s-demo get svc hello-world
curl http://<EXTERNAL_IP>/
curl http://<EXTERNAL_IP>/healthz
curl http://<EXTERNAL_IP>/readyz
```

## Trying mirrord (I used v3.186)

First make sure things are wired up:

```bash
mirrord --version
kubectl config current-context
kubectl auth can-i get pods -n k8s-demo
kubectl auth can-i create pods -n k8s-demo
```

Here's the config I landed on (`.mirrord/mirrord.json`):

```json
{
  "target": "deployment/hello-world",
  "feature": {
    "network": {
      "incoming": "mirror",
      "outgoing": true,
      "dns": { "enabled": true }
    },
    "env": { "include": "^.*$" }
  }
}
```

Then run it:

```bash
mirrord exec -n k8s-demo --config-file .mirrord/mirrord.json -- npm --prefix app start
```

Or skip the config file entirely:

```bash
mirrord exec -t deployment/hello-world -n k8s-demo -- npm --prefix app start
```

If you want your local process to actually handle the requests (not just see copies), add `--steal`:

```bash
mirrord exec -t deployment/hello-world -n k8s-demo --steal -- npm --prefix app start
```

## Things I tried debugging with it

### Downstream calls that fail only in the cluster

This was the most satisfying one. The `/downstream-check` endpoint works fine locally but you can make it fail in-cluster by applying a NetworkPolicy:

```bash
# works locally
npm --prefix app start
curl http://localhost:8080/downstream-check

# break it in the cluster
kubectl -n k8s-demo apply -f k8s/networkpolicy-deny-egress.yaml
curl http://<EXTERNAL_IP>/downstream-check   # fails

# now run locally but through the cluster's network
mirrord exec -t deployment/hello-world -n k8s-demo -- npm --prefix app start
curl http://<EXTERNAL_IP>/downstream-check   # fails locally too — you can actually see why

# clean up
kubectl -n k8s-demo delete -f k8s/networkpolicy-deny-egress.yaml
```

Being able to reproduce a cluster-only networking issue on my local machine with a debugger attached was really nice.

### Seeing live cluster traffic locally

```bash
mirrord exec -t deployment/hello-world -n k8s-demo -- npm --prefix app start
```

Then hit the external IP from another terminal — the request shows up in your local process. Set breakpoints, add console.logs, whatever. Use `--steal` if you want your local code to actually respond instead of the remote pod.

### Targeting a specific pod

When you have multiple replicas and one is acting up:

```bash
kubectl -n k8s-demo get pods -l app=hello-world -o wide
kubectl -n k8s-demo logs POD_NAME --tail=200

mirrord exec -t pod/POD_NAME/container/hello-world -n k8s-demo --steal -- npm --prefix app start
```

You can attach to exactly the pod you're suspicious of. No need to scale down to 1 replica.

## Gotchas I ran into

- **Autopilot won't work.** The mirrord agent needs capabilities Autopilot doesn't allow. Spent a while on this before switching to Standard.
- **`target_namespace` in the config blows up on v3.186.0** with an `unknown field` parse error. Just pass `-n k8s-demo` on the CLI instead.
  - I built a knowledge base skill that tracks repo changes and catches stuff like this — renamed/removed config properties across versions — so I don't get blindsided again. See: [claude knowledge base skill](https://github.com/Evelyn5410/github-knowledge-base).
- **`service/hello-world` as a target didn't work** — apparently that needs the mirrord operator, which I didn't set up.
- **Changes aren't hot-reloaded.** If you edit code, you need to restart `mirrord exec`. Node doesn't auto-reload here.
- **`gcr.io` can be flaky** in orgs with restrictions. Artifact Registry worked fine.
- **mirrord config schema changes between versions.** What works on one version might not parse on another. Check their docs for your version.
- **This won't help with low-level issues** like kernel bugs or node/runtime corruption. It's for app-level debugging.

## Cleanup

```bash
kubectl -n k8s-demo delete svc hello-world
kubectl -n k8s-demo delete deployment hello-world
kubectl -n k8s-demo delete networkpolicy hello-world-deny-egress --ignore-not-found

gcloud container clusters delete k8s-demo-standard --region us-central1
```
