# Kubernetes Interview Lab — Troubleshooting & Security

A small set of Kubernetes workloads deployed to a local cluster (minikube). Two parts
that share one story — a model-serving backend being moved onto the platform:

1. **Part 1** — the Service is misbehaving; investigate and fix it.
2. **Part 2** — harden the runtime: RBAC for a discovery agent *and* a config/secrets
   audit of the same workload.

Both parts are hands-on in the `model-runtime-lab` and `config-lab` namespaces.

## Prerequisites

- A running local cluster: `minikube start`
- `kubectl` configured for that cluster

Images used (`nginx:alpine`, `busybox:1.36`, `curlimages/curl:latest`, `bitnami/kubectl`)
are pulled from Docker Hub; no image loading required. No special CNI needed.

---

## Part 1 — Fix the failing Service

### Background

`model-runtime` serves a model behind a Kubernetes `Service`. A teammate recently rolled
out `model-canary`, a new model version. Since then, monitoring shows that **roughly a
third of requests to `model-runtime` fail** — but the rest succeed, and every pod reports
as healthy.

You have a `toolbox` pod (with `curl`) to investigate from.

```bash
kubectl apply -f model-runtime-canary-lab.yaml
kubectl get pods -n model-runtime-lab -w   # wait until all pods are Running/Ready
```

### Warm-up — a pod that won't stay up

Before the Service investigation: one pod in this namespace, `model-loader`, is **not
staying Running**. Take a look, work out *why*, and get it healthy.

1. Find the pod that isn't Running and identify the reason it keeps restarting.
2. Explain what that state means and why it's happening here.
3. Apply a fix so the pod runs stably.

### Reproduce the problem

From the `toolbox` pod, send a batch of requests to the Service and count the results:

```bash
kubectl exec -it -n model-runtime-lab toolbox -- sh -c \
  'for i in $(seq 30); do curl -s -o /dev/null -m 2 -w "%{http_code}\n" http://model-runtime/; done | sort | uniq -c'
```

You should see a mix of successful (`200`) and failed (`000`) responses.

### Your task

1. Investigate why only *some* requests fail while all pods appear healthy.
2. Identify the root cause.
3. Apply a fix so that the `model-runtime` Service returns **100% successful** responses.
4. Be ready to explain what went wrong and how your fix addresses it.

You may use any `kubectl` commands you like. Explain your reasoning as you go.

---

## Part 2 — Harden the runtime

This part has two independent exercises that both live under the "securing the platform
layer" umbrella. Do them in order — Part 2a feeds into Part 2b.

---

### Part 2a — Give the discovery agent least-privilege RBAC

#### Background

We want an agent that continuously discovers the model backends (the kind of tool that
would have caught the Part 1 bug). Deploy it:

```bash
kubectl apply -f endpoint-inspector.yaml
kubectl logs -n model-runtime-lab endpoint-inspector -f
```

The `endpoint-inspector` pod runs `kubectl` as the `endpoint-inspector` ServiceAccount and
tries to read Services, Endpoints, and Pods in the namespace. Right now every call fails
with a `Forbidden` error — the ServiceAccount has no permissions.

#### What we've given you

To keep the focus on your *reasoning* rather than YAML boilerplate, the Role and
RoleBinding are already scaffolded in **`rbac-scaffold.yaml`**. The whole structure is
there — **except the Role's `verbs:` field, which is left blank** for you to complete.

#### Your task

1. **Fill in the `verbs:`** in `rbac-scaffold.yaml` so the agent can **read** Services,
   Endpoints, and Pods in the `model-runtime-lab` namespace — and nothing more (least
   privilege, read-only). Apply it and confirm the agent's logs stop showing errors:

   ```bash
   kubectl apply -f rbac-scaffold.yaml
   kubectl logs -n model-runtime-lab endpoint-inspector -f
   ```

2. **Walk us through every field in the scaffold** — this is the main thing we're
   assessing, since the structure is given. Explain, in your own words:
   - the `apiGroups: [""]` — what the empty string means and why these resources live there;
   - the `resources` list — why exactly pods/services/endpoints, and what `["*"]` would cost;
   - the `verbs` you chose — why each specific verb is needed for a read-only discovery
     agent, the distinction between read verbs and when each matters, and why no
     write/delete/create verbs belong here;
   - `Role` vs `ClusterRole` — why a namespaced Role is the right call and what the
     `namespace` field bounds;
   - the `RoleBinding` — how `subjects` and `roleRef` wire the ServiceAccount to the Role,
     and why the names and namespaces have to line up.

3. **Prove the boundary holds.** Show us the agent can do exactly what it needs and
   *cannot* do more — without redeploying. Demonstrate both an allowed action and a denied
   one (e.g. a write verb, or a read in another namespace):

   ```bash
   # Can-i helper (fill in verb/resource/namespace):
   kubectl auth can-i <verb> <resource> \
     --as=system:serviceaccount:model-runtime-lab:endpoint-inspector -n model-runtime-lab
   ```

---

### Part 2b — Audit the config & secrets consumption

#### Background

A companion workload — the backend processor the agent is watching — is deployed alongside
the model runtime. Apply it now:

```bash
kubectl apply -f config-management-scaffold.yaml
kubectl get pods -n config-lab -w   # wait until Running
```

The pod starts and runs. The problems are **not a crash** — they are bad practices that
create risk in production. There are **two independent problems**: one is a wrong object
type choice; the other is an insecure consumption pattern for a credential.

#### Investigate

Inspect how configuration actually reaches the running container:

```bash
# All env vars the container sees:
kubectl exec -n config-lab deploy/orders-processor -- env | sort

# What /proc/1/environ exposes — everything the kernel sees for PID 1:
kubectl exec -n config-lab deploy/orders-processor -- \
  sh -c 'cat /proc/1/environ | tr "\0" "\n" | grep -E "DB_|SIGNING"'
```

#### Your task

1. **Identify both problems.** For each one explain:
   - what the current pattern is and what is wrong with it;
   - the concrete risk or failure mode it creates in production.

2. **Fix both problems** in the manifest and re-apply. The app's behaviour (what config
   values it receives) must stay the same; only the *how* changes. Here is the shape of
   each fix — fill in the blanks:

   **Problem 1 — wrong object type.** The signing key is sensitive, so it belongs in a
   `Secret`, not the `ConfigMap`. Move it:

   ```yaml
   # In the Secret (db-credentials, or a new Secret) add the key:
   stringData:
     username: "orders_app"
     password: "s3cr3t-orders-db"
     signing-key: "dummy-signing-key"   # <-- moved out of the ConfigMap

   # ...and delete SIGNING_KEY from the ConfigMap's data: block.
   ```

   **Problem 2 — insecure consumption.** Stop injecting the credential(s) as **env vars**;
   mount the Secret as a **file** via a volume instead. The skeleton:

   ```yaml
   spec:
     template:
       spec:
         containers:
           - name: app
             # Remove the DB_USER / DB_PASSWORD (and SIGNING_KEY) `env:` entries that
             # use secretKeyRef, and mount the secret instead:
             volumeMounts:
               - name: db-creds
                 mountPath: /etc/secrets/db      # app reads the files from here
                 readOnly: true
         volumes:
           - name: db-creds
             secret:
               secretName: db-credentials
               defaultMode: 0400                 # owner read-only
   ```

   Keep the non-sensitive ConfigMap env vars (`LOG_LEVEL`, `DB_HOST`, …) exactly as they
   are. Re-apply and verify the sensitive values no longer appear in
   `kubectl exec ... -- env`.

3. **Walk us through your changes**, for each fix covering:
   - *why* the corrected pattern is better — not just that it is;
   - what attack surface or failure mode is now closed;
   - operational implications (rotation, restart requirements, file permissions).

After your fix, demonstrate — or explain — how a credential can be rotated without
restarting the pod, and what the kubelet does under the hood to make that work.


