> **Zenodo DOI:** Pending — Target publication 2026-07-25

# PP-SPEC-016 · HV-MCP — Proof Protocol MCP Interface Standard

**Document ID:** PP-SPEC-016  
**Version:** 0.1 - Draft  
**Status:** Draft  
**License:** CC BY 4.0  
**Maintained by:** Proof Economy™ Standards Alliance (PESA)  
**Repository:** https://github.com/proofprotocol/pp-mcp-spec  
**Published:** 2026-07-16  

---

## Abstract

HV-MCP defines the standard interface contracts between the components of the Proof Protocol attestation stack: the Attack Corpus Runner (ACR), AgenTwin (agent under test), ProofTwin (attestation layer), and the Agent Security Control (ASC).

Any component that implements this interface is interoperable with the Proof Protocol stack and eligible for ProofStamp™ certification testing.

HV-MCP is built on the Model Context Protocol (MCP) and extends it with session lifecycle, behavioral attestation, and proof record fields required for cryptographically anchored SUT-level certification.

---

## Status of This Document

Draft. Subject to change before v1.0.

---

## Table of Contents

1. [Abstract](#abstract)
2. [Scope](#2-scope)
3. [Definitions](#3-definitions)
4. [Component Interface Map](#4-component-interface-map)
5. [Session Control Interface (ACR → ProofTwin)](#5-hv-mcp-session-control-interface-acr--prooftwin)
6. [Client Interface (AgenTwin → ProofTwin)](#6-hv-mcp-client-interface-agentwin--prooftwin)
7. [Proxy Interface (ProofTwin → ASC)](#7-hv-mcp-proxy-interface-prooftwin--asc)
8. [Verdict Interface (ProofTwin → Arena7)](#8-hv-verdict-interface-prooftwin--arena7)
9. [Tool Profile Schema](#9-tool-profile-schema)
10. [Configuration Reference](#10-configuration-reference)
11. [Compliance Requirements](#11-compliance-requirements)
12. [Versioning](#12-versioning)
13. [Prior Art and Timestamps](#13-prior-art-and-timestamps)
14. [Status and Roadmap](#14-status-and-roadmap)

---

## Related Specifications

- [PP-SPEC-001](https://github.com/proofprotocol/DECLARATION) — Proof Protocol Declaration
- [PP-SPEC-006](https://github.com/proofprotocol/pes-spec) — Proof Efficacy Score
- [PP-SPEC-015](https://github.com/proofprotocol/prooftwin-spec) — ProofTwin Behavioral Attestation Layer

---

## Authors

Nebulonium, Inc. / HACKERverse®  
Craig Ellrod, Founder & CEO  

---

*© 2026 Nebulonium, Inc. Licensed under CC BY 4.0. ProofTwin, ProofStamp, ProofRegister, HACKERverse, and Proof Economy are trademarks of Nebulonium, Inc.*
