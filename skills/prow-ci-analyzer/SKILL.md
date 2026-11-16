---
name: prow-ci-analyzer
author: OpenShift Pipelines Testing Team
description: >
  Analyze Prow periodic job failures. Use this skill when analyzing failures from Prow CI notifications
  with job names like "periodic-ci-openshift-pipelines-release-tests-*". Detects infrastructure issues
  (provisioning, image pull, network, operator installation) vs test failures. Identifies affected components,
  root cause, and recommends next steps (retry, investigation, fix, workaround).
argument-hint: "[prow-log-url]"
---

# Prow CI Job Analyzer

Analyze Prow periodic job failures from CI notifications. Triages infrastructure issues vs test failures,
identifies root cause, affected test components, and recommends actions.

## Inputs

1. **Prow Notification** from Slack with:
   - Job name: `periodic-ci-openshift-pipelines-release-tests-ginkgo-{BRANCH}-openshift-pipelines-{OCP_VERSION}-...`
   - Prow logs URL: `https://prow.ci.openshift.org/view/gs/test-platform-results-public/logs/...`

Example:
```
:failed: Job periodic-ci-openshift-pipelines-release-tests-ginkgo-release-v1.24-openshift-pipelines-ocp5.0-lp-rosa-hypershift-aws-rosa-hypershift ended with failure. View logs
Prow logs: https://prow.ci.openshift.org/view/gs/test-platform-results-public/logs/periodic-ci-openshift-pipelines-release-tests-ginkgo-release-v1.24-openshift-pipelines-ocp5.0-lp-rosa-hypershift-aws-rosa-hypershift/2122163183245332480
```

## Steps

### 1. Parse Job Name

Extract from notification:
- **Branch**: e.g., `release-v1.24`
- **Repository**: `openshift-pipelines/release-tests-ginkgo`
- **OCP Version**: e.g., `ocp5.0`
- **Platform**: e.g., `rosa-hypershift-aws-rosa-hypershift`
- **Architecture**: Extract from job metadata or logs if available; report as "unknown" if not found in job name or logs

### 2. Fetch and Analyze Prow Logs

Access the provided Prow logs URL. Look for:
- Build output and test execution logs
- Pod logs and events from cluster
- Operator installation/status logs
- Test suite output

### 3. Check for Infrastructure Failures First

Scan logs for infrastructure failure signals:

| Failure Type | Signals |
|--------------|---------|
| **Cluster provisioning** | "cluster provision failed", "resource quota exceeded", "node not ready", "cluster unreachable" |
| **Image pull errors** | "ErrImagePull", "ImagePullBackOff", "404 Not Found" for images |
| **Network issues** | "connection refused", "connection timeout", "DNS failed" |
| **Operator installation** | Operator pod crashes, CRD creation failures, operator logs showing errors |
| **RBAC/SCC issues** | "permission denied", "forbidden", "pods.core... is forbidden" |

**If infrastructure failure found**: Report specific failure, recommend retry or escalation.

### 4. If Infrastructure OK, Analyze Test Failures

Identify which test suites ran and their results:

| Suite | Expected | Signals |
|-------|----------|---------|
| **ecosystem** | buildah, s2i, git-clone tasks | Task pod failures, timeout, assertion errors |
| **triggers** | EventListener, TriggerBinding | Webhook validation, webhook pod issues |
| **operator** | Auto-install, auto-prune, addon | Operator reconciliation, CRD issues |
| **pipelines** | PipelineRun, TaskRun, workspace | Task execution, result handling |
| **pac** | Pipelines-as-Code | GitLab webhook, repository configuration |
| **chains** | Tekton Chains signing | Controller issues, signing failures |
| **results** | Tekton Results | API issues, database connectivity |

For each failed suite:
- **What failed?**: Specific test or task
- **How did it fail?**: Assertion error, timeout, pod crash, panic, etc.
- **Where?**: Which component or task implementation

### 5. Identify Known Issues

Check if failures match known patterns:
- **Flaky tests**: Known intermittent failures in specific suites
- **Platform-specific**: ROSA, hypershift, or OCP version specific issues
- **Recent regressions**: Check if recently introduced or part of ongoing saga
- **Environment issues**: Test cluster specifics, resource constraints

### 6. Report

Present structured analysis:

**Summary**:
- Job: `[name]` Build: `[number]`
- Branch: `[branch]`, OCP: `[version]`, Platform: `[platform]`
- Status: Infrastructure failure / Test failure / Mixed

**Root Cause**:
- **Category**: Infrastructure / Test code / Known issue / Flaky test
- **Component**: Specific operator, task, or suite
- **Evidence**: Quote relevant log lines with file/line context

**Affected Tests**:
- Failed test suites and specific test cases
- Failure mode for each

**Recommendations**:
- **Infrastructure**: Retry job, or escalate to platform team
- **Test failure**: Link to code, suggest fix area or investigation
- **Known issue**: Reference issue number, timeline for fix, workaround if available
- **Flaky**: Retry to verify, or add retry logic to test

## Error Handling

| Failure | Action |
|---------|--------|
| Prow logs unreachable | State logs unavailable; use job name/status only for analysis |
| Partial logs | Proceed with available data; note what could not be verified |
| Unknown test suite | Report as new suite or configuration drift |
| Multiple failures | Identify if cascading (first failure blocks others) or independent |

## Limitations

- Cannot modify or rerun Prow jobs (read-only analysis)
- Test cluster is ephemeral; cannot collect live artifacts
- Prow logs may be garbage-collected after 30 days
- Some platform-specific issues require domain knowledge of ROSA/hypershift
