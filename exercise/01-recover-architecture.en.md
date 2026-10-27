**Language:** [ภาษาไทย](01-recover-architecture.md) | English

# Task 1: Recover the architecture (approximately 15 min)

> **Lecture link:** T7 architecture recovery, AICH2 (traceability), AICH3 (context), and the JSS finding that 93% of studies never validate AI output.
> **Question:** how much of an architecture can AI recover, and how much does it depend on the evidence it reads?

**Don't look at the original Online Boutique repository during this task:** its README shows the architecture diagram you are recovering.

For each run, start a **new chat or session** in your agent and send the message below. The agent follows [`prompts/1-recover-architecture.md`](../prompts/1-recover-architecture.md) and writes its result to `exercise/outputs/1<run>-architecture.md`.

**Run A: interfaces only**
```
Follow prompts/1-recover-architecture.md
run: A
evidence: system/online-boutique/protos/demo.proto
```

**Run B: interfaces + deployment config**
```
Follow prompts/1-recover-architecture.md
run: B
evidence: system/online-boutique/protos/demo.proto, system/online-boutique/kubernetes-manifests/
```

While the agent works, watch which files it opens. If it reads files outside the evidence you allowed, write that down in your prompt log.

## Results

- Run A: [`outputs/1A-architecture.md`](outputs/1A-architecture.md)
- Run B: [`outputs/1B-architecture.md`](outputs/1B-architecture.md)

Open each file with **Markdown: Open Preview** (`Ctrl+Shift+V` / `⇧⌘V`) to see the diagram.

## Verify Run B yourself (the important part)

Check **at least five** dependencies in Run B's table against the files. Open the cited file and find the evidence. You may also check the service's source code, e.g. `src/checkoutservice/main.go`.

| Caller → Callee | AI's evidence | Verified? ✅ / ❌ / ❓ | What you found |
|---|---|---|---|
| Frontend → Checkout Service | `frontend.yaml` — `CHECKOUT_SERVICE_ADDR: "checkoutservice:5050"` | ✅ | `frontend.yaml` line 79; `src/frontend/main.go` line 136 reads the variable, and `handlers.go` line 355 calls `PlaceOrder` over gRPC. |
| Checkout Service → Cart Service | `checkoutservice.yaml` — `CART_SERVICE_ADDR: "cartservice:7070"` | ✅ | `checkoutservice.yaml` line 67; `main.go` line 325 (`GetCart`) and line 333 (`EmptyCart`). The diagram label "Reads cart" is incomplete: checkout also empties the cart. |
| Checkout Service → Email Service | `checkoutservice.yaml` — `EMAIL_SERVICE_ADDR: "emailservice:5000"`; `emailservice.yaml` — Service port 5000 | ✅ | `checkoutservice.yaml` line 63; `emailservice.yaml` lines 82–83 map Service port 5000 to container port 8080; `main.go` line 380 calls `SendOrderConfirmation`. |
| Recommendation Service → Product Catalog Service | `recommendationservice.yaml` — `PRODUCT_CATALOG_SERVICE_ADDR: "productcatalogservice:3550"` | ✅ | `recommendationservice.yaml` line 60; `recommendation_server.py` lines 131–136 open a gRPC channel and a `ProductCatalogServiceStub`. |
| Cart Service → Redis Cart | `cartservice.yaml` — `REDIS_ADDR: "redis-cart:6379"`; `redis-cart` Service port 6379 | ✅ | `cartservice.yaml` line 50 (`REDIS_ADDR`) and the `redis-cart` Deployment/Service (lines 92, 146); `src/cartservice/src/Startup.cs` line 29 reads `REDIS_ADDR` and line 40 registers `RedisCartStore`. |

- **Wrong or missing** in Run B (false dependencies, missing ones, containers to question): I found no false dependencies; all 16 rows match the manifests. Missing: Frontend → `shoppingassistantservice`, an **HTTP** call (`frontend.yaml` line 83, `handlers.go` line 464), which the agent correctly put under Gaps but left out of the diagram. Questionable: every service is labelled only "gRPC service" with no language, and Checkout → Cart is labelled "Reads cart" although checkout also empties the cart.

## Reflect (2–3 sentences each)

1. Compare A and B. What could the agent **not** know from the `.proto` file alone, and why?

   From the proto alone, the agent found the services but no dependencies, no frontend, no Redis and no end user. A `.proto` file is an interface contract: it defines what each service offers, not who calls it, where it runs or what it stores. Those facts only appear in the deployment config (the `*_ADDR` environment variables) and the source code.

2. Which **one** mistake would mislead a new team member most if the diagram were published without checking?

   Leaving `shoppingassistantservice` out of the diagram. The frontend calls it over HTTP, not gRPC, and it depends on external Google Cloud services (AlloyDB, Gemini). A new team member reading only the diagram would think the frontend talks only to internal gRPC services.
