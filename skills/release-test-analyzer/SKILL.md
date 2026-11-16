---
name: release-test-analyzer
author: OpenShift Pipelines Testing Team
description: >
  Analyze release test failures from both pre-stage/stage acceptance tests and Konflux operator builds.
  Supports both Gauge (OSP <=1.23.x) and Ginkgo (OSP 1.24+) frameworks. Detects test type from notification,
  parses operator/OSP versions and OCP versions from run names, analyzes logs (GCS or Console), and cross-references
  known SRVKP release-testing-bug issues. Provides root cause analysis and actionable recommendations.
argument-hint: "[notification-text] [logs-url]"
---

# Release Test Analyzer

Unified analyzer for OpenShift Pipelines release test failures. Handles both pre-stage/stage acceptance tests
and Konflux operator build tests. Supports Gauge (OSP <=1.23.x) and Ginkgo (OSP 1.24+) frameworks.
Analyzes GCS logs or Console-only runs, cross-references known issues, and provides root cause analysis.

## Supported Test Types

| Test Type | Notification Format | Repository | Framework | OSP Versions |
|-----------|-------------------|------------|-----------|--------------|
| **Acceptance-tests** | `acceptance-tests acceptance-tests-cp[ginkgo]-...` | release-tests / release-tests-ginkgo | Gauge / Ginkgo | <=1.23.x / 1.24+ |
| **Upgrade-tests** | `upgrade-tests upgrade-tests-cp[ginkgo]-...` | release-tests / release-tests-ginkgo | Gauge / Ginkgo | <=1.23.x / 1.24+ |
| **Operator builds** | `operator acceptance-tests openshift-pipelines-index-...` | release-tests-ginkgo | Gauge / Ginkgo | <=1.23.x / 1.24+ |
| **Operator upgrades** | `operator upgrade-tests openshift-pipelines-index-...` | release-tests-ginkgo | Gauge / Ginkgo | <=1.23.x / 1.24+ |

## Inputs

1. **Slack Notification** with format:
   ```
   :{icon}: {test-type} {run-name} {status} :{icon}: logs
   {logs-url}
   ```

2. **Logs URL**: GCS (`storage.googleapis.com`) or Console (`console-openshift-console.apps...`)

### Example Notifications

**Acceptance-test (Ginkgo with GCS logs)**:
```
:volcano: acceptance-tests acceptance-tests-cp-ginkgo-1240-prod-on-422-5w9bj failed :volcano: logs
https://storage.googleapis.com/ospqa-ci-artifacts/CI/acceptance-tests-cp-ginkgo-1240-prod-on-422-5w9bj/index.html
```

**Acceptance-test (Gauge with Console logs)**:
```
:volcano: pipeline run acceptance-tests-cp-1233-stage-on-422-gdhvb failed :volcano: logs
https://console-openshift-console.apps.ocp111119.aws.ospqa.com/k8s/ns/pipelines-ci/tekton.dev~v1~PipelineRun/acceptance-tests-cp-1233-stage-on-422-gdhvb/logs
```

**Operator-build (Ginkgo with GCS logs)**:
```
:volcano: operator acceptance tests openshift-pipelines-index-4-22-1-24-acceptance-tests-amd64-4gwbv failed :volcano: logs
https://storage.googleapis.com/ospqa-ci-artifacts/CI/openshift-pipelines-index-4-22-1-24-acceptance-tests-amd64-4gwbv/index.html
```

**Operator-build (Gauge patch with GCS logs)**:
```
:volcano: operator acceptance tests openshift-pipelines-index-4-22-1-23-3-acceptance-tests-amd64-8jk3p failed :volcano: logs
https://storage.googleapis.com/ospqa-ci-artifacts/CI/openshift-pipelines-index-4-22-1-23-3-acceptance-tests-amd64-8jk3p/index.html
```

## Steps

### 1. Detect Test Type

Identify which type of test from notification:

| Signal | Test Type | Repository |
|--------|-----------|------------|
| Has "operator" keyword | Operator-build test | Konflux operator (release-tests-ginkgo) |
| Has "acceptance-tests" or "upgrade-tests" | Release acceptance/upgrade test | release-tests / release-tests-ginkgo |
| Has "ginkgo" in name | Ginkgo framework | release-tests-ginkgo (OSP 1.24+) |
| No "ginkgo" | Gauge framework | release-tests (OSP <=1.23.x) |

### 2. Detect Framework and Parse Run Name

**For Acceptance/Upgrade tests:**

Both Ginkgo and Gauge use the same naming structure with an optional installer prefix:

**Format**: `{test-type}-[{INSTALLER}-][ginkgo-]{VERSION}-{ENVIRONMENT}-on-{OCP}-{SUFFIX}`

**Installer prefixes** (optional, first segment after test-type):
- `cp` — Cluster Provisioning (standard)
- `aro` — Azure Red Hat OpenShift
- `rosa` — Red Hat OpenShift on AWS
- `aws-ipi` — AWS IPI deployment
- (none) — No installer prefix

**Ginkgo examples** (includes "ginkgo" segment):
- `acceptance-tests-cp-ginkgo-1240-prod-on-422-5w9bj` (Cluster Provisioning)
- `acceptance-tests-ginkgo-1240-prod-on-422-5w9bj` (No installer prefix)
- `acceptance-tests-rosa-ginkgo-1240-stage-on-422-abcde` (ROSA)
- `upgrade-tests-cp-ginkgo-1232-to-1240-on-422-xyz123` (Upgrade)

**Gauge examples** (no "ginkgo" segment):
- `acceptance-tests-cp-1233-stage-on-422-gdhvb` (Cluster Provisioning)
- `acceptance-tests-1233-stage-on-422-gdhvb` (No installer prefix)
- `acceptance-tests-aro-1233-prod-on-422-klmno` (ARO)
- `acceptance-tests-aws-ipi-1233-prod-on-422-pqrst` (AWS IPI)

Extract:
- **Operator version**: cp-XYZZ (e.g., 1240 = 1.24.0, 1233 = 1.23.3)
- **Environment**: prod, stage, pre-stage
- **OCP version**: on-XXX (e.g., 422 = OCP 4.22)
- **Suffix**: Run ID
- **Upgrade info**: If present, e.g., "1232-to-1240" means 1.23.2 → 1.24.0

**For Operator-build tests:**

**Run name format**: `openshift-pipelines-index-{OCP_VERSION}-{OSP_VERSION}-{TEST_TYPE}-{ARCH}-{SUFFIX}`

Ginkgo example: `openshift-pipelines-index-4-22-1-24-acceptance-tests-amd64-4gwbv`
Gauge example: `openshift-pipelines-index-4-22-1-23-3-acceptance-tests-amd64-8jk3p`

Extract:
- **OCP version**: 4-22 (OCP 4.22)
- **OSP version**: 1-24 (OSP 1.24.0) or 1-23-3 (OSP 1.23.3)
- **Framework**: Ginkgo (1.24+) or Gauge (<=1.23.x)
- **Test type**: acceptance-tests or upgrade-tests
- **Architecture**: amd64, arm64, ppc64le
- **Upgrade info**: If present, e.g., "1-24-to-1-25" means 1.24 → 1.25

### 3. Determine Log Type

**Type A: GCS URL** (analyzable)
- Pattern: `https://storage.googleapis.com/ospqa-ci-artifacts/...`
- Contains `index.html` with test results
- ✅ **Proceed to Step 4 (Analyze)**

**Type B: Console URL** (manual required, acceptance-tests only)
- Pattern: `https://console-openshift-console.apps.*/k8s/.../PipelineRun/.../logs`
- PipelineRun logs not exported
- ⚠️ **Jump to Step 6 (Request Manual Logs)**

### 4. Analyze GCS Logs

**For Acceptance/Operator-build tests with GCS URL:**

Access `index.html`. Parse:
- **Summary section**: Pass/fail counts, failed task list
- **Test results section**: PASS/FAIL badges per suite
- **Log files**: Download each `.log` file

For each failed log:
- Extract suite name (ecosystem, triggers, operator, pipelines, pac, chains, results, versions)
- Scan for failure signals:
  - `FAILED`, `Fail` → Test assertion
  - `timeout`, `deadline exceeded` → Timeout
  - `CrashLoopBackOff`, `OOMKilled` → Pod issues
  - `forbidden`, `permission denied` → RBAC/SCC
  - `panic` → Code panic
  - `connection refused` → Network
  - `not found` → Missing resource

### 5. Analyze by Suite and Framework

Consider framework differences and versions:
- **Framework**: Gauge (BDD-driven with feature files) vs Ginkgo (function-driven)
- **OSP version**: New features, breaking changes, deprecated behavior
- **OCP version**: Platform differences, API changes
- **Architecture**: arm64 vs amd64 specific issues
- **Repository**: release-tests vs release-tests-ginkgo differences

| Suite | Framework (Ginkgo / Gauge) | Common Issues |
|-------|----------------------------|---------------|
| **ecosystem** | Ginkgo / Gauge | buildah, s2i, git-clone task failures on platforms |
| **triggers** | Ginkgo / Gauge | webhook validation, EventListener, TriggerTemplate |
| **operator** | Ginkgo / Gauge | Operator install, addon CRs, TektonConfig reconciliation |
| **pipelines** | Ginkgo / Gauge | PipelineRun execution, workspace handling, results |
| **pac** | Ginkgo / Gauge | GitLab integration, webhook routing, RBAC |
| **chains** | Ginkgo / Gauge | Signing, key management, certificates |
| **results** | Ginkgo / Gauge | gRPC APIs, database, event recording |
| **versions** | Ginkgo / Gauge | Version mismatch, CLI version checks |

Categorize failure:
- **Test code issue**: Assertion failure, expectations not met
- **Operator issue**: Operator pod crash, CRD failure, reconciliation timeout
- **Task execution issue**: Pod crash, OOMKilled, RBAC, resource limits
- **Infrastructure issue**: Cluster not ready, network, storage
- **Environment setup**: Missing secret, misconfigured resource
- **Regression**: Worked before, broke in new version
- **Platform-specific**: arm64, ppc64le, ROSA, hypershift issues

### 6. Request Manual Logs (Console-Only Case)

If Console-only (no GCS export), respond with:

```
⚠️ PipelineRun logs not available for analysis

The test run only has Console logs available.
To analyze this failure, please:

1. Download the PipelineRun logs:
   - Visit: {console-url}
   - Click "Download" or "Copy logs"
   - Paste raw logs in Slack thread

2. Or provide:
   - Pod descriptions (oc describe pod...)
   - Pod logs (oc logs...)
   - PipelineRun/TaskRun status (oc describe taskrun...)

Once logs are provided, I can analyze and identify root cause.
```

### 7. Check Known Issues (SRVKP Release-Testing Bugs)

Cross-reference against known issues:

1. **Search SRVKP for release-testing-bug label**:
   - Query: `labels = release-testing-bug AND (OSP version match OR OCP version match)`
   - Check if failure matches any known bug from this or earlier release
   - Reference: https://redhat.atlassian.net/issues/?jql=labels=%20release-testing-bug

2. **Check issue scope using version fields**:
   - For each matching ticket, check:
     - **Affects version**: Which OSP/OCP versions have this bug
     - **Fix version** or **Target version**: When the bug is/was fixed
   - Example: https://redhat.atlassian.net/browse/SRVKP-13933 shows issue scope across versions
   - If current version is in "Affects version", it's a known bug
   - If bug is already "Fixed" in current version, may be environmental

3. **Check if issue is version-specific**:
   - If Gauge <=1.23.x: Check known issues in all <=1.23.x releases (old releases, patch releases)
   - If Ginkgo 1.24+: Check known issues in 1.24+ releases
   - Check OSP release notes for breaking changes
   - Check OCP version-specific limitations
   - Check platform-specific known issues (ROSA, hypershift, etc.)

4. **Determine if known or new**:
   - **Known**: Reference ticket number, note status (check Affects/Fix versions), suggest workaround or timeline
   - **New/Unclear**: Report findings with evidence; user can file SRVKP bug if needed (many runs occur, avoid duplicate/flaky bug reports)

### 8. Report

Present structured analysis:

**Summary**:
- Test type: `acceptance-tests`, `upgrade-tests`, or `operator acceptance-tests`
- Test context: Pre-stage/stage release test OR Konflux operator build
- Framework: `Gauge` (OSP <=1.23.x) or `Ginkgo` (OSP 1.24+)
- Repository: `release-tests` or `release-tests-ginkgo`
- Operator/OSP version: `{version}` on {environment}
- OCP version: `{version}`, Architecture: `{arch}`
- Status: `{X} passed, {Y} failed` (or "All passed")

**Test Results**:
- Table: Suite | Status | Failure Type

**Failed Suites Analysis**:
For each failed suite:
- **Suite**: Name
- **Framework context**: Gauge-specific vs Ginkgo-specific failure mode
- **Root cause**: Category (code issue, operator issue, infrastructure, etc.)
- **Evidence**: Quote key log lines with context
- **Affected task**: Specific failing task

**Known Issue Cross-Reference**:
- Check if failure matches any SRVKP `release-testing-bug`
- If match: Reference ticket (e.g., SRVKP-15498), note status, suggest workaround or timeline
- If new/unclear: Report findings with evidence; user can file SRVKP bug if needed

**Regression Analysis**:
- Is this new in this OSP/OCP version?
- Did it work in previous version (<=1.23.x vs 1.24+)?
- Specific to this architecture/platform?
- Framework migration issue (Gauge→Ginkgo)?

**Recommendations**:
- **Code fix**: Point to source code area, suggest changes (note which framework/repo)
- **Configuration**: Check operator CR, environment variables, secrets
- **Re-run**: If likely transient (network, timing), recommend retry
- **Escalation**: If operator or platform issue, suggest team
- **Known issue**: Link to SRVKP ticket, note workaround or timeline
- **Framework issue**: If Gauge→Ginkgo transition problem, flag for test porting

**Next Steps**:
1. Check related SRVKP issues with `release-testing-bug` label
2. Review OSP release notes for version changes
3. If regression: Compare with previous passing version
4. Investigate operator logs: `oc describe csv tekton-operator`
5. Check cluster state: `oc get nodes`, `oc top nodes`
6. If new issue: User can file SRVKP bug with label `release-testing-bug` for future reference

## Error Handling

| Failure | Action |
|---------|--------|
| GCS URL unreachable | Report and ask for alternative access |
| index.html malformed | Describe parsing error, suggest checking URL |
| Log file download fails | Suggest direct GCS access or manual download |
| Too many failures | Focus on first failure (often cascading) |
| Timeout in log | Note exact timeout duration, identify what was waiting |
| Mixed OSP/OCP versions | Flag inconsistency, may indicate partial upgrade |
| No log file available | Request manual logs from user |
| Malformed GCS URL | Report and ask user to verify URL |

## Limitations

- Cannot modify or retry builds/tests (read-only analysis)
- Cannot access artifacts beyond public GCS URL
- Some operator-specific issues require source code access
- Architecture-specific bugs may require testing on that platform
- Flaky test analysis limited to single run (historical data not available)
- Framework migration (Gauge→Ginkgo) issues may require special investigation
