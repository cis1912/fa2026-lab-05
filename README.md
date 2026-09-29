# Lab 05: Helm

In `fa26-kube-lab`, you hand-wrote a `Deployment` and a `Service` for the 2048 game and applied them with `kubectl`. Fine for one copy of one app. Real teams run **dev** and **prod** side by side, each with its own replica count and config, defined by nearly-identical YAML — and every change has to be repeated correctly in every copy.

**Helm** is a package manager for Kubernetes: write your manifests once, as a parameterized **chart**, and generate each environment's YAML from a small set of values instead of maintaining full copies by hand.

This lab is designed to demonstrate some inefficiencies of hand-writing Kubernetes manifests! Watch as Helm removes the need for hand-curated YAML for each environment.

## Setup

You'll need `helm`. Check the [Helm installation guide](https://helm.sh/docs/intro/install) for instructions.

## Part 1: The Pain, By Hand

`manual/dev/` and `manual/prod/` each have a `deployment.yaml` + `service.yaml` for the 2048 game — same app, different `replicas` and memory `limit`. Apply both:

```bash
kubectl create namespace dev
kubectl create namespace prod
kubectl apply -f manual/dev -n dev
kubectl apply -f manual/prod -n prod
```

Now ops gets paged: 2048 pods are getting OOMKilled under load. Fix: **double the memory limit in every environment.** Do it by hand — edit `manual/dev/deployment.yaml` (`128Mi` → `256Mi`) and `manual/prod/deployment.yaml` (`512Mi` → `1024Mi`), then re-apply both. Two files for one logical change — now imagine ten services instead of one.

## Part 2: Enter Helm

`2048/` is a starter chart: `Chart.yaml` (metadata), `values.yaml` (default config), `templates/` (manifests written as Go templates over those values). `2048/values.yaml` is already filled in and deliberately matches `manual/dev` exactly — treat it as "dev's config." `2048/templates/deployment.yaml` is the same manifest with the three env-specific bits ripped out and marked `<FILL_IN>`, each with a `# TODO` naming the value from `values.yaml` that belongs there.

Fill in the three blanks using `.Values.<key>` syntax, then check your work without touching the cluster:

```bash
helm lint ./2048
helm template check ./2048
```

Compare the output to `manual/dev/deployment.yaml` — the templated fields should match.

### How values get into a template

`values.yaml` is just nested YAML — `.Values.<path>` walks it the same way you'd index the YAML by hand. Given

```yaml
image:
  repository: ghcr.io/cis1912/2048
  tag: latest
```

`.Values.image.repository` and `.Values.image.tag` are how the two leaves get referenced from a template — there's no separate "values language," it's the same dot-path you'd use to describe the key in conversation. Whatever value ends up there gets substituted in as plain text when the template renders, which is why string values that need to stay quoted in the output (like `{{ .Values.env.appEnv | quote }}` above) go through the `quote` function — Helm doesn't know your YAML expects a string unless you tell it.

## Part 3: One Chart, Two Environments

Diff `manual/prod/deployment.yaml` against `manual/dev/deployment.yaml` — the only differences are `replicas` and the memory limit. Create `2048/values-prod.yaml` overriding just those two keys (plus `env.appEnv: prod`). It should be a handful of lines, not a full manifest.

`values-prod.yaml` doesn't need to repeat everything in `values.yaml` — `-f` **merges** the file you pass on top of the chart's defaults, key by key, so an override file only lists what's different for that environment. Everything you leave out (like `image.repository`) falls back to `values.yaml`. Nesting has to match, though: to override just `resources.limits.memory` you still write out the parent keys down to it —

```yaml
resources:
  limits:
    memory: 1024Mi
```

not a flattened `resources.limits.memory: 1024Mi`.

You can also layer more than one `-f` (later files win) or override a single value inline without a file at all via `--set key=value`, e.g. `--set resources.limits.memory=1024Mi` — handy for a one-off test, but `-f` is what you want for anything you intend to keep around. If you ever forget what a chart's tunable keys are, `helm show values ./2048` prints `values.yaml` back out.

Delete the manual deployments and install both as Helm releases instead:

```bash
kubectl delete -f manual/dev -n dev
kubectl delete -f manual/prod -n prod

helm install dev-2048 ./2048 -n dev
helm install prod-2048 ./2048 -f 2048/values-prod.yaml -n prod

helm list -A
```

## Part 4: Same Incident, Easy Mode

Ops pages again — double the memory limit once more. This time, edit two numbers instead of two manifests: bump `resources.limits.memory` in `2048/values.yaml` and in `2048/values-prod.yaml`, then:

```bash
helm upgrade dev-2048 ./2048 -n dev
helm upgrade prod-2048 ./2048 -f 2048/values-prod.yaml -n prod
```

Same real change as Part 1 — this time it's two small, readable values files instead of two full manifests, and `helm upgrade` does the applying.

## Part 5: Rollback

Every `helm upgrade` creates a new numbered release revision:

```bash
helm history prod-2048 -n prod
```

With hand-written YAML, undoing a change means finding an old copy of the file — if you kept one. With Helm:

```bash
helm rollback prod-2048 1 -n prod
kubectl get deployment prod-2048 -n prod -o jsonpath='{.spec.template.spec.containers[0].resources.limits.memory}'
```

That should print the original `512Mi`.

## Tips

- `helm uninstall <release> -n <namespace>` tears down a release.
- `helm get values <release> -n <namespace>` shows what a running release was actually installed with.
- Precedence, lowest to highest: `values.yaml` defaults → each `-f` file, in the order you pass it → `--set`. So `helm upgrade 2048-prod ./2048 -f values-prod.yaml --set replicaCount=5 -n prod` would install with 5 replicas regardless of what either YAML file says.
- `kind delete cluster --name cis1912-helm` tears down the whole cluster when you're done.

## Bonus (Optional)

- Install a public chart (e.g. `bitnami/nginx`) and compare _consuming_ someone else's chart to _authoring_ your own.
- Add a third environment (`staging`) yourself — it should only take one more small values file.
