# vLLM Metrics

vLLM exposes a `/metrics` endpoint for observability.
It integrates with Prometheus, so you can build a Grafana dashboard.

Let’s define a few important metrics:

- **E2E request latency**: total time from request receipt to the final token.
- **TTFT (Time To First Token)**: time until the first token is generated.
- **TPOT (Time Per Output Token) / Inter-Token Latency (ITL)**: average time between tokens.

TTFT is higher because the first token requires prefill — computing the KV cache for the entire prompt. Subsequent tokens reuse that cache and only compute the new token’s forward pass.

The vLLM repo includes Grafana examples:

```bash
git clone https://github.com/vllm-project/vllm.git
cd vllm/examples/online_serving/prometheus_grafana
docker compose up
```

Create a Prometheus data source in Grafana and upload **grafana.json** to import the dashboard.

After some usage, we can observe sub-100ms P99 for both TPOT and TTFT. That feels very fast in practice — though it’s expected since this is a relatively small model and we’re not heavily loading it.

And that’s it. A fully self-hosted coding agent stack — model, inference server, tunneling, and agent — under your control, end to end.
