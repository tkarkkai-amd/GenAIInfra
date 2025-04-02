# Deploying AgentQnA with kubernetes/helm using AMD GPUs

This is a walkthrough of deploying the AgentQnA example with kubernetes/helm that uses AMD GPUs. It largely follows the README in GenAIInfra: https://github.com/opea-project/GenAIInfra/blob/main/helm-charts/agentqna/README.md. It uses a PVC for storage.

## TODO

- UI not working, but seemingly not working in general?
- sqlagent seems to show recursion limit on first call and show that Iron Maiden album query returns 0 rows, when there are really 21?
- currently index_data.py is not called because the test Iron Maiden query doesn't need to use it

## Prerequisites

- A kubernetes cluster with a MI300X accelerator. Other AMD GPUs with rocm support might also work, but this example was only tested with a MI300X.
- `kubectl` setup to use the cluster
- `helm` installed

### Setup

Note: You can skip this part if you've already set up your storage and example files.

`agentqna-setup-chart` folder has a small helm Chart for setting up a PVC on the cluster and a pod that copies files used by the example. Both the setup helm and agentqna helm use the same `rocm-values.yaml` file. Modify `rocm-values.yaml` file to use a different LLM model if desired. By default, `meta-llama/Llama-3.3-70B-Instruct` is used. Note that smaller models may not work well, I tried with `Qwen/Qwen2.5-7B-Instruct` but it didn't call some tools properly.

```
# Create and use namespace (optional)
kubectl create namespace agentqna
kubectl config set-context --current --namespace=agentqna
export PVC_NAME=agentqna-pvc
export HUGGINGFACEHUB_API_TOKEN=your-hf-token
# Output rendered file to a folder
helm template agentqna-setup-chart agentqna-setup-chart -f rocm-values.yaml --set global.modelUsePVC=${PVC_NAME} --set global.HUGGINGFACEHUB_API_TOKEN=${HUGGINGFACEHUB_API_TOKEN} --output-dir agentqna-setup
# Create PVC
kubectl apply -f agentqna-setup/agentqna-setup/templates/pvc.yaml
# Wait for the PVC to be ready/Bound and run the copy pod. The pod downloads files required by the example to the PVC
kubectl apply -f agentqna-setup/agentqna-setup/templates/copy-pod.yaml
```

Depending on the cluster network connection and model size, downloading model files might take 10 minutes or more.

## Deploy with Helm chart

Deploy by running these commands. Change toolPVC in rocm-values.yaml to your pvc name if using your own. `--skip-tests` is used because many of them expect the services to be running in 10 seconds or so and the logs/test pods clutter stuff, we just test manually.

```
# Run the dependency update script in folder GenAIInfra/helm-charts to include vllm changes:
./update_dependency.sh
# Then in GenAIInfra/helm-charts/agentqna
helm dependency update .
export PVC_NAME=agentqna-pvc
export HUGGINGFACEHUB_API_TOKEN=your-hf-token
helm template agentqna . -f rocm-values.yaml --set global.modelUsePVC=${PVC_NAME} --set global.HUGGINGFACEHUB_API_TOKEN=${HUGGINGFACEHUB_API_TOKEN} --set global.sharedSAName="" --skip-tests > agentqna.yaml
kubectl apply -f agentqna.yaml
```

Run `kubectl get pod`. All pods should show status 'Running'. The vllm pod might take some time to start as it loads the model (around 5-10 minutes), even though we download the safetensor files beforehand.

Port forward the supervisor service to your local computer:

```
kubectl port-forward svc/agentqna-supervisor 9090:9090
```

Then in another terminal window:

```
curl http://localhost:9090/v1/chat/completions \
    -X POST \
    -H "Content-Type: application/json" \
    -d '{"messages": "How many albums does Iron Maiden have?"}'
```

Which, for me, returns `{"id":"3fe1c7b403c109824ef48cbeb58ca2e0","text":"21","prompt":"How many albums does Iron Maiden have?"}`. Note that sometimes it takes several minutes to get a response, due to weird tool calls? Or maybe some bug?