# Duranta 5G Prometheus MCP with server-side instructions and registered prompts

**Corrects the earlier client-side AGENTS.md approach.** The Kubernetes MCP
config example you supplied has two server-side features: `server_instructions`
and `[[prompts]]`. This implementation has those same features in `prompts/config.toml`.
They are advertised by the MCP server itself, not merely displayed in the client.

The upstream Prometheus MCP does not read this TOML. A small FastMCP gateway
runs in the same Pod as the official Prometheus MCP server, proxies its tools,
resources, and built-in prompts, adds the Duranta-specific prompts, and publishes
server instructions during MCP initialization. Neither upstream Prometheus MCP
nor its Go tool definitions are modified.

## One-time build before Argo CD sync

From this directory:

```bash
podman build -f gateway/Containerfile -t quay.io/jodavis/duranta-prometheus-mcp-gateway:v0.1.0 gateway/
podman push quay.io/jodavis/duranta-prometheus-mcp-gateway:v0.1.0
```

Choose a repository you control and update deployment.yaml if needed. The public
image above is a **build target**, not a claim that the image already exists.
For stricter reproducibility, pin the resulting image digest in deployment.yaml.

## Deploy

```bash
oc apply -k .
oc get pods -n duranta-observability
oc logs -n duranta-observability deployment/prometheus-mcp -c duranta-prompt-gateway
oc logs -n duranta-observability deployment/prometheus-mcp -c upstream-prometheus-mcp
```

Add this directory to your GitOps Argo CD application. Kustomize's
`configMapGenerator` adds a hash suffix to the ConfigMap name and rewrites the
Deployment volume reference: changes to TOML trigger a rollout. The base
MCP endpoint is unchanged:

`http://prometheus-mcp.duranta-observability.svc:8080/mcp`

## Verify registered prompts and server instructions

From an in-cluster MCP Inspector, connect via Streamable HTTP to the above URL.
Check **Prompts**: `duranta-discover-app-metrics`, `duranta-sip-register`,
`duranta-sip-message-delivery`, `duranta-sip-408`,
`duranta-5gc-pdu-sessions`, `duranta-radio-signaling`.
Check the `initialize` result for `instructions` containing the Duranta 5G
focus text; client support for exposing instructions in the UI varies.
Check **Tools**: `query`, `range_query`, `metric_metadata`, `label_names`, etc.

## Scope and limitations

* Both hub and spoke point at THIS cluster's `thanos-querier` Service, without
  hardcoded hub/spoke names. Server instructions adapt to local workload roles.
* The upstream MCP is restricted to its documented read-only core tools.
* The gateway has no monitoring credentials; only the upstream container has
  the ServiceAccount token and uses it to authenticate to Thanos. However,
  anyone who can reach the gateway can issue monitoring queries with those
  effective permissions. Do not expose an unauthenticated Route.
* Kamailio is assumed deployed on hub, but the exporter metrics still need to
  be validated; Prometheus MCP does not install or scrape them itself.
* These instructions guide LLM clients but do not force compliance. MCP clients
  may ignore initialize-time instructions and/or prompt discovery.
* The source is deliberately small; it **is custom glue**, an additional
  gateway image and runtime dependency. If you want no custom code, use the
  stock Prometheus MCP and client-side instructions instead.
* Static YAML/TOML/Python syntax was validated in this environment; FastMCP
  could not be installed locally for a live MCP handshake test.

Upstream documentation:
https://github.com/prometheus/prometheus-mcp
https://github.com/PrefectHQ/fastmcp/blob/main/docs/servers/providers/proxy.mdx
