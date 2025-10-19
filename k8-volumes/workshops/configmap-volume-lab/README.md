# Kubernetes ConfigMaps & Volumes Workshop

Hey there, welcome in! I’ll play the role of your screen-recording guide as we build up a very down-to-earth web app on Kubernetes and keep layering on ConfigMaps, volumes, and secrets until we wrap things up inside a tiny Helm chart. Think of this as a cooking show where we start with the plain ingredients (a vanilla Nginx pod) and, step by step, we season it with the right configuration toppings. I’ll call out the “why” behind each move, surface the gotchas, and pause for the questions I usually hear from learners following along.

All of the manifests I reference live under `k8-volumes/workshops/configmap-volume-lab`. Each numbered folder represents the state of the world at that step, so you can diff your way through if you want to peek ahead.

---

## 0. Getting Ready (a quick mise en place)

**Mental model:** Kubernetes is basically a very opinionated logistics manager. ConfigMaps, volumes, and secrets are different delivery trucks that ferry configuration into your containers. We’ll swap trucks as we scale up the recipe.

- Cluster: any Kubernetes 1.24+ cluster works (Kind, Minikube, Docker Desktop, EKS, AKS… you name it).
- Tooling: `kubectl`, `helm`, and access to port-forward (no load balancer needed).
- Namespace: Using a dedicated namespace keeps the playground tidy.

Apply the namespace and a reusable service up front:

```bash
kubectl apply -f manifests/00-prereqs/namespace.yaml
kubectl apply -f manifests/01-base-deployment/service.yaml
```

> You might be wondering why we create the Service so early. Services are “sticky business cards” for pods. Having it around means every iteration stays reachable at the same DNS name (`nginx-lab.default.svc`), which makes verification smoother.

---

## 1. Base Deployment (plain Nginx, no frills)

Let’s start with the raw deployment to make sure nothing exotic is lurking. Check out `manifests/01-base-deployment/deployment.yaml`.

```bash
kubectl apply -f manifests/01-base-deployment/deployment.yaml
kubectl get pods -n configmap-lab
kubectl port-forward -n configmap-lab svc/nginx-lab 8080:80
```

Open `http://localhost:8080` and you’ll get the default “Welcome to nginx!” page. That’s our control group.

**Under the hood:** The Deployment creates a ReplicaSet, the ReplicaSet spawns a Pod, and the kubelet on whichever node got chosen pulls the `nginx:1.25-alpine` image. No volumes yet, so everything you see is baked into the container image.

> Learner question: *“How does Kubernetes decide where to run this pod?”* The scheduler looks at resource requests, node taints/tolerations, and node health. With a single-node cluster, the choice is easy; in production you’d see it weigh CPU/memory availability and placement rules.

---

## 2. ConfigMaps as Environment Variables

Reality check: plenty of teams start by putting configuration values into environment variables. Open `manifests/02-env-configmap/configmap.yaml` and `deployment.yaml`.

Key bits:
- We create a ConfigMap called `web-ui-config`.
- The Deployment imports it via `envFrom`.
- The container command overrides Nginx’s default entrypoint and echoes the env into a simple HTML file before booting Nginx. That small tweak lets us “see” the env vars in the UI.

Apply the new ConfigMap and the tweaked Deployment:

```bash
kubectl apply -f manifests/02-env-configmap/configmap.yaml
kubectl apply -f manifests/02-env-configmap/deployment.yaml
kubectl rollout status deployment/nginx-lab -n configmap-lab
```

Refresh the page. You should see a friendlier Nginx welcome banner showing the `WELCOME_MESSAGE` from the ConfigMap.

**Under the hood:** The kubelet on the node materializes the ConfigMap as downward API data. When the pod is launched, the container runtime literally injects those key/value pairs into the process environment. If you exec into the pod (`kubectl exec -it deploy/nginx-lab -n configmap-lab -- env | grep WELCOME`), it shows up just like any other env var.

> You might be wondering, “What if I update the ConfigMap now?” Environment variables are baked into the container at start time, so the pod won’t automatically pick up changes. You’d need to restart the pod or pair this with something like a sidecar that rewrites files. That’s why the next step leans on volume mounts.

---

## 3. ConfigMaps as Volumes (hello, hot reload)

This is where ConfigMaps shine for static assets or config files. Peek at `manifests/03-volume-configmap/configmap.yaml` and the associated Deployment.

What changes:
- The ConfigMap now contains an entire `index.html`.
- The Deployment mounts the ConfigMap at `/usr/share/nginx/html`, which is the directory Nginx reads from.
- We keep the Service from the base setup, so nothing else changes externally.

Apply the update:

```bash
kubectl apply -f manifests/03-volume-configmap/configmap.yaml
kubectl apply -f manifests/03-volume-configmap/deployment.yaml
kubectl rollout status deployment/nginx-lab -n configmap-lab
```

Port-forward again if you stopped it: `kubectl port-forward -n configmap-lab svc/nginx-lab 8080:80`.

Now, open `http://localhost:8080`. You’ll see a richer page rendered entirely from the ConfigMap volume. The JavaScript file (`lab-data.js`) also lives in that ConfigMap and populates the headline, supporting text, and the little call-to-action bubble. We still keep the environment variables from earlier so you can `kubectl exec` in and spot them, but the UI itself comes straight from the mounted files.

**Hot reload demo:**
1. Edit the `headline` string inside `manifests/03-volume-configmap/configmap.yaml` (look for `lab-data.js`).
2. Apply the ConfigMap again.
3. Wait ~30 seconds (the kubelet syncs every few seconds).
4. Refresh the browser—boom, instant update without touching the pod.

Behind the scenes, Kubernetes projects the ConfigMap into the pod’s filesystem using a tmpfs and a set of atomic symlinks. When the ConfigMap changes, the kubelet swaps the symlink target. Because Nginx reads the file fresh on each request, it sees the new contents the moment the file changes.

> Common misconception: *“Do I need to restart Nginx for the new config?”* For static assets, no. But if you mounted `/etc/nginx/nginx.conf`, Nginx would need a signal (`nginx -s reload`) because it only parses that file at startup.

---

## 4. Secrets (same idea, different delivery truck)

Now let’s mix in a Secret. We’ll inject an API token as an environment variable *and* surface it in the HTML for demonstration (never do that in production—it’s purely for inspection).

Files to inspect:
- `manifests/04-secret-injection/secret.yaml`
- `manifests/04-secret-injection/deployment.yaml`

Apply them (all three resources get an upgrade in this step):

```bash
kubectl apply -f manifests/04-secret-injection/configmap.yaml
kubectl apply -f manifests/04-secret-injection/secret.yaml
kubectl apply -f manifests/04-secret-injection/deployment.yaml
kubectl rollout status deployment/nginx-lab -n configmap-lab
```

Refresh the UI. You’ll see a masked version of the secret plus a reminder about Base64 encoding. Want to double-check the actual value? Run `kubectl exec deploy/nginx-lab -n configmap-lab -- printenv API_TOKEN` or `cat /runtime/api-token.txt`.

**Under the hood:** Kubernetes stores Secrets in etcd just like ConfigMaps, but they’re flagged for stricter RBAC and (optionally) encryption at rest. When mounted as env vars, they behave exactly like ConfigMaps: no hot reload. When mounted as volumes, they follow the same symlink pattern with updated data rolling in without restarts.

> You might be wondering, “Why Base64?” It’s not encryption—it’s just a convenient way to stick binary blobs into YAML. Don’t treat it as security. Use RBAC, encrypt the etcd store, and consider external secret managers (Secrets Store CSI Driver, External Secrets Operator) when the stakes are higher.

---

## 5. From Manifests to Helm

Time to wrap this into a reusable Helm chart so teams can stamp it out consistently. The chart lives under `helm/configmap-volume-demo`.

Structure:
- `Chart.yaml` — metadata.
- `values.yaml` — defaults for image, ConfigMap data, secrets, port, etc.
- `templates/` — parameterized versions of the Deployment, Service, ConfigMap, and Secret.

Try a dry run:

```bash
helm template nginx-lab helm/configmap-volume-demo \
  --namespace configmap-lab \
  --set image.tag=1.25-alpine \
  --set config.headline="Rendered by Helm!"
```

Install it (after optionally cleaning up previous deployments):

```bash
kubectl delete deployment nginx-lab -n configmap-lab --ignore-not-found
helm upgrade --install nginx-lab helm/configmap-volume-demo \
  --namespace configmap-lab \
  --set service.type=ClusterIP
```

Check the rendered resources with `helm get manifest nginx-lab -n configmap-lab`. You’ll notice the templates mirror the final manifest versions we walked through manually—Helm simply templatizes the knobs you’re likely to tweak.

> Common pitfall: forgetting that Helm stores release state in the same namespace. If you delete the namespace without telling Helm, the release metadata disappears and you’ll need `--replace` the next time you install.

---

## Where to Go Next

- Experiment with the `values.yaml` toggles—flip the secret volume mount on/off, point the ConfigMap to a different directory, or scale the replicas.
- Swap Nginx for your own container image and keep the volume wiring the same.
- Curious about configuration drift? I’d search for “Kubernetes configmap rollout restart best practice” to see patterns like `kubectl rollout restart` or hashed annotations.
- Want automated reloads for files like `nginx.conf`? Check out sidecars such as [stakater/Reloader] or tools like `configmap-reload`.

Wrap up thought: ConfigMaps and Secrets are just different wrappers on top of the same kubelet projection machinery. Once you understand how the files land on disk and how pods read them, everything from reloading behavior to templating with Helm clicks into place.

Happy shipping! If any step left you scratching your head, jot the question down—that curiosity is how you unlock the next layer. :)
