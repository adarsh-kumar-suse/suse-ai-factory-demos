# suse-ai-factory-demos

# Installing individual Applications

## Ollama




# Installing Blueprints

## Simple Chatbot with RAG

### Description
Simple Chatbot with RAG using ollama, open-webui and open-webui-mcpo.
This blueprint uses CPU Inference. Suitable for PoCs, quick explorations.
Retrieval-augmented generation uses the ChromaDB vector store embedded in
open-webui.

## Requirements

1) Default StorageClass defined in the cluster for persistent volumes.

2) cert-manager instance running in the cluster.

The open-webui chat UI is exposed via Ingress at https://suse-ollama-webui. 

point a DNS record (or /etc/hosts entry) for the host "suse-ollama-webui"
at the cluster's ingress controller to reach it. The hostname can also be patched after deployment.


## Customize the Deployments

### Ollama

If you want to utilize a GPU you need to enable the GPU under the ollama settings

![Edit Helm](../assets/Simple_Chatbot_with_RAG-ollama_enable.png)

