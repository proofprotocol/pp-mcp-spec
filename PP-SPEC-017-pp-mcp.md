# PP-SPEC-017: PP-MCP — Proof Protocol MCP Interface Standard

**Specification ID:** PP-SPEC-017  
**Status:** DRAFT  
**Version:** 0.1.0  
**Date:** 2026-07-16  
**Author:** Nebulonium, Inc. / HACKERverse®  
**License:** CC BY 4.0  
**Repository:** https://github.com/proofprotocol/prooftwin  

---

## 1. Abstract

PP-MCP defines the standard interface contracts between the components of the Proof Protocol attestation stack: the Attack Corpus Runner (ACR), AgenTwin (agent under test), ProofTwin (attestation layer), and the Agent Security Control (ASC). Any component that implements this interface is interoperable with the Proof Protocol stack and eligible for ProofStamp certification testing.

PP-MCP is built on the Model Context Protocol (MCP) and extends it with session lifecycle, behavioral attestation, and proof record fields required for cryptographically anchored SUT-level certification.

---

## 2. Scope

This specification covers:

- The MCP interface between AgenTwin and ProofTwin
- The MCP interface between ProofTwin and the ASC
- The session control interface between the ACR and ProofTwin
- The verdict emission interface between ProofTwin and Arena7
- The configuration schema for all components

This specification does not cover:

- Internal implementation of any component
- ProofRegister chain protocol (see PP-SPEC-003)
- NIST beacon commitment protocol (see PP-SPEC-004)
- Proof Efficacy Score calculation (see PP-SPEC-006)

---

## 3. Definitions

All terms defined in PP-SPEC-015 apply. Additional terms:

**PP-MCP Server** — Any MCP server that implements this specification. ProofTwin is the reference PP-MCP server.

**PP-MCP Client** — Any MCP client that connects to an PP-MCP server. AgenTwin is the reference PP-MCP client.

**Session** — A bounded attestation period. Begins with NIST beacon pre-commitment, ends with PTBR emission. All tool calls within a session are correlated to a single proof record.

**Tool Profile** — A machine-readable declaration of the tool surface an agent or ASC exposes. Used by the ACR to determine applicable test cases and by ProofTwin to validate tool call scope.

**Case Correlation ID** — A per-case identifier passed by the ACR to ProofTwin at session start, enabling ProofTwin to correlate each tool call to the specific attack case that triggered it.

---

## 4. Component Interface Map

```
┌─────────────────────────────────────────────────────────┐
│                    PROOF PROTOCOL SUT                   │
│                                                         │
│  ACR ──[PP-MCP Session Control]──▶ PROOFTWIN           │
│                                         │               │
│  AGENTWIN ──[PP-MCP Client]──▶ PROOFTWIN               │
│                                         │               │
│                           PROOFTWIN ──[PP-MCP Proxy]──▶ ASC │
│                                         │               │
│                           PROOFTWIN ──[HV-Verdict]──▶ ARENA7 │
└─────────────────────────────────────────────────────────┘
```

---

## 5. PP-MCP Session Control Interface (ACR → ProofTwin)

The ACR controls the ProofTwin session lifecycle via a REST API exposed by ProofTwin.

### 5.1 Session Start

```
POST /hv/session/start
Content-Type: application/json

{
  "acr_id": "aeb-gauntlet",
  "acr_version": "2.0.0",
  "corpus_id": "agent-egress-bench",
  "corpus_version": "v2.0.0",
  "corpus_hash": "<sha256>",
  "case_count": 196,
  "tool_profile_hash": "<sha256>",
  "run_id": "<uuid>"
}

Response 200:
{
  "session_id": "<uuid>",
  "nist_beacon": {
    "pulse_index": 1852788,
    "timestamp": "2026-07-16T14:00:00.000Z",
    "output_value": "<hex>",
    "commitment_hash": "<sha256>"
  },
  "baseline_status": "pending"
}
```

### 5.2 Case Annotation

The ACR notifies ProofTwin before each case fires, enabling tool call correlation:

```
POST /hv/session/{session_id}/case
Content-Type: application/json

{
  "case_id": "url-dlp-001",
  "category": "url",
  "expected_verdict": "block",
  "seq": 1
}

Response 200:
{
  "acknowledged": true
}
```

### 5.3 Session End

```
POST /hv/session/{session_id}/end
Content-Type: application/json

{
  "acr_summary": {
    "containment": 1.0,
    "detection": 1.0,
    "false_positive_rate": 0.06,
    "applicable": 176,
    "errors": 0
  }
}

Response 200:
{
  "ptbr_id": "<uuid>",
  "behavioral_verdict": "CLEAN",
  "composite_verdict": "CERTIFIED",
  "proofregister_id": "PR-2026-XXXXX"
}
```

---

## 6. PP-MCP Client Interface (AgenTwin → ProofTwin)

AgenTwin connects to ProofTwin as a standard MCP client. ProofTwin exposes the same tool surface as the downstream ASC.

### 6.1 Tool Discovery

AgenTwin calls `tools/list` on ProofTwin. ProofTwin returns the same tool list it received from the ASC downstream, ensuring AgenTwin sees the correct tool surface.

```json
{
  "tools": [
    {
      "name": "fetch",
      "description": "Fetch a URL",
      "inputSchema": {
        "type": "object",
        "properties": {
          "url": { "type": "string" }
        },
        "required": ["url"]
      }
    }
  ]
}
```

### 6.2 Tool Call Passthrough

AgenTwin calls tools on ProofTwin using standard MCP `tools/call`. ProofTwin:

1. Logs the call (timestamp, tool name, arguments, active case_id)
2. Forwards the call to the ASC unchanged
3. Logs the response (status, body, latency)
4. Returns the response to AgenTwin unchanged

ProofTwin MUST NOT modify any tool call or response. It is a transparent proxy with a logging layer.

### 6.3 PP-MCP Headers

AgenTwin MUST include the following MCP metadata on every tool call:

```json
{
  "_meta": {
    "hv_session_id": "<session_id>",
    "hv_agent_id": "agentwin-01",
    "hv_agent_version": "1.0.0"
  }
}
```

ProofTwin uses these fields to correlate tool calls to sessions and agents.

---

## 7. PP-MCP Proxy Interface (ProofTwin → ASC)

ProofTwin connects to the ASC as a standard MCP client or HTTP proxy client, depending on the ASC's supported transport.

### 7.1 Supported Transports

| Transport | ProofTwin Support | Notes |
|-----------|-------------------|-------|
| MCP stdio | Required | Default for local ASC |
| MCP HTTP | Required | For remote ASC |
| HTTP proxy | Required | For fetch-proxy ASC |
| WebSocket | Optional | For WebSocket DLP ASC |

### 7.2 ASC Configuration

ProofTwin's ASC connection is configured in `config.yaml`:

```yaml
asc:
  transport: mcp_stdio          # mcp_stdio | mcp_http | http_proxy
  cmd: "<asc-binary> mcp proxy --config <config>"   # for mcp_stdio
  addr: "127.0.0.1:9991"        # for mcp_http or http_proxy
  scan_addr: "127.0.0.1:9990"   # scan/verdict API if separate
  scan_token: "<token>"
  timeout_ms: 15000
```

### 7.3 ASC Verdict Capture

For ASCs that expose a scan API, ProofTwin queries the verdict after each tool call:

```
GET /scan/verdict?case_id={case_id}&session_id={session_id}

Response:
{
  "verdict": "block",
  "block_reason": "credential_exfil",
  "confidence": 0.99,
  "scanner": "url_dlp",
  "latency_ms": 8
}
```

This verdict is included in the PTBR wire_layer record alongside ProofTwin's behavioral observation.

---

## 8. HV-Verdict Interface (ProofTwin → Arena7)

ProofTwin posts the completed PTBR to Arena7 at session end.

### 8.1 PTBR Submission

```
POST http://{arena7_host}:{port}/api/prooftwin/ptbr
Content-Type: application/json
Authorization: Bearer <arena7_token>

{ <full PTBR as defined in PP-SPEC-015 Section 6> }

Response 200:
{
  "proof_id": "PR-2026-XXXXX",
  "block_number": 20886,
  "anchored_at": "2026-07-16T14:05:12.000Z",
  "proofregister_url": "https://chain.proofregister.com/proof/PR-2026-XXXXX"
}
```

### 8.2 Streaming Verdict (Optional)

For long-running sessions, ProofTwin MAY stream per-case behavioral verdicts to Arena7 as cases complete:

```
POST /api/prooftwin/case-verdict
{
  "session_id": "<uuid>",
  "case_id": "url-dlp-001",
  "seq": 1,
  "behavioral_verdict": "CLEAN",
  "tool_calls": [ ... ],
  "timestamp_utc": "2026-07-16T14:01:23.456Z"
}
```

---

## 9. Tool Profile Schema

The Tool Profile declares what tools an agent or ASC exposes. Used by the ACR to determine applicable cases and by ProofTwin to validate scope.

```json
{
  "schema_version": "HV-ATP-1.0",
  "component_id": "agentwin-01",
  "component_type": "agent",
  "version": "1.0.0",
  "tools": [
    "fetch",
    "read_file",
    "execute_shell",
    "mcp_tool_call"
  ],
  "transports": [
    "mcp_stdio",
    "http_proxy"
  ],
  "capabilities": [
    "url_dlp",
    "request_body_scanning",
    "mcp_input_scan"
  ],
  "profile_hash": "<sha256 of this document>"
}
```

---

## 10. Configuration Reference

### 10.1 ProofTwin config.yaml

```yaml
prooftwin:
  version: "0.1.0"
  host: "0.0.0.0"
  port: 9995
  log_level: "info"

session:
  baseline_prompt_set: "./baseline/v1.0.0"
  baseline_hash_verify: true
  nist_beacon_url: "https://beacon.nist.gov/beacon/2.0/pulse/last"

asc:
  transport: "mcp_stdio"
  cmd: "<asc-binary> mcp proxy --config <asc-config>"
  scan_addr: "127.0.0.1:9990"
  scan_token: "<token>"
  timeout_ms: 15000

arena7:
  host: "172.31.7.168"
  port: 9100
  token: "<arena7_token>"
  ptbr_endpoint: "/api/prooftwin/ptbr"

proofregister:
  host: "172.31.4.236"
  port: 8547
```

---

## 11. Compliance Requirements

A component claiming PP-MCP compatibility MUST:

**ACR:**
- POST session start before firing any cases
- POST case annotation before each case
- POST session end with ACR summary after all cases complete
- Include corpus hash in session start

**AgenTwin / Agent:**
- Connect to ProofTwin as an MCP client
- Include `_meta.hv_session_id` on every tool call
- Not modify or filter tool calls based on content

**ASC:**
- Accept MCP tool calls from ProofTwin
- Return structured verdicts including block reason and scanner classification
- Expose scan API on a queryable endpoint

**ProofTwin:**
- Fetch NIST beacon before any traffic
- Capture behavioral baseline before adversarial session
- Log every tool call with full arguments and response
- Emit PTBR to Arena7 at session end
- Not modify any tool call or response in transit

---

## 12. Versioning

PP-MCP is versioned independently of ProofTwin and other PP-SPEC documents. Interface changes that break compatibility increment the major version. Additive changes increment the minor version.

Current version: `PP-MCP/1.0`

All PP-MCP messages MUST include a version field:

```json
{ "hv_mcp_version": "1.0" }
```

---

## 13. Prior Art and Timestamps

This specification was authored by Nebulonium, Inc. and anchored to the NIST Randomness Beacon and Zenodo DOI registry on or before 2026-07-25.

---

## 14. Status and Roadmap

| Milestone | Target |
|-----------|--------|
| PP-SPEC-017 v0.1 published to Zenodo | 2026-07-25 |
| Reference ProofTwin implements PP-MCP/1.0 | 2026-08-01 |
| First third-party ASC certified via PP-MCP | 2026-09-01 |
| PP-MCP/1.0 stable | 2026-09-01 |

---

## References

- PP-SPEC-015: ProofTwin — Agent Behavioral Attestation Layer
- PP-SPEC-006: Proof Efficacy Score (PES)
- PP-SPEC-003: ProofRegister Chain Protocol
- PP-SPEC-004: NIST Beacon Commitment Protocol
- MCP Specification: https://modelcontextprotocol.io
- ProofRegister: https://chain.proofregister.com
- NIST Randomness Beacon: https://beacon.nist.gov

---

*© 2026 Nebulonium, Inc. Licensed under CC BY 4.0. ProofTwin, AgenTwin, ArenaTwin, ProofStamp, ProofRegister, HACKERverse, and Proof Economy are trademarks of Nebulonium, Inc.*
