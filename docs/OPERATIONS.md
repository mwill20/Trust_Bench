# Trust Bench Operations Guide

## Error Handling & Resilience

This document describes how Trust Bench handles errors, failures, and edge cases during repository evaluation. All behaviors documented here are based on the actual implementation in `Project2v2/`.

---

## Agent-Level Error Handling

### SecurityAgent (`run_secret_scan`)

**File Reading Errors:**
- **Pattern:** `try/except OSError` blocks in file scanning loop
- **Behavior:** Files that cannot be read (permissions, encoding errors) are **silently skipped**
- **Impact:** Scan continues with remaining files; skipped files don't affect the count
- **Code location:** `Project2v2/multi_agent_system/tools.py`, lines 51-56

```python
try:
    text = file_path.read_text(encoding="utf-8", errors="ignore")
    scanned += 1
except OSError:
    continue  # Silently skip and move to next file
```

**File Size Limits:**
- Files larger than 1.5 MB are skipped (configurable via `max_file_mb` parameter)
- No error is raised; file is excluded from scan count

**Failure Scenarios:**
- **All files unreadable:** Returns score of 100 with summary "Scanned 0 files with no high-confidence secret matches"
- **No patterns match:** Returns score of 100 (clean scan)
- **Regex compilation errors:** Would cause immediate failure (not expected with current patterns)

---

### QualityAgent (`analyze_repository_structure`)

**File Stat Errors:**
- **Behavior:** Files where `stat()` fails are skipped during iteration
- **Impact:** Total file count excludes inaccessible files
- **No explicit try/except:** Relies on `os.walk()` which handles permission errors internally

**Failure Scenarios:**
- **No files found:** Returns score of 20 (minimal diversity penalty)
- **Empty repository:** Valid result with `total_files: 0`
- **Malformed security findings:** Assumes empty list, proceeds without penalty

---

### DocumentationAgent (`evaluate_documentation`)

**README Reading Errors:**
- **Pattern:** `try/except OSError` when reading README files
- **Behavior:** Unreadable READMEs are **silently skipped**
- **Code location:** `Project2v2/multi_agent_system/tools.py`, lines 146-149

```python
try:
    text = readme.read_text(encoding="utf-8")
except OSError:
    continue  # Skip this README
```

**Failure Scenarios:**
- **No README files:** Returns score based on repository size (typically low/penalized)
- **All READMEs unreadable:** Treated same as missing READMEs
- **Malformed quality_metrics or security_findings:** Treated as empty, no crash

---

### Manager (Orchestrator)

**Workflow-Level Errors:**
- **Pattern:** No explicit try/except around agent invocations in LangGraph
- **Behavior:** If any agent raises an unhandled exception, **the entire workflow terminates**
- **Impact:** No partial results; evaluation stops at the failing agent

**Metrics Calculation Errors:**
- Metrics that cannot be calculated default to `0.0` or `"n/a"`
- Example: If `perf_session_started_at` is missing, `system_latency` = 0.0

**Report Writing Errors:**
- Errors during JSON/Markdown file writing propagate to caller
- **No retry logic** for I/O failures

---

## CLI Error Handling

### main.py

**Repository Validation:**
```python
if not repo_root.exists():
    print(f"[error] Repository directory does not exist: {repo_root}")
    return 1  # Exit with error code
```

**Behavior:**
- Pre-flight check: Repository path must exist before workflow starts
- Invalid path results in immediate exit with code `1`
- Error message printed to stdout (no logging)

**Workflow Exceptions:**
- Uncaught exceptions in `run_workflow()` propagate to caller
- No try/except wrapper around `graph.invoke(state)`
- Python stack trace visible to user on failure

**Report Writing Failures:**
- Exceptions in `write_report_outputs()` propagate upward
- Partial output possible if JSON succeeds but Markdown fails

---

## Web Interface Error Handling

### Repository Cloning (`web_interface.py`)

**Timeout Protection:**
```python
result = subprocess.run(cmd, capture_output=True, text=True, timeout=120)
```

- **Hard timeout:** 120 seconds (2 minutes)
- **Behavior on timeout:** Raises `subprocess.TimeoutExpired` exception
- **Error message:** "Repository cloning timed out (120s limit)"

**Git Availability Check:**
```python
git_check = subprocess.run(['git', '--version'], capture_output=True, text=True)
if git_check.returncode != 0:
    raise Exception("Git is not installed or not available in PATH")
```

**Repository Access Errors:**
- Invalid/private repositories: "Repository not found or not accessible"
- Network errors: "Failed to clone repository: <details>"

**Cleanup Behavior:**
- Temporary clone directories are created in system temp
- **No automatic cleanup on error** (orphaned temp directories possible)

### Input Validation (`security_utils.py`)

**URL Validation:**
- Pattern: `validate_repo_url(raw_url)` raises `ValidationError` on invalid input
- Enforced checks:
  - Non-empty URL
  - HTTPS/HTTP scheme required
  - Must be `github.com` or `www.github.com`
  - Must include owner/repo path pattern

**Prompt Sanitization:**
- Pattern: `sanitize_prompt(text)` strips injection attempts
- Behavior: Silently removes suspicious patterns, truncates to 4000 chars
- **No errors raised** - always returns sanitized string

**Security Filter Toggle:**
```python
flag = os.getenv("ENABLE_SECURITY_FILTERS", "true").strip().lower()
```

- Default: **enabled**
- Disable via: `ENABLE_SECURITY_FILTERS=false` (or `0`, `no`, `off`)

---

## Retry & Timeout Policies

### Current Implementation: **No Retries**

**Agent Execution:**
- Each agent runs exactly once in sequential order
- No retry on failure
- No timeout per agent

**Tool Invocations:**
- `run_secret_scan`, `analyze_repository_structure`, `evaluate_documentation` run once
- No internal retry logic for file I/O errors
- Failed file reads are skipped, not retried

**Web Interface:**
- Git clone has 120-second timeout (hard limit)
- No automatic retry on clone failure
- User must manually re-submit

**Future Enhancement Opportunities:**
- Add configurable retry logic for transient I/O errors
- Per-agent timeout limits in LangGraph
- Exponential backoff for network operations
- Partial result recovery on agent failure

---

## Logging & Debugging

### Current Logging: **Print Statements Only**

**CLI Output (`main.py`):**
```python
print("=== Multi-Agent Evaluation Complete ===")
print(f"Repository: {report.get('repo_root')}")
print(f"Overall Score: {summary.get('overall_score', 'n/a')}")
```

**No structured logging:**
- No Python `logging` module used
- No log files written
- No log levels (DEBUG, INFO, WARN, ERROR)
- All output to stdout

**Web Interface:**
```python
print("🚀 Starting Trust Bench Multi-Agent Auditor Web Interface...")
print("🌐 Open your browser to: http://localhost:5000")
```

**Traceability:**
- **Run identification:** Each report includes `"generated_at"` timestamp (ISO 8601 UTC)
- **No run_id:** Reports don't have unique run identifiers
- **Conversation logs:** Full agent message history stored in `report.json` under `"conversation"`
- **Timing data:** Per-agent and per-tool execution times in `metrics.per_agent_latency`

**Debugging Workflow:**
1. Check terminal output for immediate errors
2. Review `report.json` for complete agent conversation
3. Examine `metrics.per_agent_latency` for performance bottlenecks
4. Inspect `report.md` for human-readable summary

---

## Failure Modes & Recovery

### Scenario 1: Repository Path Invalid

**Trigger:** Non-existent path passed to `--repo`

**Behavior:**
```
[error] Repository directory does not exist: /invalid/path
```

**Exit code:** `1`

**Recovery:** Provide valid repository path

---

### Scenario 2: Agent Crashes Mid-Workflow

**Trigger:** Unhandled exception in SecurityAgent, QualityAgent, or DocumentationAgent

**Behavior:**
- LangGraph workflow terminates immediately
- Python traceback printed to stdout
- No report generated (no JSON/Markdown output)

**Recovery:**
- Fix underlying issue (e.g., file permissions, missing dependencies)
- Re-run evaluation from start

---

### Scenario 3: File Permissions Error During Scan

**Trigger:** Repository contains unreadable files

**Behavior:**
- SecurityAgent: Silently skips unreadable files, continues with rest
- QualityAgent: Unreadable files excluded from count
- DocumentationAgent: Unreadable READMEs skipped

**Result:** Evaluation completes with potentially incomplete data

**Recovery:** Grant read permissions to repository files, re-run

---

### Scenario 4: Git Clone Timeout (Web UI)

**Trigger:** Large repository or slow network

**Behavior:**
```json
{
    "error": "Repository cloning timed out (120s limit)"
}
```

**HTTP status:** 500 (Internal Server Error)

**Recovery:**
- Use local repository with CLI mode instead
- Increase timeout in `web_interface.py` (requires code change)
- Clone repository manually, then use CLI

---

### Scenario 5: Report Writing Fails

**Trigger:** Output directory not writable, disk full

**Behavior:**
- Exception propagates from `write_report_outputs()`
- May result in partial output (e.g., JSON written but Markdown fails)

**Recovery:**
- Ensure output directory is writable
- Free disk space
- Specify different output directory with `--output`

---

## Configuration & Environment Variables

### Required: None

All environment variables are **optional**. The system runs deterministically without any API keys.

### Optional Variables

| Variable | Default | Purpose |
|----------|---------|---------|
| `ENABLE_SECURITY_FILTERS` | `true` | Enable/disable input validation in web UI |
| `LLM_PROVIDER` | `openai` | LLM provider for web chat (not used in agents) |
| `OPENAI_API_KEY` | (empty) | API key for OpenAI (web UI only) |
| `GROQ_API_KEY` | (empty) | API key for Groq (web UI only) |
| `GEMINI_API_KEY` | (empty) | API key for Gemini (web UI only) |
| `TRUST_BENCH_WORKDIR` | `./output` | Default output directory (informational) |

**Note:** Agents do **not** make LLM calls. All scoring is deterministic based on tool outputs. LLM integration is only used for the optional web UI chat feature.

---

## Operational Best Practices

### Pre-Flight Checks

Before running evaluation:
1. ✅ Verify repository path exists: `ls <path>` or `dir <path>`
2. ✅ Ensure read permissions on repository files
3. ✅ Confirm Python 3.10+ installed: `python --version`
4. ✅ Check dependencies installed: `pip list | grep langgraph`

### Running Evaluations

**For production/automated use:**
```powershell
python main.py --repo /path/to/repo --output /path/to/output 2>&1 | Tee-Object -FilePath audit.log
```

**For development/debugging:**
```powershell
python -u main.py --repo . --output debug_output
```

(The `-u` flag forces unbuffered output for real-time progress)

### Interpreting Failures

**Exit code 0:** Success, reports written
**Exit code 1:** Pre-flight validation failed (bad repo path)
**Exit code >1:** Unexpected error (check traceback)

### Data Recovery

If workflow crashes mid-run:
- No partial reports saved
- Re-run required from start
- Use smaller repository to isolate issues

---

## Monitoring & Observability

### Metrics Available in Reports

Every successful run produces:

```json
{
    "metrics": {
        "system_latency_seconds": 0.08,
        "faithfulness": 0.62,
        "refusal_accuracy": 1.0,
        "per_agent_latency": {
            "SecurityAgent": {
                "total_seconds": 0.07,
                "tool_breakdown": {"run_secret_scan": 0.065}
            }
        }
    }
}
```

**Use cases:**
- Performance monitoring: Track `system_latency_seconds` across runs
- Agent bottlenecks: Identify slow agents via `per_agent_latency`
- Quality assurance: Monitor `faithfulness` score over time

### Run Identification

Each report includes:
- `"generated_at"`: ISO 8601 timestamp (UTC)
- `"repo_root"`: Absolute path to evaluated repository
- **No unique run_id:** Identify runs by timestamp + repo path

**Future enhancement:** Add UUID-based `run_id` for unambiguous tracking

---

## Known Limitations

1. **No parallel execution:** Agents run sequentially (Security → Quality → Documentation)
2. **No partial results:** Workflow failure = no output
3. **Silent file skipping:** Unreadable files excluded without warning
4. **No structured logging:** Only print statements, no log files
5. **No retry logic:** I/O errors require manual re-run
6. **Single-threaded:** One repository evaluation at a time
7. **No progress indicators:** CLI provides no real-time status updates
8. **Memory-bound:** Large repositories (>10,000 files) may cause performance issues

---

## Failure Recovery Checklist

When evaluation fails:

- [ ] Check terminal output for error message
- [ ] Verify repository path exists and is readable
- [ ] Confirm Python 3.10+ and dependencies installed
- [ ] Review file permissions in target repository
- [ ] Check disk space in output directory
- [ ] Try with smaller/simpler repository to isolate issue
- [ ] Inspect system temp directory for orphaned clones (web UI)
- [ ] Re-run with fresh output directory

---

## References

- **Error handling implementation:** `Project2v2/multi_agent_system/tools.py`
- **Workflow orchestration:** `Project2v2/multi_agent_system/orchestrator.py`
- **CLI error handling:** `Project2v2/main.py`
- **Web validation:** `Project2v2/security_utils.py`, `Project2v2/web_interface.py`
- **Main README:** `../README.md`
- **Agent specifications:** `AGENTS.md`
