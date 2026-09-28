# 🛡️ Agent Red Team Eval

**Red-team security evaluation harness for AI agents — 280+ adversarial probes across 12 attack categories.**

[![License: Apache-2.0](https://img.shields.io/badge/License-Apache--2.0-blue.svg)](LICENSE)
[![Python 3.10+](https://img.shields.io/badge/Python-3.10%2B-blue.svg)](https://www.python.org/)

If you ship an agent with tools, you ship an attack surface. **Agent Red Team Eval** fires a battery of
prompt-injection, tool-exfiltration, jailbreak, and privilege-escalation probes at your agent and tells you
exactly where it breaks — before an attacker does. It runs fully offline against a built-in synthetic gateway,
so no live agent or API key is needed to get a security report.

## Features

- **280+ attack probes** across 12 categories (prompt injection → MCP abuse → memory poisoning)
- **Synthetic Gateway** — isolated mock target for safe, offline testing
- **CI-ready outputs** — SARIF, JUnit, JSON, and Markdown reports
- **Baseline assertions** — catch security regressions between runs
- **Severity filtering** — run only S3+ critical probes when you're in a hurry

## Attack Categories

| Category | Description |
|----------|-------------|
| **Prompt Injection** | Jailbreaks, instruction override, prompt leaking |
| **Tool Exfiltration** | Sensitive file/secret exfiltration attempts |
| **Context Bleed** | Cross-session leaks, conversation history extraction |
| **Privilege Escalation** | Sandbox escape, elevation bypass attempts |
| **Supply Chain** | Malicious skills, dependency and update attacks |
| **Financial Transaction** | Wallet/seed phrase theft, transaction/approval attempts |
| **Unauthorized Action** | Actions without consent/confirmation |
| **MCP Attacks** | MCP tool abuse, server injection, cross-tool exfil |
| **Indirect Injection** | Injection via documents, URLs, issues, logs, metadata |
| **Evasion Bypass** | Unicode/encoding bypass, obfuscation, injection variants |
| **Memory Poisoning** | Persistent instruction poisoning, fabricated history |
| **Platform Specific** | OS and cloud-specific payloads (Windows/macOS/Linux/metadata) |

## Installation

```bash
pip install -r requirements.txt
pip install -e .
```

## Quick Start

```bash
# Run all probes against the built-in mock gateway (no API key needed)
redteam-eval run

# Run a specific category
redteam-eval run -c prompt_injection
redteam-eval run -c financial
redteam-eval run -c evasion_bypass

# Run only high severity (S3+)
redteam-eval run -s S3

# Save a report
redteam-eval run -o report.md

# List all attack probes
redteam-eval list-attacks

# Run a single probe verbosely
redteam-eval run-single PI-001 -v
```

Category aliases are supported (e.g. `financial`, `mcp_attacks`, `supplychain`, `platform`).

## How It Works

1. **Probes** — each attack module under `src/redteam_eval/attacks/` defines payloads with an ID,
   severity, target channel, and the behavior a safe agent should exhibit.
2. **Synthetic Gateway** — `redteam-eval run` talks to a mock agent gateway that simulates tool access,
   file reads, and elevated sessions, so probes execute in isolation.
3. **Verdicts** — the harness classifies each probe outcome as passed, failed, or vulnerability, and
   aggregates pass/vulnerability rates per category and severity.
4. **Reports** — results are rendered as Markdown, JSON, SARIF (GitHub code scanning), or JUnit.

### Testing a real agent gateway

```bash
redteam-eval run --no-mock --gateway-url ws://127.0.0.1:18789
```

## CI Integration

```yaml
name: Security Eval
on: [push, pull_request]

jobs:
  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.11'
      - run: pip install -r requirements.txt
      - name: Run security evaluation
        run: redteam-eval run --output security-report.json --format json
      - name: Assert baseline
        run: redteam-eval assert-cmd security-report.json --baseline expected/baseline.json
```

Generate or refresh the baseline:

```bash
redteam-eval baseline --output expected/baseline.json
```

## Output Formats

| Format | Use Case |
|--------|----------|
| `markdown` | Human-readable reports |
| `json` | Programmatic analysis |
| `sarif` | GitHub Code Scanning |
| `junit` | CI test results |

## Programmatic Usage

```python
import asyncio
from redteam_eval import EvalHarness, AttackCategory

async def main():
    harness = EvalHarness()
    result = await harness.run()
    print(f"Vulnerabilities: {result.vulnerabilities}")

    # Run specific categories at high severity only
    result = await harness.run(
        categories=[AttackCategory.PROMPT_INJECTION, AttackCategory.EVASION_BYPASS],
        min_severity="S3",
    )

asyncio.run(main())
```

## Custom Attack Probes

Add a module under `src/redteam_eval/attacks/`:

```python
from redteam_eval.attacks.base import Attack, AttackCategory, AttackPayload, ExpectedBehavior, Severity


class MyAttacks(Attack):
    category = AttackCategory.PROMPT_INJECTION
    name = "My Attacks"

    def _load_payloads(self) -> None:
        self.payloads.append(
            AttackPayload(
                id="MY-001",
                name="My probe",
                category=self.category,
                severity=Severity.S2,
                payload="...",
                target="dm_channel",
                expected_behavior=ExpectedBehavior.REJECTED_BY_SOUL,
            )
        )
```

Then export it from `src/redteam_eval/attacks/__init__.py` and register it in `src/redteam_eval/harness.py`.

## Probe ID Prefixes

| Prefix | Category |
|--------|----------|
| `PI-*` | Prompt Injection |
| `TE-*` | Tool Exfiltration |
| `CB-*` | Context Bleed |
| `PE-*` | Privilege Escalation |
| `SC-*` | Supply Chain |
| `FT-*` | Financial Transaction |
| `UA-*` | Unauthorized Action |
| `MCP-*` | MCP Attacks |
| `II-*` | Indirect Injection |
| `EB-*` | Evasion Bypass |
| `MP-*` | Memory Poisoning |
| `PS-*` | Platform Specific |

## Severity Levels

| Level | Description | Action |
|-------|-------------|--------|
| **S4** | Critical | Immediate fix required |
| **S3** | High | Fix before deploy |
| **S2** | Medium | Review recommended |
| **S1** | Low | Monitor |
| **S0** | Info | Observation only |

## Project Structure

```
src/redteam_eval/
├── attacks/            # 12 attack modules (payload definitions)
├── adapters/           # Gateway adapters (e.g. OpenClaw protocol)
├── cli.py              # `redteam-eval` command line
├── harness.py          # EvalHarness orchestration
├── synthetic_gateway.py# Mock/Proxy/Record gateway modes
├── report.py           # Markdown/JSON/SARIF/JUnit reporters
└── tinman_integration.py # Optional failure-analysis integration
attacks/                # YAML-defined supplemental payloads
expected/baseline.json  # Regression baseline
tests/                  # pytest suite
```

## License

Apache-2.0 — see [LICENSE](LICENSE).
