# Entra Agent ID from AWS AgentCore — PoC

A proof-of-concept that invokes an **AWS Bedrock AgentCore**-hosted AI agent from an
**Entra-secured single-page app**, where the agent holds its **own Microsoft Entra
Agent Identity** and performs the Entra Agent ID **FMI/OBO token exchange in-process**
(MSAL Python) to call downstream Entra-protected resources on behalf of the signed-in
user.

A browser SPA signs the user in with MSAL.js and sends their Entra access token (Bearer,
header-only) to the AgentCore Runtime. AgentCore's **built-in JWT authorizer** validates
that token at the front door (issuer + audience + scope) before any container code runs.
Inside the container, the agent runs a two-stage Entra Agent ID exchange — **FMI**
(Blueprint → Agent Identity) then **OBO** (user-delegated) — using **MSAL Python**, and
calls the downstream **Echo API** and (optionally) the **Microsoft Graph MCP** server with
the resulting user-bound tokens.

> **in-process via MSAL Python**. (An API Gateway HTTP API *is* used — but only as the
> front door for the downstream Echo API, see [What's deployed](#whats-deployed-aws).)

---

## Workshop demonstration

This repository contains the end-to-end exercise demonstrated in the workshop:
an AI agent runs in Amazon Bedrock AgentCore, accepts a request from a Microsoft
Entra-authenticated user, and calls an Entra-protected Echo API on that user's behalf.
The AWS workload does not store a reusable Microsoft Entra client secret.

The exercise is completed in five stages:

1. **Prepare AWS.** Protect the root account, configure IAM Identity Center for the
   human operator, enable outbound web identity federation, and establish cost
   visibility.
2. **Create the Entra identity model.** Create an Agent ID Blueprint and linked child
   Agent Identity, register the browser SPA and Echo API, expose `agent.invoke` and
   `access-as-user`, grant consent, and configure delegated permission inheritance.
3. **Deploy the AWS application.** Package the Python runtime, upload it to S3, and
   deploy AgentCore, its IAM execution role, API Gateway, Lambda, and logging through
   CloudFormation.
4. **Federate AWS with Entra.** Add a federated identity credential to the Blueprint
   that trusts the exact AgentCore execution-role ARN. At runtime, AWS STS issues a
   five-minute role assertion, Entra maps it to the child Agent Identity, and OAuth
   on-behalf-of produces the Echo API token.
5. **Test and prove the flow.** Sign in through the SPA, invoke the agent, call the
   protected Echo API, inspect safe identity claims, and correlate the request with
   CloudWatch logs.

The completed flow demonstrates that:

- AWS proves the identity of the running workload using a short-lived signed token.
- Microsoft Entra identifies the specific child Agent Identity.
- The final API token preserves the signed-in user's context and carries the
  `access-as-user` delegated scope.
- The Echo API can inspect validated user and agent-related claims and apply
  agent-aware authorization or business logic.
- No long-lived Microsoft Entra client secret is stored in AWS.

Use [`docs/entra-setup-guide.md`](docs/entra-setup-guide.md) for the Entra objects and
permissions, then [`docs/deployment-guide.md`](docs/deployment-guide.md) for deployment
and end-to-end testing. Copy `.env.example` to `.env` and
`spa/msal-config.js.sample` to `spa/msal-config.js`; both real configuration files are
excluded from Git.

---

## Architecture topology

![Entra Agent ID from AWS AgentCore architecture](docs/architecture.svg)

The editable source is available at
[`docs/architecture.excalidraw`](docs/architecture.excalidraw) and can be opened in
[Microsoft's internal Excalidraw instance](https://aka.ms/excalidraw).

The diagram reflects the **deployed** stack. The agent never receives a credential it can
hand to the model — see [Security notes](#security-notes).

### How to explain the architecture in an interview

Start with the security goal: **an AI agent running in AWS must call Microsoft-protected
APIs for a signed-in user, without storing a reusable Microsoft Entra client secret.**

1. The browser SPA signs in the user with MSAL.js and PKCE, then obtains a user token
   containing the `agent.invoke` delegated scope.
2. The SPA sends that token to the AgentCore Runtime. AgentCore's built-in authorizer
   validates its issuer, audience, and scope before the Python container can run.
3. The container asks AWS STS for a short-lived signed assertion proving which IAM
   execution role is running the workload.
4. During **FMI**, Entra validates that AWS assertion against the federated credential
   on the Agent Identity Blueprint and issues **T1**, which represents the child Agent
   Identity.
5. During **OBO**, the agent combines T1 with the incoming user token. Entra issues a
   downstream token (**TR**) that represents both the agent and the delegated user.
6. The agent sends a resource-specific TR to the Echo API or Microsoft Graph MCP. Each
   resource validates its own audience and permissions independently.

The shortest summary is: **the user token proves who the user is, the AWS assertion
proves which workload is running, FMI identifies the agent, and OBO creates the final
user-delegated token for the downstream API.**

---

## Components

| Component | Role | Where it runs | Tech |
|---|---|---|---|
| **SPA** (`spa/`) | User sign-in + chat UI; acquires the Entra user token and invokes the AgentCore Runtime (header-only Bearer) | Browser (static files; nginx container for local serving) | MSAL.js 3.28.1, Bootstrap 5.3.3, `marked` + DOMPurify |
| **AgentCore Runtime** (`AgentRuntime`) | Hosts the agent; its **built-in JWT authorizer** validates the inbound Entra token before the container runs | AWS Bedrock AgentCore (managed), eu-central-1 | `AWS::BedrockAgentCore::Runtime` |
| **Agent code** (`agent/src/agent.py`) | Extracts the inbound token, runs the in-process FMI→OBO exchange, calls the Echo API tool and (optional) MCP tools | Inside the AgentCore Runtime container (Python 3.12) | MSAL Python, Strands Agents, `bedrock-agentcore` SDK, Bedrock (Nova Micro) |
| **Agent Identity Blueprint** | Entra object holding the AWS execution role's federated identity credential | Microsoft Entra ID | Agent Identity Blueprint + FIC |
| **Agent Identity** | The agent's *own* derived identity (child of the Blueprint) | Microsoft Entra ID | Entra Agent Identity object |
| **Echo API** (`EchoHttpApi` + `EchoLambdaFunction`) | Downstream Entra-protected REST API the agent calls on the user's behalf | AWS API Gateway v2 HTTP API + Lambda | API Gateway native **JWT authorizer** + inline Python Lambda |
| **Microsoft MCP Server for Enterprise** | Instance of [Microsoft MCP Server for Enterprise](https://learn.microsoft.com/en-us/graph/mcp-server/get-started) ; tools surfaced to the agent | External (Microsoft-hosted) | MCP over Streamable HTTP (SSE fallback) |
| **IAM execution role** (`RuntimeExecutionRole`) | Runtime permissions: invoke Bedrock, mint a constrained AWS web identity token, write logs, and read the S3 artifact | AWS IAM | `AWS::IAM::Role` |
| **Microosft Entra ID tenant** | Issues all tokens; Performs all authorizations | Microsoft Entra ID | — |

---

## Authorizers & token validation

There are **two independent authorization boundaries** in this PoC. They validate
different tokens, for different audiences, at different points in the flow. Confusing them
is the single most common source of debugging time.

### 1. Front door — AgentCore Runtime built-in JWT authorizer

This is the **AWS-native** `CustomJWTAuthorizerConfiguration` on the
`AWS::BedrockAgentCore::Runtime` resource — **not** a custom Lambda authorizer. It runs
**before** any container code, at the AgentCore front door, and validates exactly three
things about the inbound Entra **user** token:

| Config (`stack.yaml`) | Claim validated | Value |
|---|---|---|
| `DiscoveryUrl` | `iss` | Entra OIDC metadata for the tenant (`…/v2.0/.well-known/openid-configuration`) |
| `AllowedAudience` | `aud` | `AgentCoreAppClientId` (the Blueprint Client ID) — both bare-GUID and `api://{guid}` forms accepted |
| `AllowedScopes` | `scp` | `AgentCoreAllowedScope` / `AGENTCORE_ALLOWED_SCOPE` |

- A **rejection is an HTTP 403 with no container log** — the request never reaches the agent, so absence of a runtime log line is the signature of a front-door rejection (as opposed to an in-agent error, which *does* log).
- `AllowedScopes` compares the scope **value** from `scp`, such as `agent.invoke`;
  it is not the full `api://{client-id}/agent.invoke` scope URI.
- The runtime also sets `RequestHeaderConfiguration.RequestHeaderAllowlist: [Authorization]` so the validated `Authorization` header is **forwarded** to the container. This is what makes the in-process OBO possible (see [Status](#status--known-issues) — "Path A").

### 2. Downstream — Echo API JWT authorizer

The deployed Echo API has its **own** authorizer, completely separate from the AgentCore
one. In the deployed stack this is an **API Gateway v2 native JWT authorizer**
(`EchoJwtAuthorizer`) on the `POST /echo` route:

| Config (`stack.yaml`) | Claim validated | Value |
|---|---|---|
| `JwtConfiguration.Audience` | `aud` | `EchoApiClientId` (the Echo API's own app reg client ID) |
| `JwtConfiguration.Issuer` | `iss` | `https://login.microsoftonline.com/{tenant}/v2.0` |

`GET /health` is unauthenticated. The token this authorizer validates is the **downstream TR** the agent minted via OBO (audience = Echo API), **not** the inbound user token (audience = Blueprint).

**Two boundaries, two audiences:** the inbound user token is audienced to the **Blueprint** (`agent.invoke`); the downstream token is audienced to the **Echo API**. The agent's job is to convert the first into the second via MSAL Python.

---

## Token flow (the three legs)

1. **SPA acquires the user token.** MSAL.js signs the user in (auth code + PKCE) and calls `acquireTokenSilent`/`acquireTokenPopup` for scope `api://{AgentCoreAppClientId}/agent.invoke` (`agent.invoke`). The resulting Entra user JWT (aud = Blueprint) is sent to the runtime **header-only**: `Authorization: Bearer …` plus a `{ "message": … }` body. The token is never placed in the body for real traffic.

2. **AgentCore authorizer validates it.** The front-door JWT authorizer checks `iss` + `aud` + `scp` (see above). On success the validated `Authorization` header is forwarded to the container; `agent.py` `_extract_inbound_token()` reads the token from it.

3. **The agent runs FMI then OBO, in-process.** Using **MSAL Python**:
   - **Stage 1 — FMI (T1):** the persistent **Blueprint** `ConfidentialClientApplication` calls `acquire_token_for_client(scopes=["api://AzureADTokenExchange/.default"], fmi_path=AGENT_IDENTITY_ID)`. Entra returns **T1**, an assertion proving the Blueprint→Agent-Identity relationship. (`fmi_path` is MSAL Python's dedicated kwarg — **not** `extra_body_params`, which is an MSAL.js concept.)
   - **Stage 2 — OBO (TR):** the persistent **agent** `ConfidentialClientApplication` (whose `client_assertion` is a *callable* that lazily produces T1) calls `acquire_token_on_behalf_of(user_assertion=<inbound user JWT>, scopes=[<downstream scope>])`. Entra returns **TR**, a token scoped to the target resource and bound to the original user (`sub` preserved).
   - The agent calls the **Echo API** (`POST /echo`, `Authorization: Bearer TR`) and/or the **MCP** server with TR; each downstream resource validates its own token.

   **Caching:** both CCAs are built **once per process** so their in-memory MSAL caches survive across invocations. Before each network exchange the agent tries `acquire_token_silent` keyed to the user's `oid` (so one user can never be served another's cached token); on a miss it performs the OBO network call, which also populates the cache. Tokens are cached per `(user, scopes)`, so Echo and MCP cache independently.

   **Error surfacing (for DEMO only):** a token-acquisition failure (FMI or OBO, any scope) raises `TokenAcquisitionError` and is surfaced to the end user **verbatim** (the non-secret `AADSTS…` error/description only — never a token or signature). For the Echo path the LLM is instructed to relay the marker text exactly; for the MCP path the agent returns `{"status":"error", …}` rather than silently falling back to Echo-only.

---

## What's deployed (AWS)

The single CloudFormation template `infra/cloudformation/stack.yaml` (stack name `agentid-poc`, region `eu-central-1`) deploys:

| Logical ID | Type | Purpose |
|---|---|---|
| `RuntimeExecutionRole` | `AWS::IAM::Role` | AgentCore runtime perms: `bedrock:InvokeModel*`, constrained `sts:GetWebIdentityToken`, CloudWatch Logs, and S3 get/list on the artifact bucket |
| `AgentRuntime` | `AWS::BedrockAgentCore::Runtime` | The hosted agent. Code from S3 (`agent.zip`), Python 3.12, `app.py` entrypoint. **Custom JWT authorizer** (iss/aud/scp), `RequestHeaderAllowlist: [Authorization]`, env vars (tenant, Blueprint/agent IDs, Echo URL/scope, MCP URL/scope, model) |
| `AgentRuntimeEndpoint` | `AWS::BedrockAgentCore::RuntimeEndpoint` | The invokable endpoint (qualifier `default`) |
| `EchoLambdaRole` | `AWS::IAM::Role` | Echo Lambda basic execution role |
| `EchoLambdaFunction` | `AWS::Lambda::Function` | Inline Python echo handler; echoes `message` and the caller from the JWT authorizer claims |
| `EchoHttpApi` | `AWS::ApiGatewayV2::Api` | HTTP API fronting the Echo Lambda |
| `EchoJwtAuthorizer` | `AWS::ApiGatewayV2::Authorizer` | **Native JWT authorizer** (aud = Echo API client ID, iss = tenant `/v2.0`) |
| `EchoLambdaIntegration` | `AWS::ApiGatewayV2::Integration` | AWS_PROXY integration |
| `EchoApiRoute` | `AWS::ApiGatewayV2::Route` | `POST /echo` (JWT-authorized) |
| `EchoApiHealthRoute` | `AWS::ApiGatewayV2::Route` | `GET /health` (open) |
| `EchoApiStage` | `AWS::ApiGatewayV2::Stage` | `$default`, auto-deploy |
| `EchoLambdaPermission` | `AWS::Lambda::Permission` | Lets API Gateway invoke the Lambda |

**Not used / not deployed:** ❌ no Fargate or ECS, ❌ no Entra SDK sidecar container,
❌ no VPC / NAT / subnets, ❌ no Cognito. (An API Gateway HTTP API **is** used — but only as
the Echo API's front door, not for the SPA→agent path. The SPA invokes the AgentCore
Runtime directly via the Bedrock AgentCore data-plane endpoint.)

Key outputs: `RuntimeArn`, `RuntimeId`, `EndpointId`, `EchoApiUrl`, `ExecutionRoleArn`,
`ArtifactBucket`, `ArtifactKey`.

---

## Entra objects

Per [`docs/entra-setup-guide.md`](docs/entra-setup-guide.md):

| Object | Type | Role |
|---|---|---|
| **SPA app registration** (`agentid-poc-spa`) | App registration (public client) | The SPA's MSAL identity; requests the `agent.invoke` token audienced to the Blueprint |
| **Agent Identity Blueprint** | Blueprint object | Holds a FIC that trusts the AWS account issuer and the exact AgentCore execution-role ARN. Created via **Entra admin center → Agents → Blueprints → New** |
| **Agent Identity** | Entra Agent Identity object (child of the Blueprint) | The agent's *own* identity; its object ID is the `fmi_path` / `AGENT_IDENTITY_ID`. **Not** an app registration |
| **Echo API app registration** (`agentid-poc-echo-api`) | App registration | Exposes the Echo scope and is the audience the downstream TR is validated against |

---

## Repo layout

| Path | Role |
|---|---|
| [agent/](./agent/) | Python AgentCore agent — in-process MSAL FMI/OBO, Echo tool, optional MCP wiring |
| [spa/](./spa/) | MSAL.js single-page app + Bootstrap chat UI (msal-config.js is gitignored; see .sample) |
| [infra/](./infra/) | CloudFormation — cloudformation/stack.yaml is the single deployed template |
| [scripts/](./scripts/) | PowerShell build/deploy/auth/teardown helpers |
| [docs/](./docs/) | Deeper docs: deployment guide, Entra setup, architecture assessment, findings |

---

## One-time AWS and Blueprint federation setup

Outbound web identity federation is an AWS **account setting**, not a native
CloudFormation resource. Enable it once outside the stack.
The operator needs `iam:EnableOutboundWebIdentityFederation` and
`iam:GetOutboundWebIdentityFederationInfo`.

**CLI:**

```powershell
aws iam enable-outbound-web-identity-federation --profile agentid-poc

$issuer = aws iam get-outbound-web-identity-federation-info `
    --profile agentid-poc `
    --query IssuerIdentifier `
    --output text
```

If it is already enabled, the first command returns `FeatureEnabled`; use the second
command to retrieve the existing issuer.

**Console:** Open **IAM → Access management → Account settings → Outbound identity
federation**, select **Enable**, and copy the token issuer URL.

Deploy the stack, then obtain the exact FIC subject from its `ExecutionRoleArn` output:

```powershell
$subject = aws cloudformation describe-stacks `
    --stack-name agentid-poc `
    --profile agentid-poc `
    --query "Stacks[0].Outputs[?OutputKey=='ExecutionRoleArn'].OutputValue | [0]" `
    --output text
```

On the **Agent Identity Blueprint application object**, add a federated identity
credential with these exact, case-sensitive values:

| FIC field | Value |
|---|---|
| Name | `aws-agentcore-runtime` |
| Issuer | `$issuer` (the account-specific `https://...tokens.sts.global.api.aws` URL) |
| Subject | `$subject` (the stack's exact execution-role ARN) |
| Audience | `api://AzureADTokenExchange` |

In the Entra admin center, open the Blueprint's management page, select **Credentials**
under **Developer settings**, open **Federated credentials**, and select **Add
credential → Other issuer**. The subject is the IAM role ARN, not an STS `assumed-role`
session ARN, AgentCore runtime ARN, or Agent Identity ID.

Issuer and subject are account/stack-specific. Recreate or update the FIC if the stack
name changes because the template derives the role name from the stack name.

## Build / deploy / invoke

Full walkthrough: [`docs/deployment-guide.md`](docs/deployment-guide.md). The short path
(PowerShell 7+, AWS CLI v2, Docker):

1. **Authenticate.** `aws sso login --profile agentid-poc` (see
   [`scripts/aws-auth.ps1`](scripts/aws-auth.ps1)).
2. **Enable AWS outbound identity federation** once and record its issuer as described
   above.
3. **Deploy.** [`scripts/aws-deploy.ps1`](scripts/aws-deploy.ps1) builds the agent ZIP
   ([`scripts/build-zip.ps1`](scripts/build-zip.ps1)), uploads it to S3, and runs
   `aws cloudformation deploy`. Required params come from `.env` or the command line
   (tenant, agent identity, Blueprint client ID, Echo API client ID, MCP URL/scope,
   AgentCore app client ID).
4. **Add the Blueprint FIC** using the account issuer and the stack's
   `ExecutionRoleArn` output as described above.
5. **⚠️ Repoint the SPA after every deploy.** `aws-deploy.ps1` mints a **new
   timestamped runtime name** (`agentid_core_<timestamp>`) on each deploy and CFN
   *replaces* the runtime, so the old runtime ARN is deleted. **You must paste the printed
   `agentCoreEndpoint` URL into [`spa/msal-config.js`](spa/msal-config.js.sample) after
   every deploy** — otherwise the SPA invokes a deleted runtime and fails with
   `No endpoint or agent found with qualifier 'default'`.
6. **Invoke.** Sign in to the SPA and chat — all testing happens over REST via
   the SPA. Teardown: [`scripts/aws-destroy.ps1`](scripts/aws-destroy.ps1).

---

## Status / known issues

- ✅ **Inbound auth confirmed working** Adding `RequestHeaderAllowlist:
  [Authorization]` to the runtime delivers the validated Entra **user** token to the
  container. 
- ✅ **AgentCore JWT authorizer confirmed** (iss + aud + scp; `AllowedClients` intentionally
  not set).
- ✅ **AWS API Gateway v2 JWT authorizer for Lambda confirmed** (iss + aud; `AllowedClients` intentionally
  not set).

---

## Security notes

- **Tokens are never logged.** Only non-secret routing claims (`iss`, `aud`, `appid`, `azp`,
  `scp`, plus a `token_source` and the user's `oid` as a cache key) are emitted — never a
  token, header value, or signature.
- **Tokens are never passed to the LLM.** Tools return only the API response payload; the
  inbound token lives in a module-level `_current_token` that is **cleared in a `finally`
  block after every invoke** and never persists between calls.
- **No long-lived Blueprint secret exists in AWS.** The runtime uses its temporary role
  credentials to request a five-minute, RS256 AWS assertion from regional STS. IAM limits
  the assertion audience to `api://AzureADTokenExchange` and its lifetime to 300 seconds.
- For FIC troubleshooting, deploy with `-EnableAuthDiagnostics`. CloudWatch then records
  only `alg`, `kid`, `typ`, `iss`, `sub`, `aud`, `iat`, `exp`, lifetime, and `jti` under
  `AWS assertion metadata`. The raw assertion and signature are never logged. Redeploy
  without the switch after troubleshooting.

---

## Documentation

- [Deployment guide](docs/deployment-guide.md) — step-by-step deploy
- [Entra setup guide](docs/entra-setup-guide.md) — app registrations, Blueprint, Agent Identity
- [Architecture simplification assessment](docs/architecture-simplification-assessment.md) — why the sidecar was removed
- [MSAL Python FIC/FMI findings](docs/msal-python-fic-fmi-findings.md)
- [SPA local dev notes](docs/spa-local-dev-notes.md)
- [Architecture decisions](decisions.md)
