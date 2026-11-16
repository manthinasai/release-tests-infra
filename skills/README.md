# OpenShift Pipelines Testing Analysis Skills

Specialized skills for analyzing test failures from different testing pipelines.

## Skills Overview

| Skill | Use Case | Input | Log Type |
|-------|----------|-------|----------|
| **prow-ci-analyzer** | Prow periodic job failures | Job name + Prow logs URL | Prow build logs |
| **release-test-analyzer** | Release tests (acceptance/upgrade) + Operator builds | Run name + GCS or Console URL | GCS index.html or Console logs |

## Usage Patterns

### When Chai-bot Receives a Prow Failure Notification

```
:failed: Job periodic-ci-openshift-pipelines-release-tests-ginkgo-release-v1.24-openshift-pipelines-ocp5.0-lp-rosa-hypershift-aws-rosa-hypershift ended with failure. View logs
Prow logs: https://prow.ci.openshift.org/view/gs/test-platform-results-public/logs/...
```

→ **Invoke**: `prow-ci-analyzer`
- Parses job name to extract branch, OCP version, platform
- Analyzes Prow build logs for infrastructure vs test failures
- Reports root cause and recommended action (retry, investigation, fix)

### When Chai-bot Receives a Release Test Notification (Acceptance/Upgrade/Operator-Build)

**Acceptance-test (Ginkgo with GCS)**:
```
:volcano: acceptance-tests acceptance-tests-cp-ginkgo-1240-prod-on-422-5w9bj failed :volcano: logs
https://storage.googleapis.com/ospqa-ci-artifacts/...
```

**Acceptance-test (Gauge with Console)**:
```
:volcano: pipeline run acceptance-tests-cp-1233-stage-on-422-gdhvb failed :volcano: logs
https://console-openshift-console.apps.ocp111119.aws.ospqa.com/.../logs
```

**Operator-build (Ginkgo)**:
```
:volcano: operator acceptance tests openshift-pipelines-index-4-22-1-24-acceptance-tests-amd64-4gwbv failed :volcano: logs
https://storage.googleapis.com/ospqa-ci-artifacts/...
```

→ **Invoke**: `release-test-analyzer`
- Detects test type (acceptance/upgrade/operator-build)
- Detects framework (Gauge vs Ginkgo) from run name
- If GCS URL: Downloads and analyzes test results index
- If Console URL: Requests manual logs from user
- For each failed suite, analyzes logs and performs root cause analysis
- Cross-references SRVKP `release-testing-bug` for known issues
- Reports root cause, framework-specific issues, regressions, recommendations
- User decides if new issue warrants SRVKP bug filing

## Skill Details

### prow-ci-analyzer

**Location**: `skills/prow-ci-analyzer/SKILL.md`

**Workflow**:
1. Parse job name for branch, OCP, platform
2. Fetch Prow logs
3. Check for infrastructure failures (provisioning, image pull, network, operator)
4. If OK, analyze test suite failures
5. Categorize and report root cause

**Output**: Failure category, affected component, recommended action

### release-test-analyzer

**Location**: `skills/release-test-analyzer/SKILL.md`

**Workflow**:
1. Detect test type from notification (acceptance/upgrade/operator-build)
2. Detect framework from run name (Gauge vs Ginkgo)
3. Parse operator/OSP version, environment, OCP version, architecture
4. Determine log type (GCS vs Console)
5. If GCS: Analyze index.html and per-suite logs
6. If Console: Request manual logs from user
7. Analyze by suite with framework context
8. Cross-reference SRVKP `release-testing-bug` for known issues
9. Categorize by failure type and identify regressions
10. Report root cause and user-driven recommendations

**Output**: 
- Test suite results with pass/fail per suite
- Framework-specific failure analysis
- Known issue cross-reference (SRVKP tickets)
- Root cause with evidence and recommendations
- User can file SRVKP bug if new issue identified

## Integration with Slack Workflow

The slack workflow calls chai-bot with a generic message. Instead, it should pass the notification to one of these skills:

### Current Workflow (Generic)
```
@chai-bot Analyze the failed pipelinerun logs...
[Generic instructions]
```

### Improved Workflow (Skill-Based)
```
Detect notification type from message:
- If "periodic-ci-openshift-pipelines-release-tests" → /prow-ci-analyzer
- If "acceptance-tests" or "upgrade-tests" or "operator" → /release-test-analyzer
- Else → /release-test-analyzer (default)

Pass detected skill with: notification text, logs URL
```

## File Structure

```
skills/
├── README.md                          (this file)
├── prow-ci-analyzer/
│   └── SKILL.md                       (Prow periodic job failure analysis)
└── release-test-analyzer/
    └── SKILL.md                       (Release tests: acceptance/upgrade/operator-build)
```

**Note**: Combined `acceptance-test-analyzer` and `operator-build-analyzer` into unified `release-test-analyzer`
for maintainability and shared analysis logic.

## Adding New Skills

When adding a new skill for a different testing pipeline:

1. Create directory: `skills/{skill-name}/`
2. Create `SKILL.md` with frontmatter (name, author, description, argument-hint)
3. Add workflow steps and analysis instructions
4. Update this README with new skill in the overview table
5. Test with real notifications before deploying
