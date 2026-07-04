# Ray Cluster with Docker Compose

This guide starts a vLLM Ray cluster using
[`examples/ray_serving/docker-compose.yaml`](examples/ray_serving/docker-compose.yaml).
The compose file builds a local image from
[`examples/ray_serving/Dockerfile.ray`](examples/ray_serving/Dockerfile.ray),
which extends `vllm/vllm-openai:latest` with `ray[default,cgraph]`.

## Required Variables

- `NODE_TYPE`: `head` or `worker`
- `HEAD_NODE_IP`: reachable IP address of the Ray head node
- `VLLM_HOST_IP`: reachable IP address of the current node
- `HF_HOME`: host path to the Hugging Face cache

Optional:

- `NCCL_SOCKET_IFNAME`: network interface for NCCL, for example `enp9s0`
- `RAY_PORT`: Ray head port, defaults to `6379`
- `CONTAINER_NAME`: container name, defaults to `vllm-ray-${NODE_TYPE}`
- `VLLM_RAY_IMAGE`: built image name, defaults to `vllm-openai-ray:local`
- `RAY_START_ARGS`: extra flags passed to `ray start`
- `HF_TOKEN`: Hugging Face token for gated models

## Start the Head Node

Run this on the machine that should be the Ray head:

```bash
NODE_TYPE=head \
HEAD_NODE_IP=10.0.0.80 \
VLLM_HOST_IP=10.0.0.80 \
HF_HOME=/home/ege/.cache/huggingface \
NCCL_SOCKET_IFNAME=enp9s0 \
docker compose -f examples/ray_serving/docker-compose.yaml up --build
```

Use this machine's reachable private IP for both `HEAD_NODE_IP` and
`VLLM_HOST_IP`.

## Start Worker Nodes

Run this on each worker machine. `HEAD_NODE_IP` stays the same, but
`VLLM_HOST_IP` must be the worker's own reachable IP.

```bash
NODE_TYPE=worker \
HEAD_NODE_IP=10.0.0.80 \
VLLM_HOST_IP=<WORKER_IP> \
HF_HOME=/path/to/huggingface/cache \
NCCL_SOCKET_IFNAME=<WORKER_INTERFACE> \
docker compose -f examples/ray_serving/docker-compose.yaml up --build
```

Every node must use the same network path. For example, use LAN IPs on all
nodes, or Tailscale IPs on all nodes; do not mix them.

## Verify the Cluster

In another terminal on the head node:

```bash
docker exec vllm-ray-head ray status
docker exec vllm-ray-head ray list nodes
```

You should see the head node as `ALIVE`, plus any workers that joined, with the
expected total GPU count.

## Run vLLM

Run vLLM from inside a Ray container, not on the host:

```bash
docker exec -it vllm-ray-head /bin/bash
```

Then start serving with Ray:

```bash
vllm serve /path/to/model \
  --tensor-parallel-size <GPUS_PER_NODE> \
  --pipeline-parallel-size <NUM_NODES> \
  --distributed-executor-backend ray
```

For two nodes with one GPU each:

```bash
vllm serve /path/to/model \
  --tensor-parallel-size 1 \
  --pipeline-parallel-size 2 \
  --distributed-executor-backend ray
```

## Stop the Cluster

Stop the container on each node:

```bash
docker compose -f examples/ray_serving/docker-compose.yaml down
```
