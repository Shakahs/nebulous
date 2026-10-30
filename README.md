<p align="center">
  <img src="./static/nebu_logo2_alpha.png" alt="Nebulous Logo" width="400">
</p>


__A globally distributed container orchestrator__

An alpha orchestrator focused on __accelerated compute__, with RunPod container
management, Redis Streams processors, and VPN integration. The Rust server and
CLI ship as a binary; the server also requires supporting services. :crab:
   
Why not Kubernetes? See [why_not_kube.md](docs/why_not_kube.md)   
   
> [!WARNING]
> Nebulous is in __alpha__, things may break.

> [!NOTE]
> Feature status below was audited against source at commit
> `96866a99d956dbf09b27d2065821587c60b75165` (version `0.1.91`) on 2026-10-30.
> This is a source audit, not live deployment, security, or scale validation.
> Diagrams illustrate the design; planned features are identified explicitly.

## Feature status

**Implemented** means substantive code is connected to the application.
**Partial** means important parts exist, but the broader capability is incomplete.
**Not implemented** means declarations, examples, comments, or plans do not supply
an executable implementation of the feature.

| Feature | Status | Current implementation and limits |
|---|---|---|
| RunPod GPU and CPU containers | Implemented | Provisioning, monitoring, reconciliation, exec, logs, and deletion paths exist. CPU-only provisioning is supported. See [RunPod implementation](src/resources/v1/containers/runpod.rs). |
| Cross-cloud GPU orchestration and accelerator discovery | Partial | The active [platform factory](src/resources/v1/containers/factory.rs) accepts `runpod` and `kube`. EC2, GCE, Nebius, and Azure are not connected provisioning backends. `get platforms` prints a hard-coded list including unsupported EC2/GCE entries; accelerator names alone do not establish provider support. |
| Automatic capacity selection and cross-cloud autoscaling | Partial | Processor replicas scale using Redis backlog. RunPod chooses among eligible datacenters using location/stock ordering. There is no implemented general optimizer comparing cloud prices or moving workloads to cheaper providers. |
| Kubernetes workloads | Partial | The [Kubernetes backend](src/resources/v1/containers/kube.rs) creates Jobs, requests GPU resources, and monitors status. Its reconcile, exec, logs, and delete methods are no-ops. Installing the control plane through Helm is a separate capability. |
| Tailscale/Headscale connectivity | Implemented | [VPN clients](src/vpn/mod.rs) support enrollment keys and device discovery/removal. RunPod startup joins the VPN. A configured Tailscale or Headscale deployment is required. Reachability depends on its routing, DNS, and policies. |
| Automatic namespace network isolation | Not implemented | Namespace ownership is checked at the API level, but namespace-specific VPN policies are not generated. Containers use the common `tag:container`, and Headscale enrollment uses the `default` user. Namespace membership does not by itself enforce network isolation. |
| Decentralized, lazy peer-to-peer data replication | Not implemented | There is no Iroh dependency, peer discovery, or peer-fetch path. [Issue #33](https://github.com/agentsea/nebulous/issues/33) tracks Iroh integration. There is no peer-first/S3-fallback mechanism. |
| S3-backed volumes and data transfers | Implemented | [rclone integration](src/volumes/rclone.rs) transfers data between configured sources and destinations, including sync/copy and bidirectional-sync paths. RunPod maps local destinations into `/nebu/cache` with symlinks. S3 is a direct source, not a fallback after peer lookup. The `RCLONE_MOUNT` enum value does not have a corresponding mount implementation in the inspected transfer paths. |
| Live suspension, restoration, GPU migration, and container forking | Not implemented | There is no CRIU or GPU checkpoint/restore integration. The [checkpoint selector](src/select/checkpoint.rs) selects model-training checkpoint directories; it does not capture running processes or GPU state. |
| Runtime and request/response metering | Implemented | RunPod monitoring and the HTTP container proxy emit usage events to OpenMeter. See [runtime metering](src/resources/v1/containers/runpod.rs) and [proxy metering](src/proxy/meters.rs). This is not a complete invoicing or payment system. |
| Multi-tenant resource ownership and authorization | Implemented, with limits | Authentication middleware, user/organization ownership filters, namespace authorization, scoped S3 credentials, and Redis permissions exist. This does not establish the stronger claim of complete workload or network isolation. |
| AES-256 encrypted secrets | Implemented | [AES-256-GCM](src/entities/secrets.rs), nonces, database storage, CRUD endpoints, and RunPod environment-variable injection exist. The key variable is `NEBU_ENCRYPTION_KEY`. |
| Redis Streams processors | Implemented | Creation, worker reconciliation, message submission, stream reads, return streams, and WebSocket responses exist. Workers must implement the message/response protocol; an arbitrary HTTP inference container is not automatically a processor worker. |
| Processor backlog autoscaling | Implemented | [Scaling logic](src/resources/v1/processors/standard.rs) measures consumer-group backlog and applies scale-up/down thresholds and durations. Limits are named `min_replicas` and `max_replicas`. The implementation contains RunPod-specific assumptions. |
| Processor scale-to-zero | Not implemented | Startup and scaling clamp the minimum to one using `.max(1)`. The `scale.zero` model field does not provide operational zero-scaling behavior. |
| Processor message-schema enforcement | Not implemented | Schema fields can be stored and updated, but [message submission](src/handlers/v1/processors.rs) does not validate content against the configured schema before placing it in Redis. |
| HTTP Services with stable endpoints, load balancing, and latency autoscaling | Not implemented | The [Service implementation](src/resources/v1/services/standard.rs) is commented out; service CRUD routes and normal service-controller startup are absent. A proxy to individual containers exists, but does not implement the Service resource. Service scale-to-zero and Service-specific billing are consequently unavailable. |
| Local API server and Docker startup | Implemented, with dependencies | The Rust server runs locally. `serve --docker` starts a Compose stack, not a self-contained process. Database, broker, authentication, VPN, storage, and optional metering configuration are needed according to the features used. See [server startup](src/commands/serve_cmd.rs) and [Compose configuration](deploy/docker/docker-compose.yaml). |
| Seamless operation at thousands of nodes | Unverified | No benchmark or deployment evidence establishing this scale was identified in the source audit. |

## Concepts

### Cross-cloud Autoscaling

**Partial.** Processors scale worker containers using Redis backlog thresholds
and durations. The active platform factory supports RunPod and a partial
Kubernetes backend. General cross-cloud price optimization and automatic
datacenter resource discovery are not implemented. See the
[platform factory](src/resources/v1/containers/factory.rs) and
[processor scaling](src/resources/v1/processors/standard.rs).

<div align="center">
<img src="./static/scale3.png" alt="description" width="500" height="175"/>
</div>

### Globally Segmented Networks

**Connectivity implemented; automatic namespace isolation is not.** Nebulous
integrates with [Tailscale](https://tailscale.com/kb/1136/tailnet) and
[Headscale](https://github.com/juanfont/headscale) for enrollment and device
discovery. Reachability depends on the configured VPN and its DNS/policies.
Containers share `tag:container`, and Headscale enrollment uses the `default`
user; Nebulous does not generate namespace-specific VPN ACLs. The diagram shows
the intended namespace segmentation, not an enforced isolation boundary.

<div align="center">
<img src="./static/namespace2.png" alt="description" width="700" height="175"/>
</div>

### Decentralized Data Layer

**Planned; not implemented.** The intended design uses
[Iroh](https://www.iroh.computer/) for lazy peer-to-peer replication, as shown
below. There is no Iroh dependency or peer-fetch implementation in this version;
[issue #33](https://github.com/agentsea/nebulous/issues/33) tracks integration.
Current [data transfers](src/volumes/rclone.rs) use rclone with configured sources
such as S3. S3 is a direct source, not a fallback after a peer lookup. RunPod
stores local destinations under `/nebu/cache` and exposes them using symlinks.
The `RCLONE_MOUNT` enum value has no corresponding mount implementation in the
inspected transfer paths.

<div align="center">
<img src="./static/data_layer.png" alt="description" width="450" height="175"/>
</div>

### Live Migration

**Planned; not implemented.** The diagram illustrates the intended
[CRIU](https://criu.org/Main_Page) suspension/restoration, GPU migration, and
container-forking capabilities. There is no process/GPU checkpoint-and-restore
implementation. The [checkpoint selector](src/select/checkpoint.rs) selects
model-training checkpoint directories; it does not capture running containers.

<div align="center">
<img src="./static/migration.png" alt="description" width="500" height="200"/>
</div>

### Metering

**Metering primitives implemented.** RunPod monitoring emits runtime usage events
to [OpenMeter](https://github.com/openmeterio/openmeter). The container HTTP proxy
also supports request/response metrics. This provides usage reporting, not a
complete invoicing/payment system. See [proxy metering](src/proxy/meters.rs).

<div align="center">
<img src="./static/stripe.png" alt="description" width="450" height="125"/>
</div>


### Multi-tenant

**Resource ownership and authorization implemented; isolation has limits.**
Authentication middleware, user/organization ownership filters, namespace
authorization, scoped S3 credentials, and Redis permissions exist. They do not
establish complete workload or namespace network isolation. See
[namespace authorization](src/agent/ns.rs).

<div align="center">
<img src="./static/tenant.png" alt="description" width="450" height="175"/>
</div>

### Lightweight

**Local server implemented; large-scale operation unverified.** The Rust API
server can run locally, but requires a database and message broker, plus VPN,
authentication, storage, and optional metering configuration for the features
used. Docker mode starts a multi-container stack. No benchmark or deployment
evidence establishing seamless operation at thousands of nodes was identified.

<div align="center">
<img src="./static/local.png" alt="description" width="400" height="200"/>
</div>

## Installation

```sh
curl -fsSL -H "Cache-Control: no-cache" https://raw.githubusercontent.com/agentsea/nebulous/main/remote_install.sh | bash
```
> [!NOTE]
> The current [release workflow](.github/workflows/release.yml) builds macOS arm64
> and Linux amd64. Do not assume all OS/architecture combinations are published.
> The executable is `nebulous`; the image and `make install` also provide `nebu`.
> Examples below use `nebu`.

## Usage   

Export the keys of your cloud providers.
```sh
export RUNPOD_API_KEY=...
export AWS_ACCESS_KEY_ID=...
export AWS_SECRET_ACCESS_KEY=...
```

Export the bucket-related environment variables. Dummy values may satisfy
configuration parsing, but real storage operations require an accessible bucket
and valid credentials.
```sh
export NEBU_BUCKET_NAME="XXX" 
export NEBU_BUCKET_REGION="XXX"
export NEBU_ROOT_OWNER="XXX"
```

Run a local API server on docker

```sh
nebu serve --docker
```

Or optionally host the control plane on Kubernetes with our
[Helm chart](./deploy/charts/nebulous/). This is separate from Kubernetes workload
support: that backend creates Jobs and requests GPUs, but its reconcile, exec,
logs, and delete methods are no-ops. See [backend](src/resources/v1/containers/kube.rs).
   
Connect to your configured VPN using its client. There is no `connect` subcommand
in this version of the Nebulous CLI. VPN configuration uses `TS_APIKEY`,
`TS_TAILNET`, and `TS_LOGIN_SERVER` as applicable; see [configuration](src/config.rs).
    
Inspect the CLI's platform list (it is hard-coded and includes unsupported GCE
and EC2 entries; the active workload factory accepts `runpod` and `kube`).
```sh
nebu get platforms
```

> [!TIP]
> Prefer a pythonic interface? Try [neblous-py](https://github.com/agentsea/neblous-py)   
>     
> Prefer a higher level LLM interface? Try [orign](https://github.com/agentsea/orign)

### Containers

Let's run our first container. We'll create a container on runpod with 2 A100 GPUs which trains a model using TRL.   
   
First, let's find what accelerators are available.
```sh
nebu get accelerators
```

Now lets create a container.
```yaml
kind: Container
metadata:
  name: trl-job
  namespace: training
image: "huggingface/trl-latest-gpu:latest"
platform: runpod
command: |
  source activate trl && trl sft --model_name_or_path $MODEL \
      --dataset_name $DATASET \
      --output_dir /output \
      --torch_dtype bfloat16 \
      --use_peft true
env:
  - key: MODEL
    value: Qwen/Qwen2.5-7B 
  - key: DATASET
    value: trl-lib/Capybara 
volumes:  
  - source: s3://YOUR_BUCKET/data/training/models
    dest: /models
    driver: RCLONE_SYNC
    continuous: false
accelerators:
  - "2:A100_SXM"
restart: Never
```
The volume source must be accessible using the workload's storage credentials.
It syncs files to local storage rather than providing a lazy peer-backed mount.
To create the container

```sh
nebu create container -f mycontainer.yaml
```
> [!TIP]
> See our [container examples](examples/containers) for more.

List all containers
```sh
nebu get containers
```

Get the container by its ID from the listing. This version's `get containers`
command accepts an ID, not a name/namespace pair.
```sh
nebu get containers CONTAINER_ID
```

Exec a command in a container
```text
nebu exec trl-job -n training -c "echo hello"
```

Get logs from a container
```sh
nebu logs trl-job -n training
```

Send an HTTP request to a listening container, if VPN routing and DNS permit it.
This addresses one container, not a stable load-balanced Service endpoint.
```sh
curl http://container-{id}:8000
```
#### Meters

Runtime usage reporting is supported through [OpenMeter](https://openmeter.cloud/)
using the `meters` field. Invoicing and payment collection are separate.

```yaml
meters:
  - cost: 0.1
    unit: second
    currency: USD
    metric: runtime 
```

> [!TIP]
> See [container examples](examples/containers) for more.

### Secrets

Secrets store sensitive information such as API keys and credentials. They use
`AES-256-GCM` encryption with nonces and are stored in the database. Set
`NEBU_ENCRYPTION_KEY` to exactly 32 bytes. RunPod supports injecting their values
into container environment variables.

Create a secret
```sh
nebu create secret my-secret --value "$MY_SECRET_VALUE" --namespace my-app
```

Get all secrets
```sh
nebu get secrets
```

Get a secret by ID (this CLI does not accept a namespace flag for secret lookup).
```sh
nebu get secrets SECRET_ID
```

Delete a secret through `DELETE /v1/secrets/NAMESPACE/NAME` (or the ID endpoint).
The API implements deletion, but this version has no `delete secrets` CLI
subcommand. See [API routes](src/routes/mod.rs).

Secrets can be used in container environment variables.

```yaml
kind: Container
metadata:
  name: my-container
  namespace: my-app
env:
  - key: MY_SECRET
    secret_name: my-secret
```

### Namespaces

Namespaces provide a means to segment groups of resources across clouds.  

```yaml
kind: Container
metadata:
  name: llama-factory-server
  namespace: my-app
```
   
Namespace ownership is enforced at the API level; automatic namespace network
isolation is not implemented. Individual containers can be reached at addresses
such as `http://container-12345:8000` when the configured VPN DNS, routing, and
policies permit it.
    
Use a configured Tailscale or [Headscale](https://github.com/juanfont/headscale)
deployment. Configure the login server through `TS_LOGIN_SERVER`, not
`TAILSCALE_URL`. Availability of a free hosted service is not established by this
source audit.

#### Tenancy

Nebulous is multi-tenant from the ground up. Tenancy happens at the namespace level, when creating a namespace a user can set the owner to their user or an organization they are a member of.

```yaml
kind: Namespace
metadata:
  name: my-app
  owner: acme
```

This illustrates the namespace ownership model. Creation is exposed through
`POST /v1/namespaces`; this CLI has no namespace CRUD commands. The API request
contains the `metadata` object shown above. Resources created in the namespace
inherit its owner.
   
The authorization hierarchy is
```
owners -> namespaces -> resources
```

### Processors

Processors are containers that work off real-time data streams and are autoscaled based on back-pressure. Streams are provided by [Redis Streams](https://redis.io/docs/latest/develop/data-types/streams/).

**Implemented with limits.** Backlog thresholds, scale-up/down durations, worker
reconciliation, message submission, and return streams exist. Workers must
implement the message/response protocol; an arbitrary HTTP server is not
automatically a processor worker. See [worker examples](examples/processors/).

Processors are best used for bursty async jobs, or low latency stream processing.

The following illustrates the processor model. The image and worker command are
placeholders, not a runnable worker implementation. Creation is exposed at
`POST /v1/processors`; this CLI has no processor creation subcommand. The API
generates the processor stream rather than accepting a `stream` request field.

```yaml
kind: Processor
metadata:
  name: translator
  namespace: my-app
container:
  image: corge/translator:latest
  command: "redis-cli XREAD COUNT 10 STREAMS my-app:workers:translator"
  platform: runpod
  accelerators:
    - "1:A40"
min_replicas: 1
max_replicas: 10
scale:
  up:
    above_pressure: 100
    duration: 10s
  down:
    below_pressure: 10
    duration: 5m
```

**Scale-to-zero is not implemented.** The intended configuration below does not
work as a zero-scaling policy: startup and autoscaling clamp the minimum to one,
and the `scale.zero` field is not acted upon.

```yaml
min_replicas: 0  # Not operational: currently clamped to one.
scale:
  zero:
    duration: 10m  # Planned; not implemented.
```

**Schema enforcement is not implemented.** A schema can be stored as metadata,
such as the illustrative definition below, but message submission does not
validate the payload against it.

```yaml
schema:
  - name: text_to_translate
    type: string
    required: true
```

Send data to a processor stream

```sh
printf '%s\n' '{"text_to_translate":"Dlrow Olleh"}' | \
  nebu send messages translator --namespace my-app --wait
```

Read data through the API's stream/return endpoints; this CLI has no
`read processor` command. See [routes](src/routes/mod.rs).

List all processors

```sh
nebu get processors
```

> [!TIP]
> See [processor examples](examples/processors) for more.

### Services [planned; not implemented]

The intended design exposes containers on a stable address, balances traffic,
and scales using latency. **This resource is not operational:** the
[implementation](src/resources/v1/services/standard.rs) is commented out, Service
CRUD routes are absent, and normal startup does not run a Service controller.
A proxy to individual containers exists, but does not implement this resource.

The examples below document the proposed interface, not working commands or a
supported deployment schema. They are retained to describe the design intent.

```yaml
kind: Service
metadata:
  name: vllm-qwen
  namespace: inference
container:
  image: vllm/vllm-openai:latest
  command: |
    python -m vllm.entrypoints.api_server \
      --model Qwen/Qwen2-7B-Instruct \
      --tensor-parallel-size 1 \
      --port 8000
  accelerators:
    - "1:A100"
platform: gce
min_containers: 1
max_containers: 5
scale:
  up:
    above_latency: 100ms
    duration: 10s
  down:
    below_latency: 10ms
    duration: 5m
  zero:
    below_latency: 10ms
    duration: 10m
```

```sh
# Proposed CLI; not available in this version:
nebu create service -f vllm-qwen.yaml
```

The proposed interface would return the IP in `status`; this is not implemented.

```sh
# Proposed CLI; not available in this version:
nebu get services vllm-qwen -n inference
```

Service scale-to-zero is also planned, not implemented:

```yaml
min_containers: 0
```

The proposed Service interface includes response-token metering:

```yaml
meters:
  - cost: 0.001
    unit: token
    currency: USD
    response_json_value: "$.usage.prompt_tokens"
```

> [!TIP]
> The implemented **container proxy** can report response metrics using
> `metric: response_value` and `json_path: "$.usage.prompt_tokens"`. This applies
> only to traffic through that proxy, not direct VPN requests. The proposed
> Service field `response_json_value` above is not its implemented schema.

## SDK

:snake: Python https://github.com/agentsea/neblous-py    
   
:crab: Rust https://crates.io/crates/neblous/versions

## Roadmap

- [x] Support non-gpu containers
- [x] Processors
- [ ] Support for Nebius Cloud
- [ ] Support for AWS EC2
- [ ] Services
- [ ] Clusters
- [ ] Support for GCE
- [ ] Support for Azure
- [ ] Support for Kubernetes

Kubernetes workload support is partial rather than entirely absent: Job creation
and monitoring exist, but lifecycle methods remain stubs. Processor scaling is
implemented above a minimum of one replica. The following also remain open:

- [ ] Iroh/P2P replication and peer-first storage fallback
- [ ] CRIU/GPU suspension, restoration, live migration, and container forking
- [ ] Automatic namespace-specific VPN isolation
- [ ] Processor scale-to-zero and message-schema enforcement
- [ ] General cross-cloud price optimization

### Open upstream PRs

At the source-audit date, these PRs were open with merge conflicts:

| PR | Scope | Limits |
|---|---|---|
| [#47](https://github.com/agentsea/nebulous/pull/47) | Docker startup and image/deployment changes | Core Compose startup already landed through [#50](https://github.com/agentsea/nebulous/pull/50), the current baseline. |
| [#10](https://github.com/agentsea/nebulous/pull/10) | Universal platform abstraction and Docker/SSH adapters | Unfinished: SSH execution is a placeholder, EC2 is a Docker-over-SSH alias, GCE methods contain TODOs, and automatic platform selection/capacity checks are incomplete. |
| [#5](https://github.com/agentsea/nebulous/pull/5) | Integrated Headscale Helm deployment fixes | Deployment plumbing, not namespace network isolation. Several corresponding settings already exist on the baseline. |

None supplies the missing P2P, live-migration, HTTP Service, processor-zero-scaling,
or schema-validation capabilities. Recheck upstream before relying on PR status.

## Contributing

Please open an issue or submit a PR.

## Developing

Add all the environment variables shown in the [.env_](.env_) file to your environment.

Run a postgres and redis instance locally. This can be done easily with docker.

```sh
docker run -d --name redis -p 6379:6379 redis:latest
docker run -d --name postgres -p 5432:5432 postgres:latest
```

To configure the secrets store you will need an encryption key. This can be generated with the following command.
```sh
openssl rand -hex 16
```
Then set this to the `NEBU_ENCRYPTION_KEY` environment variable (exactly 32 bytes).
     
To optionally use OpenMeter for usage reporting, open an account with either
[their cloud](https://openmeter.cloud/) or run their
[open source](https://github.com/openmeterio/openmeter) and set `OPENMETER_TOKEN`
and `OPENMETER_URL`.
     
For VPN networking, use [Tailscale](https://tailscale.com/) or your own
[Headscale](https://github.com/juanfont/headscale) instance. Configuration reads
`TS_APIKEY`, `TS_TAILNET`, and `TS_LOGIN_SERVER` as applicable; consult
[configuration](src/config.rs) and [VPN interfaces](src/vpn/mod.rs).
   
Install locally
```
make install
```
   
Run the server
```
nebu serve
```
   
Login to the auth server. When you do, set the server to `http://localhost:3000`.
```
nebu login
```
   
Now you can create resources

```sh
nebu create container -f examples/containers/trl_small.yaml
```
   
When you make changes, simply run `make install` and `nebu serve` again.

## Inspiration

- [Kubernetes](https://kubernetes.io/)
- [Aurea](https://github.com/aurae-runtime/aurae)
- [SkyPilot](https://github.com/skypilot-org/skypilot)
- [RunPod](https://runpod.io/)
- [Prime Intellect](https://primeintellect.com/)
