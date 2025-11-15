<p align="center">
  <img src="Project2v2/assets/images/TrustBench.png" alt="Trust Bench logo" width="200" />
</p>

# Trust Bench - Multi-Agent Security Evaluation Framework [![Version](https://img.shields.io/badge/version-Project2v2-blue)](#) [![Watch the demo](https://img.shields.io/badge/watch%20the%20demo-%F0%9F%8E%A5-ff5722)](https://1drv.ms/v/c/2c8c41c39e8a63cb/EQil7I1iewdEuzPdQGBqVOYBHaRg9tBcyogZvmKUXKFLyw?e=8Dl8P5)

**Official Publication:** [Trust Bench: Multi-Agent Security Evaluation Framework](https://app.readytensor.ai/publications/trust-bench-multi-agent-security-evaluation-framework-F6PS953ZZuo5)

Trust Bench (Project2v2) is a LangGraph-based multi-agent workflow that inspects software repositories for security leakage, code quality gaps, and documentation health. The system emphasizes cross-agent collaboration, transparent reasoning, and reproducible outputs that graders can run entirely offline.

## Contents
1. [Overview](#overview)
2. [Agent Architecture](docs/AGENTS.md) 📘
3. [Operations & Error Handling](docs/OPERATIONS.md) 🔧
4. [Tool Integrations](#tool-integrations)
5. [Installation & Setup](#installation--setup)
6. [Usage / Quickstart](#usage--quickstart)
7. [Human-in-the-Loop Workflow](#human-in-the-loop-workflow)
8. [Evaluation Metrics Instrumentation](#evaluation-metrics-instrumentation)
9. [Demo Video](#demo-video)
10. [Example Results (Project2v2 self-audit)](#example-results-project2v2-self-audit)
11. [MCP Server (Scope Decision)](#mcp-server-scope-decision)
12. [File Structure (trimmed)](#file-structure-trimmed)
13. [Security & Hardening Notes](#security--hardening-notes)
14. [Maintenance & Support](#maintenance--support)
15. [Credits & References](#credits--references)
16. [License](#license)

---

## Overview

- **Agents**: Manager (plan/finalize), SecurityAgent, QualityAgent, DocumentationAgent  
- **Core Tools**: regex secret scanner, repository structure analyzer, documentation reviewer  
- **Collaboration**: agents exchange messages and adjust scores based on peer findings (security alerts penalize quality/documentation; quality metrics influence documentation, etc.)  
- **Deliverables**: JSON and Markdown reports containing composite scores, agent summaries, conversation logs, and instrumentation metrics

```
[Manager Plan]
     |
[SecurityAgent] --> alerts --> [QualityAgent] --> metrics --> [DocumentationAgent]
     \____________________________ shared context _____________________________/
                           |
                   [Manager Finalize] --> report.json / report.md
```

---

## Tool Integrations

| Tool | Consumed By | Capability Extension |
|------|-------------|----------------------|
| `run_secret_scan` | SecurityAgent | Detects high-signal credentials (AWS, GitHub, RSA keys) |
| `analyze_repository_structure` | QualityAgent | Counts files, languages, estimated test coverage |
| `evaluate_documentation` | DocumentationAgent | Scores README variants by coverage and cross-agent context |
| `serialize_tool_result` | All agents | Normalizes tool dataclasses for message passing |

> MCP endpoints are intentionally **not** shipped in Project2v2. See [MCP Server (Scope Decision)](#mcp-server-scope-decision).

---

## Installation & Setup

### Prerequisites

- **Python:** 3.10 or higher (`python --version` to check)
- **Operating System:** Windows 10/11, macOS, or Linux
- **Hardware:** CPU-only (no GPU required)
- **Git:** Required only for web interface GitHub cloning (optional for CLI)
- **Disk Space:** ~50 MB for dependencies + space for repository clones

### Installation Steps

```powershell
# 1. Clone the repository
git clone https://github.com/mwill20/Trust_Bench.git
cd Trust_Bench

# 2. Create virtual environment (recommended)
python -m venv .venv
.\.venv\Scripts\activate          # Windows PowerShell
# OR: source .venv/bin/activate   # macOS/Linux

# 3. Install core dependencies
pip install -r Project2v2/requirements-phase1.txt

# 4. (Optional) Install extras for advanced features
pip install -r Project2v2/requirements-optional.txt
# Extras include: ragas (evaluation), semgrep (static analysis), streamlit (dashboards)

# 5. (Optional) Configure environment variables
copy Project2v2\.env_example .env  # Windows
# OR: cp Project2v2/.env_example .env  # macOS/Linux
# Edit .env file to add API keys if using web chat feature
```

### Verify Installation

Run this sanity check to confirm everything is working:

```powershell
cd Project2v2
python main.py --repo test_repo --output verification_test
```

**Expected output:**
```
=== Multi-Agent Evaluation Complete ===
Repository: <path>\test_repo
Overall Score: 52.67
Grade: fair
System Latency: 0.02 seconds
...
Report (JSON): verification_test\report.json
Report (Markdown): verification_test\report.md
```

If you see the above output, Trust Bench is correctly installed!

### Offline Operation

**Trust Bench runs completely offline by default.** All agent scoring is deterministic and does not require API keys. Leave `.env` variables empty if you don't need the optional web UI chat feature. The core evaluation workflow works without any external services.

### Environment Variables Reference

**All variables are optional:**

| Variable | Default | Purpose | Required For |
|----------|---------|---------|-------------|
| `ENABLE_SECURITY_FILTERS` | `true` | Enable input validation | Web UI (recommended) |
| `LLM_PROVIDER` | `openai` | LLM provider selection | Web UI chat only |
| `OPENAI_API_KEY` | (empty) | OpenAI API access | Web UI chat only |
| `GROQ_API_KEY` | (empty) | Groq API access | Web UI chat only |
| `GEMINI_API_KEY` | (empty) | Gemini API access | Web UI chat only |
| `TRUST_BENCH_WORKDIR` | `./output` | Default output location | (informational) |

**Note:** Agents do **not** call external APIs. LLM integration is only for the optional web interface chat feature.

---

## Usage / Quickstart

### Default Evaluation (Simplest)

Evaluate the current directory and save results to `output/`:

```powershell
cd Project2v2
python main.py --repo . --output output
```

**What happens:**
1. SecurityAgent scans for credentials and secrets
2. QualityAgent analyzes code structure and test coverage
3. DocumentationAgent reviews README files
4. Manager aggregates results and generates reports

**Outputs created:**
- `output/report.json` - Detailed JSON with all agent results, metrics, conversation log
- `output/report.md` - Human-readable Markdown summary

### Command-Line Interface (CLI)

**Basic syntax:**
```powershell
python main.py --repo <path_to_repository> --output <output_directory>
```

**Examples:**
```powershell
# Analyze parent directory
python main.py --repo .. --output parent_analysis

# Analyze specific repository
python main.py --repo "C:\Projects\MyRepo" --output myrepo_results

# Analyze test repository
python main.py --repo test_repo --output test_results
```

**Command-line arguments:**
- `--repo` (required): Path to repository to evaluate (relative or absolute)
- `--output` (optional): Output directory for reports (default: `multi_agent_output`)

### Web Interface (Recommended for Interactive Use)

**Start the web server:**
```powershell
cd Project2v2
python web_interface.py
```

**Then open your browser to:** `http://localhost:5000`

**Features:**
- Clone and analyze GitHub repositories directly from URL
- Select local repository paths via dropdown presets
- Real-time agent progress visualization
- Color-coded scoring (excellent/good/fair/needs_attention)
- Interactive conversation log viewer
- Optional LLM-powered Q&A about reports

**Stopping the server:** Press `Ctrl+C` in terminal

### Convenience Scripts

**PowerShell (Windows):**
```powershell
cd Project2v2
.\run_audit.ps1 <repo_path> [output_dir]

# Examples:
.\run_audit.ps1 .                    # Current directory
.\run_audit.ps1 .. parent_results    # Parent with custom output
```

**Batch (Windows CMD):**
```batch
cd Project2v2
run_audit.bat <repo_path> [output_dir]

# Examples:
run_audit.bat .
run_audit.bat .. parent_results
run_audit.bat "C:\path\to\repo"
```

**Interactive launcher:**
```batch
cd Project2v2
launch.bat
```

Provides a menu with options for:
1. CLI mode with examples
2. Web interface launcher
3. Quick analysis presets
4. Exit

### Legacy CLI (Backward Compatibility)

```powershell
python -m trustbench_core.eval.evaluate_agent --repo <path> --output <dir>
```

This command forwards to `Project2v2/main.py`. The new entrypoint (`main.py`) is the single source of truth.

---

## Report Outputs

### Output Files

Each evaluation produces two report files:

#### 1. `report.json` (Machine-Readable)

**Structure:**
```json
{
    "generated_at": "2025-11-15T10:30:00.123456+00:00",
    "repo_root": "/absolute/path/to/repository",
    "summary": {
        "overall_score": 61.33,
        "grade": "fair",
        "notes": "Composite built from agent contributions."
    },
    "agents": {
        "SecurityAgent": {
            "score": 80.0,
            "summary": "Scanned 58 files and detected 0 potential secret hit(s).",
            "details": {"matches": [], "scanned": 58}
        },
        "QualityAgent": {...},
        "DocumentationAgent": {...}
    },
    "metrics": {
        "system_latency_seconds": 0.08,
        "faithfulness": 0.62,
        "refusal_accuracy": 1.0,
        "per_agent_latency": {...}
    },
    "conversation": [
        {
            "sender": "Manager",
            "recipient": "SecurityAgent",
            "content": "Task assigned: Scan the repository for high-signal secrets...",
            "data": {...}
        },
        ...
    ]
}
```

**Use for:**
- Automated processing and analysis
- Integration with CI/CD pipelines
- Tracking metrics over time
- Debugging agent behavior via conversation log

#### 2. `report.md` (Human-Readable)

**Structure:**
```markdown
# Trust Bench Multi-Agent Evaluation Report
- Generated at: 2025-11-15T10:30:00
- Repository: /path/to/repo

## Composite Summary
{overall_score, grade, collaboration metrics}

## Agent Findings
### SecurityAgent
- Score: 80
- Summary: ...

### QualityAgent
...

## Evaluation Metrics
| Metric | Value |
...

## Conversation Log
- Manager -> SecurityAgent: ...
...
```

**Use for:**
- Quick manual review
- Sharing results with team members
- Documentation and audit trails
- Understanding agent collaboration

### Output Locations

**CLI mode:**
- Reports written to directory specified by `--output` argument
- Default: `multi_agent_output/` if not specified

**Web interface:**
- Timestamped directories: `github_analysis_<repo>_<timestamp>/`
- Also copies to `output/` for consistent location

**Convenience scripts:**
- Default: `audit_output/` (configurable via second argument)

### Reading Reports

**View Markdown in terminal:**
```powershell
Get-Content output\report.md  # PowerShell
type output\report.md          # CMD
cat output/report.md           # macOS/Linux
```

**View JSON programmatically:**
```python
import json
from pathlib import Path

report = json.loads(Path("output/report.json").read_text())
print(f"Overall Score: {report['summary']['overall_score']}")
print(f"Security Score: {report['agents']['SecurityAgent']['score']}")
```

---

## Human-in-the-Loop Workflow

Trust Bench is designed as a **human-in-the-loop security evaluation tool** where analysts maintain control over the assessment process while leveraging AI agent capabilities.

### Human Decision Points

#### 1. Repository Selection
**When:** Before evaluation starts  
**Human choice:**
- Select which repository to analyze (local path or GitHub URL)
- Decide on evaluation scope (full repo vs. specific subdirectory)
- Choose whether to use web interface or CLI

**Example workflow:**
```
Analyst sees suspicious project → Decides to audit → Provides repo path to Trust Bench
```

#### 2. Interface Selection
**When:** At invocation  
**Human choice:**
- **Web UI:** For interactive exploration, GitHub cloning, visual progress
- **CLI:** For automation, scripting, CI/CD integration, batch processing
- **Convenience scripts:** For quick one-off evaluations

**Decision factors:**
- Interactivity needed? → Web UI
- Automation/scripting? → CLI
- One-time quick check? → Convenience scripts

#### 3. Report Review & Analysis
**When:** After evaluation completes  
**Human action:**
- Read generated `report.md` for high-level summary
- Examine agent conversation logs to understand collaborative reasoning
- Review specific findings (e.g., SecurityAgent's detected secrets)
- Assess whether scores align with expectations
- Identify areas requiring deeper investigation

**Example questions analysts ask:**
- Why did the security score drop to 0?
- What specific files contain credentials?
- Why is documentation scored lower than expected?
- How did agents collaborate on this assessment?

#### 4. Configuration Adjustments
**When:** Based on initial results  
**Human choice:**
- Enable/disable security filters (`ENABLE_SECURITY_FILTERS`)
- Adjust file size limits (requires code modification)
- Add custom secret patterns (requires code modification)
- Tune scoring thresholds (requires code modification)

**Example scenario:**
```
Initial run flags test fixtures as secrets → Analyst reviews → Decides legitimate →
Adds exception patterns → Re-runs evaluation
```

#### 5. Re-Running with Different Parameters
**When:** When initial results need refinement  
**Human action:**
- Change output directory to preserve previous results
- Target different repository or subdirectory
- Run after fixing identified issues to verify improvements
- Compare results across different commits/branches

**Example workflow:**
```bash
# Initial evaluation
python main.py --repo . --output baseline

# Fix identified issues in code
# ...

# Verify improvements
python main.py --repo . --output after_fixes

# Compare results
diff baseline/report.json after_fixes/report.json
```

#### 6. Follow-Up Investigation (Web UI)
**When:** During or after report review  
**Human action:**
- Use optional LLM chat to ask questions about findings
- Request clarification on specific security issues
- Ask for remediation recommendations
- Explore collaboration patterns between agents

**Example questions:**
- "Why did SecurityAgent flag this file?"
- "What are best practices for fixing these credential leaks?"
- "How did QualityAgent's findings influence DocumentationAgent?"

### Analyst Workflow Example

**Scenario:** Security analyst auditing a new open-source project

1. **Discovery:** Analyst finds project on GitHub that handles sensitive data
2. **Initial scan:** Uses web UI to clone and analyze: `https://github.com/owner/repo`
3. **Review findings:** Reads `report.md`, notes security score of 20 (needs attention)
4. **Deep dive:** Examines `report.json` conversation log to understand why
5. **Identify issues:** SecurityAgent found 4 potential credential leaks
6. **Human judgment:** Analyst reviews flagged files:
   - 2 are test fixtures (acceptable)
   - 2 are actual leaks (critical)
7. **Decision:** Analyst decides to:
   - File security disclosure for real leaks
   - Document test fixtures for future reference
8. **Verification:** After maintainer fixes issues, analyst re-runs:
   ```bash
   python main.py --repo . --output verification_scan
   ```
9. **Confirmation:** Security score improves to 100 (excellent)

### No Mid-Run Confirmations

**Current implementation:** Trust Bench runs **fully automated** once started. There are no mid-run prompts or confirmations.

**Human control points are:**
- **Before:** Repository selection, interface choice, configuration
- **After:** Report review, decision-making, re-running

**No interruptions for:**
- ❌ Confirming each agent execution
- ❌ Approving tool invocations
- ❌ Validating intermediate findings
- ❌ Adjusting parameters mid-flight

**Rationale:** Designed for unattended operation in CI/CD and batch processing scenarios while maintaining human oversight through configuration and post-analysis review.

### Integration with Security Workflows

**Trust Bench fits into analyst workflows as:**

1. **Triage tool:** Quick initial assessment of repository security posture
2. **Baseline establishment:** Record current state before remediation
3. **Verification tool:** Confirm fixes after addressing issues
4. **Continuous monitoring:** Integrate into CI/CD for ongoing evaluation
5. **Audit trail:** Generate documentation for security reviews

**Typical integration points:**
- Pre-deployment security checks
- Open-source dependency audits
- Code review automation
- Security training demonstrations
- Research and academic studies

---

## Evaluation Metrics Instrumentation

Every run records deterministic metrics alongside agent results:

- **System latency** - overall wall-clock time plus per-agent/per-tool timings (`metrics.system_latency_seconds`, `metrics.per_agent_latency`)
- **Faithfulness** - heuristic alignment of summaries with tool evidence (`metrics.faithfulness`)
- **Refusal accuracy** - simulated unsafe prompt harness (returns 1.0 while LLM calls are disabled) (`metrics.refusal_accuracy`)

Metrics appear in both `report.json` (under `metrics`) and `report.md` (rendered table). Example CLI output:

```
System Latency: 0.08 seconds
Faithfulness: 0.62
Refusal Accuracy: 1.0
Per-Agent Timings:
  - SecurityAgent: 0.07 seconds
  - QualityAgent: 0.003 seconds
  - DocumentationAgent: 0.002 seconds
```

---

## Reporting Outputs

Each audit (web or CLI) produces:

- `report.json` - timestamp, repo path, composite summary, per-agent results, metrics, full conversation log  
- `report.md` - human-readable summary with agent cards, instrumentation metrics, conversation log  
- Optional timestamped archives (`github_analysis_*`) when launched through the web interface

---

## Demo Video

<div align="center">
  <video controls width="640" poster="Project2v2/assets/images/TrustBench.png">
    <source src="https://1drv.ms/v/c/2c8c41c39e8a63cb/EQil7I1iewdEuzPdQGBqVOYBHaRg9tBcyogZvmKUXKFLyw?e=8Dl8P5" type="video/mp4" />
    Your browser does not support the video tag. You can download the walkthrough
    <a href="https://1drv.ms/v/c/2c8c41c39e8a63cb/EQil7I1iewdEuzPdQGBqVOYBHaRg9tBcyogZvmKUXKFLyw?e=8Dl8P5">here</a>.
  </video>
</div>

_If playback doesn't work on GitHub, download the file locally from the same link above._

> The full-resolution video is hosted via OneDrive to keep the repository history lean. If you want an offline copy, download it from the link above and place it under `Project2v2/assets/images/`.

---

## Example Results (Project2v2 self-audit)

- Overall Score: ~32/100 (`needs_attention`)  
- Security: seeded secrets detected (score 0) drive collaboration penalties  
- Quality: medium score, automatically penalized by SecurityAgent findings  
- Documentation: strong base score but reduced for missing security/testing guidance  
- Collaboration: more than five cross-agent messages; Manager summarizes adjustments in the final log

---

## MCP Server (Scope Decision)

Project2v2 prioritizes deterministic, offline-capable tooling. To keep grading reproducible and avoid external runtime dependencies, the earlier MCP server has been **intentionally deprecated** for this version. Required tool integrations (three or more) are provided as direct Python callables. MCP can be revisited later if cross-client interoperability (Claude Desktop, Cursor, etc.) becomes necessary, but it is **not required** for Module 2 compliance.

---

## File Structure (trimmed)

```
Trust_Bench/
|-- Project2v2/
|   |-- main.py
|   |-- web_interface.py
|   |-- multi_agent_system/
|   |   |-- agents.py
|   |   |-- orchestrator.py
|   |   |-- tools.py
|   |   `-- reporting.py
|   |-- requirements-phase1.txt
|   |-- requirements-optional.txt
|   |-- run_audit.(bat|ps1)
|   |-- launch.bat
|   `-- output/
|-- trustbench_core/      (legacy CLI wrapper forwarding to Project2v2/main.py)
`-- Project2v2/checklist.yaml (optional-features register)
```

---

## Security & Hardening Notes

- All detected secrets are synthetic and included solely for demonstration purposes. No real credentials are exposed.
- `security_utils.py` and the web UI sanitize repository URLs, prompts, and API keys.
- Optional extras (`ragas`, `semgrep`, `streamlit`) enable deeper analytics and dashboarding when desired.

---

## Maintenance & Support

### Project Status

**Current phase:** Academic/Experimental (Ready Tensor AI Agent Course - Module 2)

**Version:** Project2v2 (October 2025)

**Active development:** This project is maintained as part of academic coursework and ongoing research into multi-agent security evaluation systems.

### Reporting Issues

**Found a bug or have a suggestion?**

1. **Check existing issues:** Visit [GitHub Issues](https://github.com/mwill20/Trust_Bench/issues) to see if already reported
2. **Open new issue:** Click "New Issue" and provide:
   - Clear description of the problem or feature request
   - Steps to reproduce (for bugs)
   - Expected vs. actual behavior
   - System information (Python version, OS)
   - Relevant error messages or logs
3. **Be specific:** Include code snippets, file paths, or screenshots when applicable

**Example bug report:**
```
Title: SecurityAgent fails on files with non-UTF8 encoding

Description:
When analyzing a repository with ISO-8859-1 encoded files,
SecurityAgent crashes instead of skipping the file.

Steps to reproduce:
1. Create file with ISO-8859-1 encoding
2. Run: python main.py --repo . --output test
3. Observe error: UnicodeDecodeError

Expected: File should be skipped silently
Actual: Workflow crashes

System: Python 3.11, Windows 11
```

### Version History & Tagging

**Current versioning approach:**
- Major versions indicated by directory name (e.g., `Project2v2`)
- Git commits track incremental changes
- No formal semantic versioning (yet)

**Future versioning plan:**
- Transition to semantic versioning (e.g., `v1.0.0`, `v1.1.0`)
- Git tags for stable releases
- CHANGELOG.md for release notes

**Checking your version:**
```powershell
cd Trust_Bench
git log -1 --pretty=format:"%H %ci" Project2v2/
```

### Getting Help

**For usage questions:**
1. Read this README thoroughly
2. Check [docs/AGENTS.md](docs/AGENTS.md) for agent specifications
3. Review [docs/OPERATIONS.md](docs/OPERATIONS.md) for error handling
4. Search [closed GitHub Issues](https://github.com/mwill20/Trust_Bench/issues?q=is%3Aissue+is%3Aclosed)
5. Open new issue with "question" label

**For development questions:**
1. Review code documentation in `Project2v2/multi_agent_system/`
2. Examine existing agent implementations
3. Check LangGraph documentation for orchestration patterns
4. Open discussion in GitHub Issues

### Contributing

**Contributions welcome!** This is an academic project, but improvements and extensions are encouraged.

**How to contribute:**
1. Fork the repository
2. Create feature branch: `git checkout -b feature/my-enhancement`
3. Make changes with clear commit messages
4. Add tests if applicable (see `Project2v2/tests/`)
5. Update documentation to reflect changes
6. Submit pull request with description of changes

**Contribution areas:**
- Additional agents (e.g., PerformanceAgent, ComplianceAgent)
- New security patterns for secret detection
- Improved error handling and retry logic
- Additional output formats (PDF, HTML)
- CI/CD integration examples
- Bug fixes and performance improvements

### Roadmap (Potential Future Work)

**Not committed, but under consideration:**

- [ ] Parallel agent execution for faster analysis
- [ ] Configurable scoring weights per agent
- [ ] Structured logging with log levels
- [ ] UUID-based run identifiers
- [ ] Partial result recovery on agent failure
- [ ] Plugin architecture for custom agents
- [ ] REST API for remote evaluations
- [ ] Docker containerization
- [ ] Integration with GitHub Actions
- [ ] Machine learning for adaptive scoring

### Contact

**Project maintainer:** @mwill20  
**Repository:** [github.com/mwill20/Trust_Bench](https://github.com/mwill20/Trust_Bench)  
**Course context:** Ready Tensor AI Agent Course - Module 2  

For academic inquiries or collaboration opportunities, open a GitHub issue or discussion.

---

## License

This project is licensed under the MIT License. See the [LICENSE](LICENSE) file for details.

---

## Credits & References

- Ready Tensor AI Agent Course - Module 2 (Multi-Agent Evaluation)  
- LangGraph, CrewAI, AutoGen (collaboration inspiration)  
- Semgrep, OpenAI/Groq/Gemini APIs (referenced integrations)  
- Project2v2 implementation by @mwill20 and collaborators

_Version Project2v2 - October 2025 - Refactored for offline deterministic evaluation._

