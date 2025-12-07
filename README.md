# Green Paper: Standardizing Autonomous Economic Integration

**Title:** The Verifiable Integration Protocol (VIP) Framework  
**Author:** AminCad (https://x.com/AminCad)  
**Status:** Request for Comment (RFC)  
**Category:** Interface / Standard  
**Created:** December 2025  

## 1\. Abstract

This paper proposes a standardized framework for on-chain economic integration, composed of two primitive layers: the **Autonomous Economic Actor (AEA) Schema** and the **AttestationRouter**.

Currently, economic integration between protocols requires bespoke off-chain due diligence and custom on-chain adapters. There is no standard language for an actor (DAO, Protocol, Bridge) to declare its boundaries, dependencies, and control surfaces, nor is there a standard mechanism to verify that these declarations meet a counterparty's risk requirements.

We propose **AEA-0.1**, a machine-readable JSON schema for self-description, and **IAttestationRouter**, a smart contract interface that aggregates modular proofs (ZK, Human, IoT) against a verification specification. Together, these standards enable "programmable due diligence," allowing protocols to automate the risk assessment and onboarding of external assets and systems.

## 2\. Motivation

Ethereum’s composability allows protocols to interact, but it does not inherently allow them to *reason* about one another.

1.  **The Metadata Gap:** To integrate a new asset or protocol, a team must manually reconstruct the target's governance model, upgrade capability, and dependency graph. This information usually lives in documentation or mental models, not on-chain.
2.  **The Verification Silo:** Verification logic is currently fragmented. Systems like Kleros (subjective), ZK-verifiers (cryptographic), and Chainlink (data) operate in silos. There is no standardized "glue" layer to aggregate these diverse proofs into a single, composable "Approved" signal.
3.  **The Re-verification Problem:** Every integrator performs their own redundant checks on the same assets.

By standardizing the **description layer** (AEA Schema) and the **verification logic layer** (AttestationRouter), we can transition from "trust-based integration" to "verifiable, attribute-based integration."

## 3\. Specification

The framework consists of two complementary standards.

### 3.1 The AEA-0.1 Schema (Data Definition)

The AEA schema is a standard JSON structure allowing any on-chain entity to declare its operational semantics. It is **descriptive**, not predictive. It answers: *What am I? Who controls me? What do I depend on?*

**Schema Structure:**

```json
{
  "actor": {
    "name": "String (e.g., Protocol Name)",
    "class": ["Array of Strings (e.g., lending, bridge)"],
    "chain_id": "Int"
  },
  "boundary": {
    "description": "Defines state sets considered 'internal' to the system",
    "contracts": ["Array of Addresses"]
  },
  "control": {
    "governance_type": "String (e.g., token_weighted, multisig)",
    "timelock_duration": "Int (seconds)",
    "admin_capabilities": ["pause", "upgrade", "mint"]
  },
  "dependencies": [
    {
      "target": "String (ENS or DID)",
      "type": "String (e.g., oracle, settlement, bridge)",
      "exposure_metric": "Float (0.0-1.0)"
    }
  ],
  "invariants": [
    {
      "id": "String (e.g., solvency_check)",
      "status": "String (monitored | audited | proven)",
      "proof_uri": "String (IPFS Link)"
    }
  ]
}
```

### 3.2 The AttestationRouter (Logic Interface)

The AttestationRouter is a smart contract standard designed to sit between the **Ethereum Attestation Service (EAS)** and specialized verification logic. It functions as a logic gate that aggregates granular attestations into a final "Verified" state.

**Core Interfaces (Solidity):**

```solidity
// Interface for the Router
interface IAttestationRouter {
    /**
     * @notice Processes a verification task based on a spec.
     * @param taskID The unique ID of the integration task.
     * @param verificationSpec The ruleset (encoded AEA requirements).
     * @param inputAttestations Array of EAS UIDs acting as evidence.
     * @return success Boolean indicating if the aggregation passed.
     */
    function routeAndVerify(
        bytes32 taskID, 
        bytes calldata verificationSpec, 
        bytes32[] calldata inputAttestations
    ) external returns (bool success);

    /**
     * @notice Emits a final 'Approved' attestation if conditions are met.
     */
    function finalizeTask(bytes32 taskID) external;
}

// Interface for Modular Handlers (e.g., Kleros Adapter, ZK Verifier)
interface IVerificationHandler {
    /**
     * @notice Verifies a specific slice of the spec.
     * @param data The data fragment to verify.
     * @return verified Boolean result.
     */
    function verify(bytes calldata data) external view returns (bool verified);
}
```

## 4\. Implementation Reference

To illustrate utility, we describe a standard **Market Onboarding Flow** using this architecture.

**Scenario:** A Lending Protocol listing a new collateral asset.

1.  **Publication:** The Asset Issuer publishes their **AEA-0.1 Manifest** to IPFS, declaring their dependencies (e.g., Chainlink) and control model (e.g., 48h Timelock).
2.  **Specification:** The Lending Protocol’s governance submits a **verificationSpec** to the AttestationRouter requiring:
      * *Subjective Check:* 2/3 Analyst Signatures validating the AEA manifest.
      * *Objective Check:* A ZK-proof confirming historical volatility \< 5%.
      * *Data Check:* A generic attestation from a Decentralized Oracle Network confirming price feed uptime.
3.  **Aggregation:** The **AttestationRouter** receives the individual attestations. It calls `IVerificationHandler` for each type.
4.  **Finalization:** Upon success, the Router mints a single `MarketApproved` attestation via EAS. The Lending Protocol’s `PoolConfigurator` contract reads this single bit to enable the asset.

## 5\. Rationale

**Why separate Schema and Logic?**
By decoupling the definition of the actor (AEA) from the verification of the actor (Router), we allow risk models to evolve without requiring actors to update their self-definitions. A single AEA manifest can be evaluated by conservative, moderate, or aggressive Routers differently.

**Why not a full simulation model?**
Critics may argue AEA-0.1 is an oversimplification of complex economic dynamics. This is intentional. AEA is a **data-definition layer**, not a simulation engine. It standardizes the *inputs* required for simulation, preventing the "Garbage In, Garbage Out" problem, but leaves behavioral prediction to higher-order tools.

**Why middleware?**
Directly embedding verification logic into every dApp creates vendor lock-in and redundant code. A Router architecture allows dApps to swap verification providers (e.g., switching from Kleros to UMA) by changing a handler address, without rewriting core protocol logic.

## 6\. Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
