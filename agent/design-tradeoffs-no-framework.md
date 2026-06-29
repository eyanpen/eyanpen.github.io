# Design Trade-offs: Why Hermes (and Many Popular Agents) Don't Use LangChain / LangGraph

> Note: The Hermes repo contains **no** explicit statement saying "we don't use LangChain because…".
> This article works on two levels: the **industry-wide common reasons** (general analysis) and the **orientation empirically visible in Hermes's code** (with source-backed evidence and citations).

## 1. Premise: The Core of an Agent Loop Is Actually Simple

Many people assume orchestrating an agent requires a "framework," but the core loop is essentially just a while:

```text
while not termination_condition:
    resp = llm(messages, tools)
    if resp.tool_calls:
        execute tools, append results back to messages
    else:
        return resp.content
```

Hermes's `run_conversation` (`agent/conversation_loop.py`) is essentially this.

**What's truly hard isn't the loop itself, but the engineering details around it**: streaming output, interruption, budget control, context compression, prompt caching, provider failover, concurrent tool execution, error classification and retries… And these are precisely where generic frameworks abstract the shallowest and most easily "get in the way." This is the key to understanding "why so many projects don't use a framework."

## 2. Industry Level: Common Reasons Popular Agent Projects Bypass Frameworks

Applicable to most projects with their own hand-rolled loops (Hermes, OpenHands, Aider, Codex CLI, Claude Code, etc.):

1. **Mismatch between abstraction and control**. Frameworks wrap LLM calls / messages / memory / tools into objects (Chain / Runnable / Graph node). But a production-grade agent needs precise control over "every byte sent to the model"—for example, which message Anthropic's `cache_control` is attached to, how reasoning content is stored, how to degrade to a fallback model on errors. Framework abstractions make this "last mile" harder, often forcing you to monkey-patch around the framework. An analogy: a framework is like a universal remote—it controls most appliances, but the one device in your home with the latest features happens to be missing a button, so you have to open the back panel and wire it directly.

2. **Differences in API shape across providers**. When you need to support OpenAI chat.completions, Anthropic Messages, Bedrock, Codex Responses and other API shapes simultaneously, a framework's "unified LLM interface" tends to lag behind each vendor's latest features (new models, new parameters, reasoning, prompt caching), making you reactive in catching up. It's like translation software always being a step behind the original: capabilities a model vendor just released only get supported by the framework in its next version, while you want to use them immediately.

3. **Debugging and readability**. A hand-rolled loop's stack trace points straight to your own code; frameworks are often multi-layer abstractions + callbacks, with deep error stacks and implicit behavior. Long-maintained projects value readability more.

4. **Dependency and supply-chain risk**. A framework is itself a huge transitive dependency tree, with frequent version changes and unstable APIs, enlarging the supply-chain attack surface.

5. **Version churn**. LangChain's early API changed drastically (`LLMChain` → LCEL → LangGraph). Binding your core logic to a fast-moving framework makes migration costly.

## 3. Orientation Empirically Visible in Hermes's Code (Verifiable)

- **Extreme dependency minimization + supply-chain defense**. `pyproject.toml` has long comment sections explaining: core dependencies are **all exact-pinned** (`==X.Y.Z`, no ranges), triggered by the Mini Shai-Hulud worm attack of 2026-05; it explicitly states *"smaller dependencies = smaller blast radius for the next supply-chain attack"*, and provider-specific dependencies are all lazily installed (`tools/lazy_deps.py`). A project that manages "dependency footprint" as a first-class concern naturally won't pull in a heavy dependency like LangChain.

- **The only LLM SDK is `openai==2.24.0`**; all other multi-provider support relies on a hand-rolled Transport / Adapter layer (`agent/transports/`, `agent/*_adapter.py`) for adaptation, using the OpenAI message format as the intermediate representation.

- **Lots of hand-rolled engineering around the loop**: interruption checks, `agent/iteration_budget.py`, `agent/error_classifier.py` (failover), `agent/context_compressor.py`, `agent/prompt_caching.py`, `agent/tool_executor.py` (concurrent tool execution). The single file `run_agent.py` alone is about 5,300 lines, and the loop-related modules total tens of thousands of lines, showing they **deliberately invested in owning this loop** rather than outsourcing it to a framework.

- **It's not "build everything yourself."** When they're willing to hand off control (e.g., handing the tool loop to the OpenAI Codex app-server), Hermes integrates explicitly; what it opposes is "replacing your own core loop with a generic framework," not all integration.

> Summarizing Hermes's "reasoning": **make controllability and supply-chain security the top priority, and since the agent core loop is simple enough, the payoff of building it yourself > the convenience a framework brings.**

## 4. Deep Dive: Extreme Dependency Minimization + Supply-Chain Defense

Hermes's supply-chain defense isn't a slogan—it's a multi-layer mechanism baked into `pyproject.toml` plus four concrete modules. Understanding this discipline makes it clear why "not using a framework" is a **necessary corollary** rather than a matter of taste.

### The Real Attack That Triggered This Design

Both `hermes_cli/security_advisories.py` and the `pyproject.toml` comments name the same event:

> **Mini Shai-Hulud worm (2026-05)** — poisoned `mistralai 2.4.6` on PyPI. This is a class of "self-propagating supply-chain worm": compromise a maintainer's account → publish a new version carrying malicious code → the malicious code steals more credentials at install/run time → use the stolen credentials to poison more packages, snowballing outward.

`pyproject.toml` puts it bluntly: had mistralai used a range declaration like `>=2.3.0,<3` at the time, then in the few hours before that malicious version was quarantined, **every install would have automatically pulled the poisoned version**. This is the direct motivation for pinning versions.

### Attack Type → Defense Strategy Mapping

Breaking Hermes's defenses down item by item, each one maps to a specific class of attack scenario:

1. **Poisoned new version on PyPI** (a worm or hijacked account publishing a malicious X.Y.Z+1). Strategy: core dependencies are **all exact-pinned** `==X.Y.Z`, with `uv.lock` locking transitive dependencies; a new version can only enter via "human edits the pin + re-lock + code review." Evidence: `pyproject.toml` comments + `[project.dependencies]` being all `==`.

2. **Transitive dependency blast surface** (few direct dependencies, but they indirectly pull in hundreds of packages, any one of which being poisoned compromises you). Strategy: **minimize core dependencies**—only packages used by EVERY session belong in core; provider/search/TTS/messaging-platform-specific dependencies are kicked out of core and switched to **lazy install**. Evidence: the "Scope rule" comment in `pyproject.toml` + `LAZY_DEPS` in `tools/lazy_deps.py`.

3. **`[all]` collateral failure** (one extra's transitive dependency gets quarantined, causing the entire `[all]` resolution to fail, silently degrading new installs and losing features). Strategy: move optional backends out of `[all]` into lazy-install, so a single package's quarantine only affects that feature without dragging down the rest. Evidence: the "Fragility" section in `tools/lazy_deps.py`'s docstring + the `[all]` comments.

4. **Malicious MCP extension package** (a third-party MCP server pulled by `npx`/`uvx` may be a poisoned package). Strategy: query the **OSV database** before startup, and **BLOCK** on a hit for an `MAL-*` malware advisory—only blocking confirmed malware, not ordinary CVEs, and allowing on network failure (fail-open). Evidence: `tools/osv_check.py::check_package_for_malware`.

5. **Hijacked install source via config** (a malicious config redirects installs to an attacker's mirror, git, or local path). Strategy: lazy-install **only allows installing from PyPI by package name**, doesn't support `--index-url`/`git+https`/`file:`, can only install allowlisted specs, and acts only on the current venv, never touching the system Python. Evidence: the "Security model" section in `tools/lazy_deps.py`.

6. **Dependencies with known CVEs**. Strategy: annotate CVEs item by item on pinned versions (`requests`/`aiohttp`/`starlette`/`PyJWT`/`anthropic`, etc.), with upgrades being intentional. Evidence: the inline `# CVE-2026-xxxxx` comments in `pyproject.toml`.

7. **A poisoned package already installed in the user's environment** (a detection backstop after the first line of defense is breached). Strategy: on every CLI/gateway startup, use `importlib.metadata.version()` to compare against a **list of known-compromised versions**, alerting + giving remediation guidance on a hit; the user can `hermes doctor --ack <id>` to acknowledge and persist it. Evidence: `ADVISORIES` in `hermes_cli/security_advisories.py`.

### Key Strategies Expanded

- **Pinned versions + lockfile (strategy 1)**: A range declaration hands the decision of "when to pull a new version" over to PyPI and time; pinning takes it back to "one explicit human commit." The cost is manual `uv lock`; the benefit is that an attacker **has no automatic channel to reach the user**. The pyproject explicitly requires: an upgrade must simultaneously change the pin and regenerate `uv.lock`, and "don't add ranges back without a written reason."

- **Minimization + lazy install (strategies 2, 3) — the core of "blast radius"**: The engineering meaning of the original line *"smaller dependencies = smaller blast radius for the next supply-chain attack"* is: the shorter the core dependency list, the lower the probability that the next supply-chain attack reaches you. So dozens of provider-specific packages like `anthropic`, `firecrawl`, `edge-tts`, `modal`, `mautrix`, `elevenlabs` are all moved out of core and installed on first use via `lazy_deps.ensure("feature.name")`. A user who only uses one model vendor will never pull the dependency trees of dozens of other providers into the attack surface.

- **OSV malware interception (strategy 4)**: The only place with an "active outbound query"—before the agent actually launches an MCP server via `npx`/`uvx`, it first asks the Google OSV API "does this package have an `MAL-*` advisory?" It **deliberately blocks only confirmed malware, not ordinary CVEs** (to avoid false positives), and is **fail-open** (allow on network failure, never blocking normal use). The idea was inspired by Block/goose's extension checks.

### Why This Discipline Naturally Rejects LangChain

Putting it all together: LangChain/LangGraph is a **heavy dependency** that itself drags along a huge, frequently-changing transitive dependency tree. For a project that treats "core dependencies must be short, every package must be CVE-annotatable, and optional dependencies must all be lazified" as a hard rule, pulling in such a framework breaks strategies 1/2/3/6 all at once—the blast radius explodes directly. So "not using a framework" isn't an isolated preference, but a **necessary corollary** of this supply-chain discipline.

### Aside: Dependencies Are Just One Layer

The trust model in `SECURITY.md` shows the supply chain is only part of the defense. Hermes treats everything that "enters the agent context" (web scrapes, email, gateway messages, files, MCP responses, tool results) as an **untrusted input surface**; in addition, `tools/url_safety.py`, `tools/threat_patterns.py`, `tools/skills_guard.py`, `tools/skills_ast_audit.py`, and `tools/tirith_security.py` handle prompt injection and skill-code auditing. Dependency minimization solves "is the code you installed trustworthy?"; these modules solve "is the data fed to the model at runtime trustworthy?"

## 5. Pros and Cons of Not Using a Framework

### Pros
- Full control over prompt / messages / caching / retries / degradation, able to use each vendor's new model features immediately.
- Few dependencies, small attack surface, reproducible builds, maintainable long-term.
- Intuitive debugging, short stacks, explicit behavior.
- Not dragged along by framework version upgrades.

### Cons
- **You have to build many wheels yourself**: retries, compression, memory, tool schemas, concurrency, observability—Hermes wrote tens of thousands of lines for this; the cost is real.
- **Lack of plug-and-play ecosystem**: LangChain has a vast supply of ready-made retrievers / loaders / integrations; building your own means wiring each one up.
- **Concepts must be developed yourself**: capabilities that LangGraph provides directly—graph orchestration, state machines, checkpoints—must be designed yourself (Hermes implemented similar capabilities itself using Kanban + delegate).
- **Team onboarding curve**: without the shared vocabulary of a common framework, newcomers must read the project's private abstractions.

## 6. When You Should Use a Framework Instead

Taking a balanced view, frameworks aren't without value:
- **Rapid prototyping / demos / one-off scripts**: ready-made integrations save time.
- **You need complex, visual, stateful orchestration and don't want to build it yourself**: LangGraph's graphs / checkpoints / human-in-the-loop are real value.
- **The team doesn't want to maintain the low-level loop** and is willing to trade abstraction constraints for speed.

**Rule of thumb**: in the exploration phase, frameworks let you move fast; once a product needs to evolve long-term, needs fine control over model behavior, and needs to control dependencies and security, most serious projects converge—like Hermes—back to "a hand-rolled thin core loop + the OpenAI SDK." This is also why popular coding agents like Aider, Codex CLI, and Claude Code likewise don't depend on LangChain / LangGraph.

## 7. One-Sentence Summary

An agent's core loop is simple enough that it's not worth wrapping in a heavy framework, while the genuinely hard engineering around the loop (caching / degradation / compression / multi-provider / supply chain) is precisely where framework abstractions get in the way—so Hermes chooses a hand-rolled thin core loop, trading dependency minimization for controllability and security.

---

If you found this article helpful, feel free to **like, bookmark, and follow**. I'll keep sharing more valuable content. Your support is my greatest motivation to create!
