Implementation blueprint: Copilot SDK + LangGraph + isolated execution on AKS

For your existing test-code-generation system, I would build this as a job-oriented service with four clear responsibilities:

Repository service: retrieves a pinned version of the test automation repository and supplies relevant code context.
Generation agent: uses Copilot SDK to create or modify test scripts in a restricted workspace.
Execution and repair service: runs Selenium/Playwright Java tests in a separate, disposable worker and returns diagnostics.
Publishing service: creates a GitHub pull request only after validation and human approval.

Your existing Python/FastAPI, LangGraph, GitHub integration and LangSmith choices fit this architecture well.

1. Target architecture

FastAPI — Job API

Test-case JSON · repository · framework · job ID

Repository preparation service

GitHub checkout at commit SHA · framework detection · index retrieval

Generation pod / container

Copilot SDK · pre-tool policy · post-tool audit · job-scoped workspace

(Read code) (Search symbols) (Write generated files) (Static validation)

Execution worker pod

Pinned JDK + Maven/Gradle + browser · run approved test command · collect reports

Repair → validate → human approval

Bounded repair attempts · diff review · GitHub PR publisher

The generation and execution environments should be separate security boundaries. A generated script is untrusted code, even when the model is trusted.

2. Decide exactly what the generation agent may do

Don't start with “allow all except the operating environment.” Build a positive allowlist and grant each capability only when a real workflow step needs it.

Proposed tool policy
Tool or capability	Generation agent	Execution worker
Read repository files	Allow	Read-only source/test mount
Search files and symbols	Allow	Usually unnecessary
List directories	Allow	Allow
Edit files in job workspace	Allow	No source edits
Write generated artifact	Allow, restricted path	Write reports only
Run Python syntax checks	Prefer your application	Not needed
Run arbitrary shell commands	Deny by default	Deny arbitrary commands; invoke fixed runner
Run Maven/Gradle	Prefer execution service	Allow fixed, validated test command
Install arbitrary packages	Deny	Deny at runtime; use prebuilt images
Read environment variables/secrets	Deny through agent tools	Only explicitly provisioned test secrets
Git fetch/clone	Repository service only	No Git credentials
Git push / create PR	Deny	Deny
Modify GitHub branches	Deny	Deny
Access arbitrary network destinations	Deny by default	Allow only required destinations
Access Kubernetes API	Deny	Deny
Create containers or pods	Deny	Deny
Use MCP tools	Only explicitly approved tools	Normally none

Important: available_tools and on_pre_tool_use are Copilot SDK tool controls, not a substitute for container security. The current SDK documents the pre-tool hook as a way to allow, deny or modify a tool invocation.

3. Use a controlled toolset instead of giving Copilot the full CLI toolset

This is the design decision I would make first.

The Copilot SDK supports different runtime modes, including an empty mode in which you can provide your own tools rather than relying on the normal built-in CLI tools. It also supports custom tools, pre-tool hooks and post-tool hooks.

For your first production version, I'd use mode="empty" and expose a small set of application-owned tools.

The exact toolset I would implement
Generation agent — allowed tools

read_workspace_file

Read a file within the pinned repository checkout. Reject paths outside the workspace, symbolic-link escapes, and sensitive files such as .env and private-key files.

list_workspace_files

List files under an approved directory, excluding .git, build output, dependency caches, and secrets.

search_workspace

Search source files and configuration for existing page objects, fixtures, base classes, test annotations, and framework conventions. Implement with a bounded search service rather than arbitrary shell execution.

write_generated_file

Write a new file into a job-specific output directory. Accept only approved relative paths and file extensions. Initially, do not let the agent overwrite the checked-out repository.

get_framework_manifest

Return a structured description of the repository's supported framework, build tool, Java version, test runner, approved test command, and relevant reusable classes. This is an application-owned deterministic tool.

Tools I would deliberately not expose

Do not give the generation agent direct access to:

Arbitrary bash, shell, PowerShell or command execution.
git push, gh pr create, or branch-management operations.
Kubernetes API operations, Docker socket access, or pod creation.
Package installation and arbitrary network requests.
Environment-variable dumps, credential stores, or general filesystem access.
A general-purpose tool that executes an arbitrary command passed by the model.

This avoids depending on an ever-changing list of built-in Copilot CLI tools. If you instead use the default CLI runtime, tool names and available capabilities depend on the pinned CLI/runtime version and configuration. Maintain a version-specific allowlist and test it whenever you upgrade.

4. Implement pre-tool and post-tool policies

Use the hooks for enforcement at the tool-call boundary and auditing around it. The pre-tool hook can allow, deny or modify a tool call; the post-tool hook runs after successful execution. There is a separate failure hook for unsuccessful calls.

The following is the structure I'd use. It assumes you have registered the five application-owned tools listed above and that your audit service is asynchronous.

# src/codegen/agents/generation/tool_policy.py

import json
import logging
from pathlib import Path

logger = logging.getLogger(__name__)

ALLOWED_GENERATION_TOOLS = {
    "read_workspace_file",
    "list_workspace_files",
    "search_workspace",
    "write_generated_file",
    "get_framework_manifest",
}

FORBIDDEN_PATH_PARTS = {
    ".git",
    ".env",
    ".ssh",
    ".azure",
    ".aws",
    "__pycache__",
    "node_modules",
}

def safe_workspace_path(
    workspace: Path,
    relative_path: str,
    *,
    output: bool = False,
) -> Path:
    """Resolve a relative path and enforce workspace boundaries."""

    root = workspace.resolve(strict=True)
    candidate = (root / relative_path).resolve(strict=False)

    if not candidate.is_relative_to(root):
        raise PermissionError("Path escapes the job workspace")

    parts = Path(relative_path).parts

    if any(part in FORBIDDEN_PATH_PARTS for part in parts):
        raise PermissionError("Access to this path is prohibited")

    if output and not relative_path.startswith("generated/"):
        raise PermissionError(
            "Generated files must be under generated/"
        )

    return candidate

async def on_pre_tool_use(input_data, invocation):
    tool_name = input_data["toolName"]
    args = input_data.get("toolArgs") or {}

    if tool_name not in ALLOWED_GENERATION_TOOLS:
        logger.warning(
            "Blocked tool=%s session=%s",
            tool_name,
            invocation.get("session_id"),
        )
        return {
            "permissionDecision": "deny",
            "permissionDecisionReason": (
                "This tool is not permitted in generation sessions."
            ),
        }

    # Tool handlers must validate all paths and arguments again.
    # A hook is not a substitute for handler-level validation.
    return {"permissionDecision": "allow"}

async def on_post_tool_use(input_data, invocation):
    # Avoid logging source code, full tool results, or secret values.
    tool_name = input_data["toolName"]
    args = input_data.get("toolArgs") or {}

    await audit_tool_event(
        session_id=invocation.get("session_id"),
        tool_name=tool_name,
        outcome="success",
        # Log only allowlisted metadata, not raw arguments/results.
        metadata={
            "path": args.get("path"),
        },
    )

    return None

async def on_post_tool_use_failure(input_data, invocation):
    await audit_tool_event(
        session_id=invocation.get("session_id"),
        tool_name=input_data["toolName"],
        outcome="failure",
        # Store a redacted, size-limited error separately if needed.
        metadata={},
    )

    return {
        "additionalContext": (
            "The tool failed. Do not retry by using an alternative "
            "unapproved tool or attempting direct shell access."
        )
    }

audit_tool_event is your application function; implement it to write structured audit events to your database or logging pipeline.

Two details matter:

The safe_workspace_path function must also be called inside the actual read/write tool handlers. A policy hook alone is not enough.
Path.resolve() is a useful starting point, but concurrent filesystem changes and symlink races require additional protection. For higher-assurance file handling, use a dedicated workspace service with safe file operations and a read-only source mount.
Create the session with only your own tools

The SDK's Python API documents CopilotClient, create_session, tools, and hook registration. It also supports mode="empty" for a session where you supply the intended tools.

# Illustrative integration pattern.
# Pin the SDK/runtime version and verify against that version.

from copilot import CopilotClient, PermissionDecisionApproveOnce

from codegen.agents.generation.tool_policy import (
    on_pre_tool_use,
    on_post_tool_use,
    on_post_tool_use_failure,
)

async def generate_test(workspace, tools, prompt):
    async with CopilotClient(
        mode="empty",
        working_directory=str(workspace),
        base_directory=str(workspace / ".copilot-state"),
    ) as client:
        async with await client.create_session(
            tools=tools,
            available_tools=[],
            on_permission_request=lambda req, inv: (
                PermissionDecisionApproveOnce()
            ),
            hooks={
                "on_pre_tool_use": on_pre_tool_use,
                "on_post_tool_use": on_post_tool_use,
                "on_post_tool_use_failure": (
                    on_post_tool_use_failure
                ),
            },
        ) as session:
            return await session.send_and_wait(prompt)

Here, tools should be the five application-owned tool definitions. This code illustrates the intended configuration; validate the exact constructor and session signatures against your pinned SDK version before deploying.

Do not switch to PermissionHandler.approve_all and assume that makes the setup safe. The permission handler and the tool allowlist serve different purposes. Keep the pre-tool policy, restrict which tools are registered, and rely on the container boundary as an independent security layer.

Also note that hooks are an audit and policy mechanism, not a substitute for sandboxing. The process running the SDK must still be restricted by the OS and Kubernetes.

5. How the tool handlers should work

Each custom tool should have a narrow contract. For example:

Tool	Input	Enforced behaviour
read_workspace_file	Relative path	Read only approved source/configuration files; enforce file-size limit
list_workspace_files	Relative directory	Return a bounded list; never expose secret directories
search_workspace	Query, directory, file patterns	Limit query length, file count, output size and execution time
write_generated_file	Relative path, content	Allow only approved extensions and generated/ paths; reject symlinks and oversized files
get_framework_manifest	None or approved framework ID	Return a trusted manifest built by your application

For search_workspace, use a controlled search library or a fixed rg invocation with argument arrays and no shell interpolation. Never allow the model to supply arbitrary command-line flags that can change the security properties of the operation.

What happens after generation?

Don't allow the model to declare that its output is validated. Your application should independently:

Validate the output schema and expected file paths.
Scan generated content for secrets and prohibited operations.
Run deterministic syntax and policy checks.
Build an explicit patch against the pinned repository checkout.
Submit the patch to the execution service.
Store the tool audit trail, output hashes, validation results and execution logs against the job ID.

The post-tool hook should record what happened, not decide whether the test passed. Test results should come from the execution worker's exit code and machine-readable reports.

6. What commands can the execution worker run?

I would not expose a generic run_command(command: str) tool to Copilot. Instead, expose a single constrained application operation such as submit_for_execution(job_id, artifact_id) to the orchestrator—not as an arbitrary shell tool to the generation agent.

Your execution service then selects an approved command based on a trusted repository manifest.

Repository type	Example approved command	Notes
Maven + JUnit/TestNG	mvn -B -ntp test	Use the repository's configured test plugins and profiles
Gradle + JUnit/TestNG	./gradlew test	Invoke the checked-in wrapper, with controlled options
Maven with a named test profile	A predefined Maven profile	Select the profile from an allowlist
Custom enterprise test runner	A predefined runner entrypoint	Use only a reviewed repository configuration

These are examples, not commands to run blindly. Your manifest should identify the actual build tool, task, test runner, Java version, browser requirements and approved test-selection mechanism. Avoid constructing commands by concatenating model-generated text.

The worker should enforce these limits
Maximum execution time and CPU/memory limits.
Maximum log output and artifact size.
No privileged container, no host mounts and no Docker socket.
No production credentials or GitHub publishing token.
Network access only to the test application and required services.
No package installation during execution unless explicitly needed and controlled.
Bounded repair attempts, with the repair agent unable to weaken assertions merely to get a green result.

A build tool executes project-defined plugins and code, so even a fixed mvn test command must be treated as executing untrusted code. Container isolation and network restrictions are essential.

7. Should you keep a temporary copy of the repository?

Yes. For your use case, use an ephemeral, job-specific checkout of the repository. Don't give the agent direct access to the live GitHub repository, and don't share a writable checkout between concurrent jobs.

The lifecycle should be:

1. Resolve the repository commit SHA

Pin the exact source version for reproducibility.

2. Checkout into job-specific temporary storage

Fetch using a short-lived GitHub App token; keep credentials outside the agent environment.

3. Give the generation agent constrained access

Read the pinned checkout; write generated files into a separate output directory.

4. Create a patch and pass it to the worker

The execution worker gets the pinned source and generated patch, not GitHub credentials.

5. Store approved artifacts, then destroy the workspace

Persist logs and reports under retention policy; remove temporary source and build files.

How to implement the temporary workspace

For an initial implementation, create a unique job directory or use a Kubernetes emptyDir volume for each job. Do not use one shared writable directory for multiple jobs.

The repository service should:

Fetch the exact commit and verify the resulting HEAD SHA.
Record repository identity, commit SHA and checkout timestamp.
Avoid putting the token into the remote URL, shell logs or agent environment.
Remove the credential helper or other temporary authentication material after checkout.
Make the source read-only to the generation process if your custom tools can read it separately.
Keep generated files, build outputs and test reports in distinct directories.

Use a disposable worker workspace for test execution as well. It can be constructed from the same pinned source plus the validated patch. If the generation agent and execution worker run in separate pods, transfer the patch through controlled artifact storage rather than mounting a shared writable source volume.

8. Do you still need CodeGraph or Graphify?

Yes, I would retain code indexing—but as a retrieval accelerator, not as a mandatory dependency for every small job.

A local checkout gives the agent access to files. It does not automatically give the agent an efficient, complete understanding of a large repository.

Your repository contains different automation frameworks and likely shared fixtures, page objects, base classes, build profiles and conventions. The challenge is not simply finding a Java file; it's finding the right files and relationships for a given test case.

Approach	Strength	Limitation
Copilot tools + direct file search	Simple; useful for small repositories and focused tasks	May require repeated exploration and consume more time/context
Fast symbol/file index	Quick discovery of classes, methods, imports and configuration	Doesn't necessarily capture deeper semantic relationships
CodeGraph / Graphify	Can supply richer structural relationships, depending on the indexer	Indexing cost, freshness and query quality need measurement
Hybrid retrieval	Uses fast search first and deeper graph retrieval when needed	Requires a little more orchestration
Recommended retrieval strategy

Test case + framework + repository commit SHA

Fast retrieval

Framework manifest, filenames, symbols, annotations, imports, fixtures

Need more context?

Retrieve graph relationships and relevant implementations from CodeGraph or Graphify

Bounded context package

Only relevant files and snippets, with paths and source commit recorded

How to manage index freshness

Key the index by repository identity and commit SHA, or maintain an incremental index with explicit freshness metadata. If the index does not match the job's pinned source version, either refresh it or fall back to direct search.

I would benchmark CodeGraph and Graphify on the same representative jobs before making either one mandatory. Measure indexing time, retrieval latency, relevant-file recall, token usage, and first-pass test execution success.

In short: you can build an MVP without code indexing, but I wouldn't remove it from your intended enterprise architecture. Keep a fast fallback and avoid waiting for a full graph rebuild on every job.

9. Deploy it on AKS

I would not run the API, Copilot runtime, and browser test execution all inside the same long-lived pod.

Use separate workloads with different identities and permissions.

A. API and orchestration Deployment

FastAPI, LangGraph, job status, approval workflow, retrieval coordination. No shell access to a job's repository.

B. Ephemeral generation Job

Copilot SDK and CLI runtime, job-scoped checkout, restricted tool policy, temporary output directory. Terminates after generation and artifact collection.

C. Ephemeral execution Job

Prebuilt Java/Maven or Gradle image with compatible browser dependencies. Runs only the selected test job and returns reports.

D. Shared managed services

Azure Database for PostgreSQL for job state/checkpoints; Blob Storage for artifacts; Azure Key Vault for secrets; LangSmith for traces and usage where appropriate.

Step-by-step implementation order
Build the API and job state first. Create a job ID, persist the repository and commit SHA, test-case JSON, framework, state and retry budget. Run long jobs asynchronously rather than blocking the HTTP request.
Build the repository service. Use your existing GitHub integration to fetch the pinned commit into a unique workspace. Create a framework manifest and retrieval package.
Build the generation container. Pin the Copilot SDK and compatible runtime versions. Register only your custom tools, configure hooks, and validate that shell, Git publishing and unrelated MCP tools are unavailable.
Build deterministic validation. Validate paths, JSON schemas, generated Java files and repository policies before running any code. Store the generated patch as an artifact.
Build the execution image. Preinstall the required JDK, Maven/Gradle, browser binaries and test dependencies. Execute a fixed, approved command in a fresh workspace with resource and time limits.
Add repair through LangGraph. Classify failures into compilation, missing dependency, locator, application/environment, assertion and infrastructure categories. Send only relevant diagnostics back to the repair agent. Bound repair attempts and rerun validation after each change.
Add approval and publishing last. Show the diff, test results and provenance to the reviewer. After approval, let the publishing service create the branch and PR using a separate GitHub App identity.
AKS security baseline

For both generation and execution pods, start with these controls:

Run as a non-root user and drop Linux capabilities.
Use a read-only root filesystem, with explicitly mounted writable emptyDir locations for temporary files, SDK state and output.
Apply CPU, memory, ephemeral-storage and execution-time limits.
Use the RuntimeDefault seccomp profile and enforce your cluster's Pod Security Standards.
Avoid privileged containers, host namespaces, host paths, service-account token mounts and access to the Docker socket.
Use separate Kubernetes service accounts and Azure workload identities for services that genuinely need Azure access.
Apply NetworkPolicies to restrict pod-to-pod traffic; use suitable egress controls for external destinations.

Microsoft's AKS guidance recommends least privilege, restricting credential exposure and controlling pod network traffic.

A basic security-context starting point looks like this:

securityContext:
  runAsNonRoot: true
  runAsUser: 10001
  runAsGroup: 10001
  seccompProfile:
    type: RuntimeDefault

containers:
  - name: generation-worker
    image: your-acr/generation-worker:version-pinned
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop: ["ALL"]
    resources:
      requests:
        cpu: "500m"
        memory: "1Gi"
        ephemeral-storage: "2Gi"
      limits:
        cpu: "2"
        memory: "4Gi"
        ephemeral-storage: "6Gi"

This is a starting fragment, not a complete Deployment or Job manifest. You must add the writable volumes, compatible image user, environment configuration, lifecycle, network policy and job-specific limits. A Java/browser execution worker may need different resource allocations from a generation worker.

One further detail: a standard Kubernetes NetworkPolicy does not provide general hostname-based egress rules. For external API allowlists, consider an appropriate egress gateway or Azure Firewall; use a supported FQDN-filtering solution where available.

10. Observability, cleanup and failure handling

Because the system will run multiple concurrent jobs on AKS, every stage should emit structured telemetry using the same job_id and trace ID.

Capture at least:

Repository and commit SHA.
Generation session ID, model identifier and available token/usage metadata.
Tool name, decision, duration, outcome and redacted arguments.
Static validation results.
Execution command ID, exit code, elapsed time, and test report summary.
Repair count, failure classification and final outcome.
Approval identity, approved diff hash and resulting PR URL.

Use LangSmith for workflow traces and model-stage analysis, while your application database remains the source of truth for job status and your audit system records security decisions. Do not assume every Copilot SDK call exposes complete token usage automatically; record usage when available and explicitly mark unavailable values as unknown.

On success, failure, cancellation or timeout:

Stop the running session and worker process.
Collect approved reports and logs.
Persist the final job state and audit events.
Remove temporary workspaces and credentials.
Delete the ephemeral Kubernetes Jobs according to your retention policy.
11. Final decisions I'd lock in
Design question	Decision
Copilot runtime	SDK in empty mode with custom tools
Pre-tool hook	Deny unknown tools; validate each invocation
Post-tool hook	Redacted audit events; separate failure hook
Repository access	Ephemeral, commit-pinned checkout
Generated output	Separate, job-scoped staging directory
Code indexing	Hybrid retrieval; fast search first, graph retrieval when useful
Execution	Separate ephemeral AKS Job with prebuilt browser/test image
Repair	LangGraph-controlled and bounded
GitHub operations	Separate publishing service after human approval
Secrets	Key Vault/workload identity where applicable; never expose publishing credentials to agents
Cleanup	Automatic on every terminal job state

The most important architectural principle is this: Copilot should decide what code to generate, your application should decide which tools and operations are permitted, and the execution worker should decide whether the code actually works. No single agent should have all three responsibilities and unrestricted access.

This preserves your existing Python, FastAPI, LangGraph and LangSmith design while making the system easier to scale and secure on AKS.

This might also interest you.

Xpherium

AI Agents for Data

Build AI agents for your business data.

Ad
More options
How do i set up the execution runtime? Also is maven enough for running automation test scripts
1. Is Maven enough to run automation scripts?

No. Maven is enough to orchestrate the test build only when the repository is already configured correctly. It is not the complete execution runtime.

Maven's Surefire plugin runs supported Java test frameworks such as JUnit and TestNG, compiles test code as part of the Maven lifecycle, and produces test reports.

For your Selenium and Playwright Java repositories, think of the runtime as a bundle:

Java JDK

Compiles and runs the Java test code. Use the Java version required by the repository.

Maven or Gradle

Resolves dependencies, compiles tests, invokes the configured test runner, and reports results.

Browser and browser dependencies

Chromium, Chrome, Firefox or another supported browser, plus compatible OS libraries and any required driver/browser management.

Test environment

Network access to the application under test, approved test accounts, configuration, test data and any required APIs or databases.

Reporting and artifacts

JUnit XML, logs, screenshots, browser traces and other outputs needed to diagnose failures.

Selenium versus Playwright
Selenium Java: Selenium Manager can manage browser drivers and, in supported configurations, browsers. However, downloads, browser availability and network access must be considered.
Playwright Java: The Playwright library requires compatible browser binaries. Its official documentation describes installing those binaries and their operating-system dependencies.

For a production AKS worker, I would preinstall the browser and OS dependencies in a versioned container image rather than download them every time a job runs.

2. How I would set up the execution runtime

I'd build a small execution platform around prebuilt container images + Kubernetes Jobs, with one disposable Job per execution attempt.

You don't need a separate runtime image for every test case. You need a managed set of compatible runtime images and a trusted way to select the correct one for each repository.

Step 1 — Identify the repository's runtime requirements

When your repository service checks out a commit, have it create a manifest similar to this:

{
  "repository": "automation-tests",
  "commit_sha": "pinned-commit-sha",
  "language": "java",
  "build_tool": "maven",
  "java_version": "17",
  "test_framework": "testng",
  "browser_framework": "selenium",
  "browser": "chrome",
  "execution_profile": "ci",
  "test_command_id": "maven-testng-default"
}

This is an example, not a manifest to hardcode for every repository.

The detection service should inspect:

pom.xml or build.gradle / build.gradle.kts.
.mvn/wrapper or gradlew.
Java version and compiler configuration.
Selenium or Playwright dependencies.
JUnit or TestNG dependencies.
Existing CI workflows and test-runner configuration.
Browser selection, environment configuration and required test secrets.

Security rule: the manifest's test_command_id must resolve to a command your application has approved. Do not execute a command supplied directly by the LLM.

Step 2 — Build versioned runtime images

I would initially maintain two runtime image families:

java-selenium-runner

JDK + Maven or supported wrapper + compatible browser + driver management + reporting utilities.

java-playwright-runner

JDK + Maven or supported wrapper + the browser binaries and OS dependencies compatible with the repository's Playwright version.

Pin the image by version or digest. Update it through your CI pipeline, run compatibility checks, and publish it to Azure Container Registry (ACR).

For Selenium, choose whether your browser runs inside the execution container or on a dedicated Selenium Grid. For Playwright, make sure the image's browser binaries match the Playwright Java dependency used by the repository.

Example: Playwright browser preparation

For a repository whose Playwright Java dependency and Maven configuration are already available, its browser installation step can look like this:

mvn exec:java \
  -e \
  -Dexec.mainClass=com.microsoft.playwright.CLI \
  -Dexec.args="install chromium"

The official Playwright Java documentation describes this installation mechanism. Run it during image preparation, not in every execution job. If the project uses a different Playwright version or browser, prepare and validate the corresponding image accordingly.

For Selenium, similarly preinstall the browser and compatible system dependencies if your chosen image requires them. Selenium Manager can help manage drivers, but don't rely on downloading unpinned components during every production test run.

Step 3 — Package the job's source and generated patch

Your execution worker should receive:

The exact repository checkout at the recorded commit.
The generated patch or approved output files.
The selected runtime image.
A trusted test command identifier.
Only the test configuration and credentials needed for that run.

Apply the patch in a fresh working directory, then validate the resulting diff before launching the test command. Keep the original checkout and generated artifact separately available for debugging and provenance.

Use an artifact store or controlled transfer mechanism to pass the patch and results between pods. Avoid a shared writable repository volume between the generator and executor.

Step 4 — Launch an ephemeral AKS Job

Your FastAPI/LangGraph orchestration service should ask a worker service to create a Kubernetes Job. The worker, not Copilot, chooses the image and command from the trusted manifest.

A simplified Job template:

apiVersion: batch/v1
kind: Job
metadata:
  generateName: test-execution-
  namespace: test-execution
spec:
  backoffLimit: 0
  activeDeadlineSeconds: 900
  ttlSecondsAfterFinished: 3600
  template:
    metadata:
      labels:
        app: test-execution
    spec:
      restartPolicy: Never
      automountServiceAccountToken: false
      securityContext:
        runAsNonRoot: true
        runAsUser: 10001
        runAsGroup: 10001
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: runner
          image: your-acr.azurecr.io/java-selenium-runner:17-v1
          command: ["/opt/runner/execute-job"]
          args: ["--job-id", "JOB_ID"]
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: ["ALL"]
          resources:
            requests:
              cpu: "1"
              memory: "2Gi"
              ephemeral-storage: "2Gi"
            limits:
              cpu: "2"
              memory: "4Gi"
              ephemeral-storage: "6Gi"

This is a template to adapt—not a complete deployable manifest. The worker entrypoint, writable volumes, image permissions, network policy, configuration and result-upload mechanism still need to be implemented. Replace JOB_ID using your trusted Job creation code.

Important design choices:

backoffLimit: 0 avoids Kubernetes automatically rerunning a potentially expensive or non-idempotent test. Your orchestrator decides whether a retry is appropriate.
activeDeadlineSeconds places an upper bound on the job's lifetime.
ttlSecondsAfterFinished allows completed Job objects to be cleaned up later; it does not by itself remove artifacts stored elsewhere.
The worker should not need a Kubernetes API token. Let the orchestration service's separate identity create Jobs.
Apply a restrictive NetworkPolicy and suitable egress controls. Browser tests need access to the application under test, but should not have unrestricted access to internal infrastructure.

The exact CPU and memory values should be tuned using representative browser workloads. A single headless browser may fit comfortably within these limits, but parallel tests, multiple browsers and heavy pages can need substantially more.

Step 5 — Run the approved command and collect reports

Your worker entrypoint should resolve the manifest's command ID through an allowlist, execute it with a fixed working directory and a timeout, and collect the result.

For example, for a Maven project that uses the wrapper:

./mvnw -B -ntp test

If the repository does not include a Maven wrapper, use the pinned Maven version in the runtime image:

mvn -B -ntp test

Maven Surefire runs tests during the test lifecycle phase and normally produces XML reports under target/surefire-reports.

Don't assume every repository uses mvn test as its final execution command. Integration tests may be configured under another lifecycle phase, profile or plugin. Your manifest should capture the repository's established CI command.

The worker should return a structured result such as:

{
  "job_id": "job-123",
  "exit_code": 1,
  "status": "failed",
  "tests_passed": 8,
  "tests_failed": 1,
  "tests_skipped": 0,
  "report_artifact": "artifact://job-123/surefire-reports",
  "failure_category": "assertion_failure"
}

These values are illustrative. The worker must derive them from the actual process exit code and parsed reports, not from model interpretation alone.

Step 6 — Feed failures back into LangGraph

Once execution finishes, your workflow can branch on the result.

I would initially allow at most two or three repair attempts. A repair must never silently weaken assertions, remove failing tests or change unrelated production code just to obtain a successful run.

For a Selenium or Playwright failure, capture enough information to distinguish an actual application defect from an invalid selector, stale test data, an environment outage, a browser mismatch or an execution timeout.

Step 7 — Handle dependencies and credentials correctly

A few things commonly cause failures even when the generated script is correct:

The application under test is unreachable from the AKS subnet.
The test environment has not been deployed or is unavailable.
Browser binaries don't match the library version.
The worker cannot download dependencies because egress is restricted.
Required test credentials or configuration are missing.
Tests rely on data created by another test or a previous run.

Resolve dependencies during image preparation or through a controlled internal artifact repository. Inject only the credentials required for the particular test job, and never expose them to Copilot or include them in logs and reports.

3. Do you need a separate image for every framework?

Not necessarily. Start with two image families, then specialize only when the actual repositories require it.

Repository requirement	Recommended execution image
Selenium + Java + Maven	Selenium runner
Playwright + Java + Maven	Playwright runner
Selenium + Java + Gradle	Selenium runner with Gradle support
Playwright + Java + Gradle	Playwright runner with Gradle support
Selenium Grid / remote browser	Java runner plus access to an approved Grid
Repository-specific dependencies or custom plugins	A validated repository-specific image, if necessary

You can eventually use a common base image and add runtime-specific layers. The objective is to avoid rebuilding an image for every test while preserving version compatibility and isolation.

My recommended MVP

Implement this in the following order:

Build one Selenium Java/Maven execution image using a representative repository.
Build one Playwright Java/Maven execution image.
Implement a worker entrypoint that runs only approved commands and exports structured test reports.
Create an ephemeral AKS Job per execution attempt.
Integrate the worker with your LangGraph repair loop.
Add the second build tool, custom runtime images and repository-specific configurations as needed.

The main takeaway: Maven is your build and test orchestrator, not the whole execution environment. Your actual runtime is the combination of the JDK, build tool, test framework, browser and OS dependencies, application connectivity, test configuration, and a secure container that runs them.

That containerized runtime is what allows your generation agent to support many different automation repositories without giving the agent unrestricted execution access.

This might also interest you.

Azul Systems, Inc.

Self-Supported vs. Azul Core

Compare self-supported OpenJDK and Azul Core on speed, regressions, zero-days, cost and more.

Ad
More options
Is there a better way? Maybe i can spin up sandboxed environment within the same container as my generation agent one? Or keep these in the same pod? 

Yes — there is a potentially better approach for your AKS deployment: keep the generation agent and execution worker in the same Kubernetes Pod, but run test execution inside a separately isolated sandbox. This can reduce orchestration and artifact-transfer overhead, but only if you implement the isolation correctly.

The important distinction is that same Pod does not have to mean same container, and same container does not have to mean same security boundary. For your system, I'd evaluate three options.

1. Compare the deployment options

Option A — Same container, sandboxed subprocess

(Least isolation)

Copilot and Java/browser tests run in one container, with tests launched through a sandbox runtime such as gVisor or another appropriate isolation mechanism.

Lowest orchestration overhead, but the sandbox must be a genuine security boundary. Ordinary subprocesses, user permissions or Python wrappers are insufficient for untrusted code.

Option B — Same Pod, separate containers

(Practical compromise)

One container runs the generation agent; another runs the execution service. They can exchange a patch and test results through a shared, tightly controlled volume or local service.

Simpler coordination, but containers in the same Pod share the network namespace and Pod-level security context. A shared volume can also undermine isolation if writable access is not controlled.

Option C — Separate generation and execution Pods

(Strongest default boundary)

Generation and test execution use separate workloads, service accounts, network policies and resource limits. The execution Pod is disposable.

More coordination and transfer overhead, but clearer security boundaries, independent scaling and failure isolation.

My recommendation is to start with Option C for production, while considering Option B if measured latency or workload overhead justifies the compromise. I would not put arbitrary generated Java and browser execution directly into the Copilot container merely to save a Pod.

2. A better option specifically for AKS: Kata sandboxed execution Pods

Since you're deploying on Azure, there is a useful middle ground between a standard container and running a separate VM manually: AKS Pod Sandboxing with Kata Containers.

AKS can run a Pod inside a lightweight VM with its own guest kernel. You can select this runtime using runtimeClassName: kata-vm-isolation, subject to the cluster's supported configuration.

The architecture I'd choose is:

Generation Pod

Copilot SDK · repository context · controlled tools · generated patch

Execution Job — Kata-isolated Pod

Dedicated guest kernel · Java/Maven · Selenium or Playwright · browser · test reports

(Stronger execution isolation)

Artifact store + LangGraph

Results → repair decision → rerun or human review

Example execution Pod configuration:

apiVersion: batch/v1
kind: Job
metadata:
  generateName: isolated-test-
  namespace: test-execution
spec:
  backoffLimit: 0
  activeDeadlineSeconds: 900
  template:
    spec:
      runtimeClassName: kata-vm-isolation
      restartPolicy: Never
      automountServiceAccountToken: false
      containers:
        - name: test-runner
          image: your-acr.azurecr.io/java-playwright-runner:v1
          command: ["/opt/runner/execute-job"]
          resources:
            requests:
              cpu: "1"
              memory: "2Gi"
            limits:
              cpu: "2"
              memory: "4Gi"

This is an illustrative Job manifest; you still need to configure writable volumes, the trusted job-artifact transfer, network policy, image access, and the worker entrypoint. The AKS node pool must support Kata sandboxing; it isn't enabled simply by adding the runtime class to any arbitrary cluster. Review the official AKS sandboxing setup.

One important nuance: Kata isolates a Pod from the host and other Pods; it does not create a separate VM boundary for each container within that Pod. So if generation and execution containers share one Pod, Kata does not give the execution container its own independent kernel boundary from the generation container.

3. How to optimize cost and latency

You can preserve most of the security benefits without paying the full cost of creating every resource from scratch for every retry:

Build and cache the runtime images in ACR.
Reuse the same pinned repository checkout and prepared artifact within a job's bounded repair loop.
Keep a pool of pre-provisioned execution capacity if measurements show Pod startup is a bottleneck.
Reuse a sandbox only within the same trusted job lifecycle, cleaning its workspace between attempts as appropriate. Never share a writable workspace across unrelated jobs.
Scale execution independently from generation, so browser-heavy workloads don't consume all the capacity needed for LLM calls.

Kata Pods have additional resource and startup overhead compared with standard containers, so benchmark actual Selenium and Playwright workloads before enabling it for every job.

Final recommendation

For your first production version, I'd use a generation Pod plus an ephemeral, Kata-isolated execution Pod, with a controlled artifact handoff between them. Keep the execution image prebuilt, and let LangGraph orchestrate repair attempts.

If you later find that Pod creation latency is materially affecting throughput, benchmark a same-Pod design with separate containers—but treat it as a deliberate reduction in isolation, not as equivalent security.

This might also interest you.
