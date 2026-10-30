# GhostCode Architecture Diagrams

These diagrams are drawn from `docs/IMPLEMENTATION_PLAN.md` and the current
`harness/` tree. Solid boxes already exist in the fork. Dashed boxes are planned
and do not exist yet (`harness/packages/ghostcode/` has not been created).

## 1. System architecture

```mermaid
flowchart TB
    subgraph clients["Clients"]
        cli["CLI<br/>apps/cli<br/>ghostcode / ghost"]
        webui["Web UI<br/>apps/web<br/>127.0.0.1:3080"]
        desktop["Desktop<br/>apps/desktop"]
        android["Android app<br/>Flutter, Phase 8"]:::planned
    end

    subgraph harness["GhostCode harness (fork of DeepSeek Harness, Cordis plugins)"]
        core["Agent loop + session log<br/>packages/core, packages/session"]
        base["base bundle<br/>packages/bundle/base<br/>credentials, sandbox, approval, tools"]
        llmsvc["ctx.llm service<br/>packages/llm/llm"]
        toolsvc["ctx.tools service"]
        websvc["web_search / web_fetch<br/>packages/web/*<br/>Exa, Perplexity, HTTP fetch"]
        creds["Credentials<br/>packages/credentials"]

        subgraph gc["packages/ghostcode/* (planned)"]
            gbundle["ghostcode bundle + profile"]:::planned
            profiler["ghostcode-profiler<br/>RAM, VRAM, CPU, disk, battery"]:::planned
            router["ghostcode-router<br/>local or BYOK, which model"]:::planned
            models["ghostcode-models<br/>catalog, pull, list, delete"]:::planned
            ollama["ghostcode-ollama<br/>ctx.llm adapter"]:::planned
            byok["ghostcode-byok<br/>OpenAI-compatible adapter"]:::planned
            laya["ghostcode-laya<br/>sidecar manager + decision hooks"]:::planned
            gweb["ghostcode-web<br/>search backend, offline cache"]:::planned
        end
    end

    subgraph external["Outside the harness"]
        ollamasrv["Ollama server<br/>localhost:11434"]
        layasrv["laya-serve<br/>Python sidecar, HTTP"]:::planned
        provider["Provider API<br/>user key"]
        internet["Web pages / search APIs"]
    end

    cli --> core
    webui --> core
    desktop --> core
    android -. "LAN, sdk profile JSON-RPC" .-> core

    core --> llmsvc
    core --> toolsvc
    base --- core
    gbundle -. mounts .-> profiler & router & models & ollama & byok & laya & gweb

    profiler --> router
    router -->|local| ollama
    router -->|BYOK| byok
    ollama -. registers on .-> llmsvc
    byok -. registers on .-> llmsvc
    byok --> creds
    models --> ollamasrv
    ollama --> ollamasrv
    byok --> provider

    laya -. "hooks: agent/pre-step,<br/>tools/pre-execute, tools/post-execute,<br/>agent/turn-stopping" .-> core
    laya --> layasrv

    gweb -. registers on .-> toolsvc
    websvc -. registers on .-> toolsvc
    websvc --> internet

    classDef planned stroke-dasharray: 5 5
```

## 2. Router decision (Phase 4)

```mermaid
flowchart TD
    start(["Before every model load"]) --> probe["Profiler reads free RAM, VRAM,<br/>CPU, disk, battery"]
    probe --> need["need = weights + kv_cache(num_ctx) + overhead"]
    need --> forced{"User forced a mode?"}
    forced -->|yes| useForced["Use forced mode"]
    forced -->|no| fits{"Some catalog model fits?<br/>GPU: need ≤ free_vram x 0.9<br/>RAM: need ≤ free_ram x 0.7"}
    fits -->|yes| largest["Pick largest model that fits,<br/>prefer GPU fit"]
    fits -->|no| key{"BYOK key present?"}
    key -->|yes| byokMode["Use BYOK"]
    key -->|no| smallest["Smallest model +<br/>warn: this will be slow"]
    largest --> load["Load in Ollama"]
    smallest --> load
    load --> oom{"Out of memory?"}
    oom -->|yes| stepDown["Step down one model size"] --> load
    oom -->|no| run(["Run session, show mode badge"])
    byokMode --> online{"Online?"}
    online -->|yes| run
    online -->|no| fallback["Fall back to local with notice"] --> fits
    useForced --> run
```

## 3. One agent turn with Laya decisions (Phase 6)

```mermaid
sequenceDiagram
    autonumber
    actor User
    participant Loop as Agent loop
    participant Laya as ghostcode-laya
    participant Sidecar as laya-serve
    participant LLM as ctx.llm (Ollama or BYOK)
    participant Tools as ctx.tools
    participant Log as Session log

    User->>Loop: Request
    Loop->>Laya: agent/pre-step
    Laya->>Sidecar: task type (choice), needs web (noul), too hard (noul)
    Sidecar-->>Laya: scores, about 33 ms
    Note over Laya,Sidecar: Skip decision if no reply in 200 ms
    Laya->>Log: log decision event
    Laya-->>Loop: prompt section, tool subset, maybe suggest BYOK
    Loop->>LLM: agent/request, llm/stream
    LLM-->>Loop: text + tool call
    Loop->>Laya: tools/pre-execute
    Laya->>Sidecar: is this command destructive? (noul)
    alt destructive
        Laya-->>User: ask for approval
    end
    Loop->>Tools: tools/execute
    Tools-->>Loop: result
    Loop->>Laya: tools/post-execute
    Laya->>Sidecar: did it succeed? (noul)
    Laya-->>Loop: retry or continue
    Loop->>Laya: agent/turn-stopping
    Laya->>Sidecar: is the request fully done? (noul)
    Laya->>Log: log decision event
    Laya-->>Loop: stop or continue
    Loop-->>User: Answer
```

## 4. Phase dependencies

```mermaid
flowchart LR
    p0["Phase 0<br/>Environment setup"] --> p1["Phase 1<br/>Fork + Customize<br/>Track A"]
    p0 --> p2["Phase 2<br/>Model download<br/>Track B"]
    p0 --> p3["Phase 3<br/>Ollama adapter<br/>Track C"]
    p1 --> p4["Phase 4<br/>Profiler + router"]
    p2 --> p4
    p3 --> p4
    p4 --> p5["Phase 5<br/>BYOK"]
    p5 --> p6["Phase 6<br/>Laya<br/>Track D"]
    p6 --> p7["Phase 7<br/>Web tools"]
    p7 --> p8["Phase 8<br/>Android"]
    p8 --> p9["Phase 9<br/>Eval + demo"]

    classDef demo fill:#e8f4ea,stroke:#2e7d32
    class p1,p3,p4,p5 demo
```

Green phases (1, 3, 4, 5) make up the minimum demo from plan section 9.
