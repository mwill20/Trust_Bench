# Trust Bench Agent Architecture Documentation

## Overview

Trust Bench employs a collaborative multi-agent architecture orchestrated via LangGraph. Each agent has specialized capabilities and actively communicates with peer agents to provide contextual, intelligent repository evaluation. This document provides comprehensive technical specifications for all agents in the system.

---

## Agent Roles Summary

| Agent Name | Primary Role / Responsibility | Inputs Received | Outputs Produced | Tools / Capabilities Used | When It Runs / Trigger |
|------------|-------------------------------|-----------------|------------------|---------------------------|------------------------|
| **Manager (Orchestrator)** | Coordinates overall workflow, routes tasks to agents, tracks collaboration metrics | User configuration, repository path, agent outputs | Task assignments, final composite report, collaboration summary | LangGraph routing, state management, metrics aggregation | Always runs first (plan) and last (finalize) |
| **SecurityAgent** | Scans repository for credentials, secrets, and security vulnerabilities; alerts other agents | Repository path, file contents | Security findings, risk assessments, cross-agent alerts | `run_secret_scan` (regex pattern matching for AWS keys, GitHub tokens, RSA keys, etc.) | After Manager assigns task, before QualityAgent |
| **QualityAgent** | Analyzes code structure, language distribution, test coverage; incorporates security context | Repository path, security findings from SecurityAgent | Quality metrics, adjusted scores, language histogram | `analyze_repository_structure` (file counting, language classification, test detection) | After SecurityAgent completes |
| **DocumentationAgent** | Reviews documentation completeness; evaluates based on quality metrics and security findings | Repository path, quality metrics, security alerts | Documentation scores, improvement recommendations | `evaluate_documentation` (README parsing, section counting, keyword detection) | After QualityAgent completes |

---

## Detailed Agent Specifications

### 🛡️ SecurityAgent

#### Goal & Scope
Identify high-confidence security risks in repository files, including exposed credentials, API keys, and private keys. Provides foundational security context that influences downstream agent assessments.

**In scope:**
- Scanning text files for credential patterns
- Detecting AWS access keys, GitHub tokens, Slack tokens, RSA private keys
- Providing exact file locations and code snippets
- Alerting other agents about security findings

**Out of scope:**
- Dynamic code execution or runtime analysis
- Binary file inspection
- Network-based vulnerability scanning
- Third-party dependency security audits (e.g., npm audit, pip check)

#### Inputs

| Input | Type | Description | Constraints |
|-------|------|-------------|-------------|
| `repo_root` | `pathlib.Path` | Absolute path to repository directory | Must exist and be readable |
| `shared_memory` | `Dict[str, Any]` | Shared state from orchestrator | Provided by LangGraph state |
| `messages` | `List[Message]` | Conversation history | Read-only during execution |

#### Outputs

| Output | Type | Description |
|--------|------|-------------|
| `agent_results["SecurityAgent"]` | `AgentResult` | Contains score (0-100), summary, and detailed findings |
| `shared_memory["security_findings"]` | `List[Dict]` | List of security matches with file paths and snippets |
| `shared_memory["security_context"]` | `Dict` | Risk level (low/medium/high), findings count, attention flag |
| `messages` (appended) | `List[Message]` | Updates to other agents (QualityAgent, DocumentationAgent) |

**Output Schema (AgentResult):**
```python
{
    "score": float,  # 0-100, reduced by 20 per finding
    "summary": str,  # e.g., "Scanned 58 files and detected 6 potential secret hit(s)."
    "details": {
        "matches": [
            {
                "file": str,      # Relative path to file
                "pattern": str,   # Pattern name (e.g., "AWS Access Key")
                "snippet": str    # 80-char context around match
            }
        ],
        "scanned": int  # Total files scanned
    }
}
```

#### Capabilities
- Can scan up to 1.5 MB per file (configurable via `max_file_mb`)
- Detects 4 high-signal credential patterns (AWS, GitHub, Slack, RSA)
- Automatically excludes common build/cache directories (`.git`, `.venv`, `__pycache__`, `node_modules`)
- Provides 40-character context before/after each match
- **Proactive communication:** Immediately alerts QualityAgent and DocumentationAgent when findings exist

#### Limitations
- **Pattern-based detection only:** May miss obfuscated or encoded credentials
- **No semantic analysis:** Cannot detect logical security flaws (SQL injection, XSS, etc.)
- **File size limits:** Skips files larger than 1.5 MB by default
- **Text files only:** Binary files are ignored
- **False positives possible:** May flag test fixtures or example credentials
- **No historical analysis:** Only scans current repository state, not git history

#### Example Input/Output

**Input:**
```python
repo_root = Path("c:/Projects/Trust_Bench")
```

**Output (with findings):**
```python
{
    "score": 0.0,
    "summary": "Scanned 58 files and detected 6 potential secret hit(s).",
    "details": {
        "matches": [
            {
                "file": "Project2v2/test_repo/secrets.txt",
                "pattern": "AWS Access Key",
                "snippet": "AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE\\nAWS_SECRET_ACCESS_KEY=..."
            }
        ],
        "scanned": 58
    }
}
```

**Cross-Agent Messages Sent:**
- To QualityAgent: `"FYI: Found 6 security issues that may impact quality assessment."`
- To DocumentationAgent: `"Alert: 6 security findings detected - please check if docs address security practices."`

#### Failure Behavior
- If file reading fails (permissions, encoding errors), file is silently skipped
- If no files can be scanned, returns score of 100 with summary indicating no scan occurred
- Malformed regex patterns would cause immediate failure (not expected in current implementation)

---

### 🔍 QualityAgent

#### Goal & Scope
Assess repository structure, language diversity, and test coverage. Adjust quality scores based on security context provided by SecurityAgent.

**In scope:**
- Counting total files and categorizing by language
- Detecting test files via naming conventions
- Calculating test-to-code ratios
- Applying security-based score penalties
- Providing metrics for DocumentationAgent

**Out of scope:**
- Static analysis (linting, complexity metrics)
- Code formatting checks
- Performance profiling
- Dependency version checks

#### Inputs

| Input | Type | Description | Constraints |
|-------|------|-------------|-------------|
| `repo_root` | `pathlib.Path` | Absolute path to repository | Must exist |
| `shared_memory["security_findings"]` | `List[Dict]` | Security findings from SecurityAgent | Optional; empty list if none |
| `shared_memory["security_context"]` | `Dict` | Risk level and metadata | Optional |

#### Outputs

| Output | Type | Description |
|--------|------|-------------|
| `agent_results["QualityAgent"]` | `AgentResult` | Quality score (adjusted), summary, language histogram |
| `shared_memory["language_histogram"]` | `Dict[str, int]` | File counts per language |
| `shared_memory["quality_metrics"]` | `Dict` | Total files, test ratio, security adjustment flag |
| `messages` (appended) | `List[Message]` | Collaboration acknowledgment to SecurityAgent and Manager |

**Output Schema:**
```python
{
    "score": float,  # 0-100, adjusted down by up to 25 points for security findings
    "summary": str,  # e.g., "Indexed 58 files across 5 language group(s); 0 appear to be tests. (Adjusted for 6 security findings)"
    "details": {
        "language_histogram": {
            "python": int,
            "markdown": int,
            "yaml": int,
            "other": int
        },
        "total_files": int,
        "test_files": int,
        "test_ratio": float  # 0.0 to 1.0
    }
}
```

#### Capabilities
- Classifies files by extension into Python, TypeScript, JavaScript, Markdown, YAML, JSON, or "other"
- Detects test files by directory name (`tests/`, `test/`) or filename prefix (`test_*.py`)
- Calculates quality score: 40% language diversity + 60% test coverage
- **Collaborative scoring:** Reduces score by 5 points per security finding (max 25-point penalty)
- Sends direct acknowledgment to SecurityAgent when incorporating findings

#### Limitations
- **Naming convention dependent:** Only detects tests following `test_*` or `tests/` patterns
- **No execution:** Cannot verify tests actually run or pass
- **Superficial metrics:** File count and language diversity don't measure code quality
- **Limited language support:** Only recognizes 6 language categories explicitly
- **No complexity analysis:** Doesn't assess cyclomatic complexity, duplication, or maintainability

#### Example Input/Output

**Input:**
```python
repo_root = Path("c:/Projects/Trust_Bench")
security_findings = [...]  # 6 findings from SecurityAgent
```

**Output:**
```python
{
    "score": 35.0,  # Base 60 - 25 (security penalty)
    "summary": "Indexed 58 files across 5 language group(s); 0 appear to be tests. (Adjusted for 6 security findings)",
    "details": {
        "language_histogram": {"python": 18, "markdown": 12, "yaml": 3, "json": 5, "other": 20},
        "total_files": 58,
        "test_files": 0,
        "test_ratio": 0.0
    }
}
```

**Message Sent:**
- To SecurityAgent: `"Incorporated your 6 security findings into quality assessment."`

#### Failure Behavior
- If no files found, returns score of 20 (minimal diversity penalty)
- If file stats cannot be read (permissions), file is skipped
- If security findings are malformed, assumes empty list and proceeds without penalty

---

### 📝 DocumentationAgent

#### Goal & Scope
Evaluate documentation completeness, structure, and contextual relevance based on repository characteristics discovered by other agents.

**In scope:**
- Parsing README files for word count and section structure
- Detecting presence of quickstart and architecture sections
- Adjusting scores based on project size (from QualityAgent)
- Penalizing docs that don't address security or testing gaps
- Providing improvement recommendations

**Out of scope:**
- Grammar/spelling checks
- API documentation validation
- Link checking
- Markdown linting beyond basic structure

#### Inputs

| Input | Type | Description | Constraints |
|-------|------|-------------|-------------|
| `repo_root` | `pathlib.Path` | Absolute path to repository | Must exist |
| `shared_memory["quality_metrics"]` | `Dict` | File counts, test ratios from QualityAgent | Optional |
| `shared_memory["security_findings"]` | `List[Dict]` | Security issues from SecurityAgent | Optional |

#### Outputs

| Output | Type | Description |
|--------|------|-------------|
| `agent_results["DocumentationAgent"]` | `AgentResult` | Documentation score (adjusted), summary, metadata |
| `shared_memory["documentation"]` | `Dict` | README paths, word counts, collaboration adjustments |
| `messages` (appended) | `List[Message]` | Collaboration messages to QualityAgent and SecurityAgent |

**Output Schema:**
```python
{
    "score": float,  # 0-100, adjusted for project context
    "summary": str,  # e.g., "Found 3 README file(s) with roughly 2500 words and 15 section heading(s). (Adjusted based on quality metrics)"
    "details": {
        "readme_paths": List[str],
        "total_words": int,
        "sections": int,  # Count of ## headings
        "has_quickstart": bool,
        "has_architecture": bool
    }
}
```

#### Capabilities
- Scans for all `README*.md` files in repository root
- Counts words and `## ` section headings
- Detects "quick start" and "architecture" keywords
- Applies logarithmic scaling for word count (diminishing returns after 2000+ words)
- **Contextual penalties:**
  - Large projects (100+ files) with weak docs: -10 points
  - No tests detected but no testing docs: -5 points
  - Security issues with no security guidance: -5 points
- Sends collaboration acknowledgments to QualityAgent and SecurityAgent

#### Limitations
- **README-only:** Does not evaluate inline code comments, API docs, or wikis
- **Keyword-based:** Detection of sections like "quickstart" is simple substring matching
- **No quality assessment:** Doesn't judge clarity, accuracy, or usefulness of content
- **English-centric:** May miss non-English documentation
- **No versioning:** Doesn't track documentation staleness or version alignment

#### Example Input/Output

**Input:**
```python
repo_root = Path("c:/Projects/Trust_Bench")
quality_metrics = {"total_files": 58, "test_ratio": 0.0}
security_findings = [...]  # 6 findings
```

**Output:**
```python
{
    "score": 35.0,  # Base 45 - 5 (no tests) - 5 (security gap)
    "summary": "Found 3 README file(s) with roughly 2500 words and 15 section heading(s). (Adjusted based on quality metrics)",
    "details": {
        "readme_paths": ["README.md", "Project2v2/README.md", "Project2v2/READMEV2.md"],
        "total_words": 2500,
        "sections": 15,
        "has_quickstart": True,
        "has_architecture": True
    }
}
```

**Messages Sent:**
- To QualityAgent: `"Used your metrics (files: 58, test ratio: 0.0%) to enhance documentation assessment."`
- To SecurityAgent: `"Noted your 6 findings - documentation lacks security guidance."`

#### Failure Behavior
- If no README files found, returns score based on repository size (penalized)
- If README cannot be read (encoding issues), file is skipped
- Malformed quality_metrics or security_findings treated as empty

---

### 🤖 Manager (Orchestrator)

#### Goal & Scope
Coordinate the multi-agent workflow, track collaboration metrics, aggregate results, and produce final composite assessment.

**In scope:**
- Defining task sequence and agent dependencies
- Tracking cross-agent messages and collaboration events
- Computing composite scores and grades
- Collecting instrumentation metrics (latency, faithfulness, refusal accuracy)
- Generating final reports (JSON and Markdown)

**Out of scope:**
- Performing direct repository analysis
- Making domain-specific judgments (delegates to specialist agents)
- Modifying agent scores (only aggregates)

#### Inputs

| Input | Type | Description |
|-------|------|-------------|
| `repo_root` | `pathlib.Path` | Repository to evaluate | From CLI/web UI |
| Initial `MultiAgentState` | State dict | Empty state with repo_root | Constructed by main.py |

#### Outputs

| Output | Type | Description |
|--------|------|-------------|
| `report` | `Dict[str, Any]` | Composite summary with overall score and grade |
| `metrics` | `Dict[str, Any]` | System latency, faithfulness, refusal accuracy, per-agent timings |
| `messages` (final) | `List[Message]` | Complete conversation log including collaboration summary |
| `shared_memory["composite_assessment"]` | `Dict` | Final evaluation summary |
| `shared_memory["collaboration_summary"]` | `Dict` | Interaction count and adjustment descriptions |

**Composite Score Calculation:**
```python
overall_score = mean([SecurityAgent.score, QualityAgent.score, DocumentationAgent.score])
grade = "excellent" if score >= 85 else "good" if score >= 70 else "fair" if score >= 50 else "needs_attention"
```

#### Capabilities
- **Task Planning:** Defines agent execution order (Security → Quality → Documentation)
- **Timing Instrumentation:** Records start/end times for each agent and tool call
- **Faithfulness Scoring:** Compares agent summaries to their detail tokens (0.0-1.0 scale)
- **Refusal Testing:** Runs simulated unsafe prompt tests (currently returns 1.0 deterministically)
- **Collaboration Tracking:** Counts cross-agent messages (sender ≠ Manager, recipient ≠ Manager)
- **Report Generation:** Formats JSON and Markdown outputs with full conversation logs

#### Limitations
- **Sequential execution only:** Agents run in fixed order (no parallel execution)
- **Simple score aggregation:** Uses mean; doesn't weight by importance
- **No retry logic:** If an agent fails, workflow terminates
- **Deterministic refusal tests:** Not yet integrated with real LLM safety harness

#### Failure Behavior
- If any agent raises an exception, the entire workflow fails
- If metrics cannot be calculated, defaults to 0.0 or "n/a"
- If report writing fails, error is propagated to caller

---

## Agent Collaboration Architecture

### Communication Patterns

Trust Bench agents use **direct message passing** via the `messages` list in shared state. Messages have the following structure:

```python
{
    "sender": str,       # Agent name
    "recipient": str,    # Target agent or "All Agents"
    "content": str,      # Human-readable message
    "data": Dict[str, Any]  # Structured payload
}
```

### Collaboration Flows

#### 1. SecurityAgent → QualityAgent
**Purpose:** Alert about security findings to influence quality assessment

**Message Example:**
```python
{
    "sender": "SecurityAgent",
    "recipient": "QualityAgent",
    "content": "FYI: Found 6 security issues that may impact quality assessment.",
    "data": {
        "security_context": {
            "findings_count": 6,
            "risk_level": "high"
        }
    }
}
```

**Impact:** QualityAgent reduces score by up to 25 points (5 per finding)

#### 2. SecurityAgent → DocumentationAgent
**Purpose:** Alert about security gaps in documentation

**Message Example:**
```python
{
    "sender": "SecurityAgent",
    "recipient": "DocumentationAgent",
    "content": "Alert: 6 security findings detected - please check if docs address security practices.",
    "data": {
        "security_alert": {
            "findings_count": 6,
            "risk_level": "high"
        }
    }
}
```

**Impact:** DocumentationAgent may reduce score by 5 points if docs lack security guidance

#### 3. QualityAgent → DocumentationAgent
**Purpose:** Share repository metrics for contextualized documentation assessment

**Message Example:**
```python
{
    "sender": "QualityAgent",
    "recipient": "DocumentationAgent",
    "content": "Repository structure completed. No security issues found - maintaining base quality score.",
    "data": {
        "quality_metrics": {
            "total_files": 58,
            "test_ratio": 0.0,
            "adjusted_for_security": False
        }
    }
}
```

**Impact:** DocumentationAgent adjusts scores based on project size and test coverage

#### 4. All Agents → Manager
**Purpose:** Report completion and results

**Impact:** Manager aggregates into final composite assessment

### Collaboration Metrics

The Manager tracks:
- **Total cross-communications:** Count of agent-to-agent messages (excludes Manager)
- **Collaborative adjustments:** List of score modifications based on peer insights
- **Influence graph:** Which agents influenced which other agents' scores

Example collaboration summary:
```json
{
    "total_interactions": 5,
    "collaborative_adjustments": [
        "Security findings influenced quality and documentation scores",
        "Quality assessment incorporated security analysis",
        "Documentation score adjusted based on quality metrics"
    ]
}
```

---

## Tool-Agent Mapping

| Tool Name | Function | Used By | Purpose |
|-----------|----------|---------|---------|
| `run_secret_scan` | Scans files for credential patterns using regex | SecurityAgent | Detect exposed secrets/keys |
| `analyze_repository_structure` | Counts files, classifies languages, detects tests | QualityAgent | Measure code organization and test coverage |
| `evaluate_documentation` | Parses README files, counts sections and words | DocumentationAgent | Assess documentation completeness |
| `serialize_tool_result` | Converts ToolResult dataclasses to dicts | All agents | Enable JSON serialization for message passing |

---

## Instrumentation & Metrics

All agents are instrumented with:
- **Execution timing:** Per-agent and per-tool elapsed time (seconds)
- **Faithfulness:** Semantic alignment between summaries and detailed findings
- **Tool invocation logs:** Stored in `shared_memory["timings"]`

Example timing output:
```json
{
    "SecurityAgent": {
        "total_seconds": 0.07,
        "tool_breakdown": {
            "run_secret_scan": 0.065
        }
    },
    "QualityAgent": {
        "total_seconds": 0.003,
        "tool_breakdown": {
            "analyze_repository_structure": 0.002
        }
    }
}
```

---

## Extension Points

To add a new agent:
1. Define agent function in `multi_agent_system/agents.py` following the signature pattern
2. Add tool implementation (if needed) in `multi_agent_system/tools.py`
3. Update `build_orchestrator()` in `orchestrator.py` to include new node and edges
4. Document the agent in this file with full specification
5. Update collaboration flows if the agent needs cross-agent communication

---

## References

- **Implementation:** `Project2v2/multi_agent_system/agents.py`
- **Orchestration:** `Project2v2/multi_agent_system/orchestrator.py`
- **Tool Definitions:** `Project2v2/multi_agent_system/tools.py`
- **Type Definitions:** `Project2v2/multi_agent_system/types.py`
- **Main Entry Point:** `Project2v2/main.py`
- **Web Interface:** `Project2v2/web_interface.py`

For usage examples and user-facing documentation, see:
- Root README: `../README.md`
- User Guide: `../Project2v2/USER_GUIDE.md`
- Collaboration Summary: `../Project2v2/COLLABORATION_SUMMARY.md`
