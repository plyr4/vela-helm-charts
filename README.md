# Running Vela in K8s using kind

1. Install [`kind`](https://kind.sigs.k8s.io/docs/user/quick-start/).

2. Install [`kubectl`](https://kubernetes.io/docs/tasks/tools/).

3. Create a local kubernetes cluster using `kind`.

```bash
# kind create cluster
```

4. Before moving on, confirm that you have a kubernetes cluster running, and that you can interact with it using `kubectl`.

```bash
# kubectl get pods
```

> You might need to generate a local kubeconfig using `kind`, if creating a cluster doesn't do it for you.

5. Convert the docker-compose.yml into kubernetes yaml using [`kompose`](https://kompose.io/getting-started/)
**_OR_**
Clone the converted files from github.com/plyr4/vela-helm-charts under the `services/` directory.

I used `kompose` to generate those files using the server's `docker-compose.yml` then cleaned a bunch of stuff up, like reordering env variables to match the way we see them in the compose yaml.

So... it will save you time to just use the code found in github.com/plyr4/vela-helm-charts.

Make sure the namespaces in all of the yaml files are pointing to the namespace you have running locally through `kind`. For me, it is usually just `default`, unless you name it something different during creation. (I've tested using `vela` and it still works fine.)

6. Update the values in the configMap that say `<FILL_ME>`, also point the SCM to GHE if necessary.

7. Deploy the core stack (server, ui, database, queue, vault) using `kubectl`.
```bash
# kubectl -f apply services/
```

Before moving on, use `kubectl` to ensure that all of the deployments/services booted up fine. There should be a pod for each part of the `docker-compose` stack, excluding the worker, we will boot that up separately.

8. Deploy the worker using `kubectl`.

```bash
# kubectl -f apply services/worker/
```

9. Expose the ui/server to access them over localhost using `kubectl port-forward`.

First grab the `<POD_ID>` for the ui and server using `kubectl get pods` then run the following command:


Expose the server pod on ports `8080:8080`.

```bash
# kubectl port-forward server-<POD_ID>  8080:8080
```

Expose the ui pod on ports `8888:80` (I think the UI has to be routed to port `80` to serve HTTP).

```bash
# kubectl port-forward ui-<POD_ID>  8888:80 
```

I started using this one-liner as a shortcut:

```bash
# kubectl port-forward $(kubectl get pods | grep ui | awk '{print $1}') 8888:80
```

## Running the worker service on M1

#### THIS SECTION IS IN-PROGRESS - YOUR MILEAGE MAY VARY DEPENDING ON YOUR ARCHITECTURE

If you have an m1 mac, when the worker tries to spin up step containers you will likely see this error in the `clone` step logs:

```
runtime: failed to create new OS thread (have 2 already; errno=22)
fatal error: runtime.newosproc
```

This is due to an architecture mismatch between your M1/arm64 host machine and the underlying kubernetes running in docker.

We need a custom worker image that allows us to provide an override to the kubernetes pause image: `registry.k8s.io/pause:3.6`. This pause image is vela uses to boot up the build/step container pods.

When running that image in place of the worker image, you MUST provide the environment variable that overrides the pause image:
```
VELA_K8S_PAUSE_IMAGE: registry.k8s.io/pause:3.6
```
