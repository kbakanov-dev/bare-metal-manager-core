# Managedhost Attestation via SPDM

- Feature Name: Managed Host and DPU Attestation via Redfish/SPDM
- Design Start: 28-Sep-2025
- Design (Pull Request URL: URL to this PR - update after submitting)

- PIC: ~~Abhishek Varshney~~, now Konstantin Bakanov

Contributors:

- Binu Ramakrishnan

Table of Contents

[[_TOC_]]

# Introduction
This document outlines the design for implementing SPDM-based attestation of hosts and DPUs via Redfish. As a critical security measure, Carbide must verify that all managed hosts are authentic, sourced from approved vendors, and free from tampering. To achieve this, Carbide will conduct attestation at multiple stages throughout the host lifecycle. This design details the attestation stages, methods for evidence collection, appraisal policies, verification APIs, and other relevant considerations.

SPDM attestation has to be ready by the time DS9 is shipped, which is July 2026. The following devices need to be supported: Vera CPU, Rubin GPU, BF4 DPU (Optional), CX10 (NIC card in the PCI slot). The hardware to test the devices will become available around March 2026. Before that date, the tests can only be done using the existing GB200 machines in pdx-lab or YTL (pending machines availability and NRAS readiness).

# Actors
The actors are:
- Carbide - generic term is DCMS (Data Center Management Software)
- DC Operator

# Use Cases
```plantuml
:Carbide: -up-> (A. Collect Evidence and Certificate from Device using SPDM)
:Carbide: -left-> (B.Collect FirmwareInventory \nfrom Device using SPDM)
:Carbide: -right-> (C. Submit Evidence to NRAS-Verifier)
:Carbide: -down-> (D. Analyse Attestation Results via Appraisal Policy)

:DC Operator: --> (Setup Appraisal Policy for Attestation Results)
:DC Operator: --> (Handle Failure Scenarios)
```

## Terminology
- Evidence - a checksum obtained via BMC's Redfish service that attests to the integrity of a device.
- NRAS - a verifier service which compares evidence, firmware version etc against predefined values known as RIMs.
- Appraisal policy - a policy set by the DC operator that guides how attestation results should be interpreted.

# Lifecycle
## Setup
- Configure appraisal policy.
- Configure NRAS connection.

## Attestation
```plantuml
Carbide -> "SPDM (Client)" : Get FirmwareInventory (to obtain firmware versions)
"SPDM (Client)" -> Device : Get FirmwareInventory
Device -> "SPDM (Client)" : FirmwareInventory
"SPDM (Client)" -> Carbide : FirmwareInventory

Carbide -> "SPDM (Client)" : Collect Metadata (with Certificate) from Device
"SPDM (Client)" -> Device : Get Metadata via Redfish
Device -> "SPDM (Client)" : Metadata
"SPDM (Client)" -> Carbide : Metadata

Carbide -> "SPDM (Client)" : Collect Evidence from Device
"SPDM (Client)" -> Device : Get Evidence via Redfish
Device -> "SPDM (Client)" : Evidence
"SPDM (Client)" -> Carbide : Evidence


Carbide -> "NRAS (Client)" : Submit Evidence, Certificate and Firmware Version for verification
"NRAS (Client)" -> NRAS : Verify Evidence \n against its own policy
NRAS -> "NRAS (Client)" : Attestation Results
"NRAS (Client)" -> Carbide : Attestation Results

note left
  Starting to Apply Appraisal Policy
end note

alt Attestation Successful
  Carbide -> Carbide : Update machine's state
else Attestation Failed
  Carbide -> Carbide: Add health override and move machine to Failed state
else Firmware Version Incorrect
  Carbide -> "SPDM (Client)" :  Do firmware Update
  alt Update Failed
    Carbide -> Carbide : Move machine to Failed state
  else Update Succeeded
    Carbide -> Carbide : Update machine's state
  end
end
```
### Case: Attestation Not Supported by Verifier
If NRAS does not support attestation of any target/component as requested by carbide, carbide will assume it success and will proceed with next handling.

In general, if a machine type, e.g. DELL or Lenovo, are not supported in principle, this can be configured via a usual configuration parameter, where the allowed machines types are listed together. E.g. vmass_attestation_supported_machines="GB200,GB300". However, another solution would be to try and attempt attestation of all machines and all devices, as described above. In situation when we cannot: (1) fetch evidence, or (2) NRAS cannot attest a specific device, we just mark it as success and proceed forward.

### Case: Attestation Failed by Verifier
If attestation fails, the host should remain on the network where the attestation was initiated. For failures occurring during ingestion, the host must stay on the admin network until the issue is resolved or force-deleted. Manual intervention is required in these cases, and the machine should be force-deleted. 

**Target Network vs Attestation Phase**
| Phase            | Target Network|
|------------------|---------------|
|Ingestion         |  Admin        |
|Instance Creation |  Admin        |
|Instance Deletion |  Tenant       |

### Case: Firmware Version Mismatch
NRAS will perform firmware version check. If it fails, the remediation action will be dictated by the appraisal policy.

## Quarantine
When a machine fails attestation, it should be isolated from the rest of the network to prevent potential spread of compromise. Currently, Carbide does not implement this network isolation feature.

# Carbide

## Attestation State Machine
Each attestation target on every machine is treated as an independent entity. This means that all targets, across all machines, are processed concurrently without regard to machine boundaries.
```plantuml
hide empty description
state FetchMeasurements #pink {
  state TriggerMeasurements {
  }
  state PollMeasurements {
  }
  state FetchMetaData {
  }
  state m <<choice>>
  [*] -> FetchMetaData
  FetchMetaData -> m
  m -> FetchCertificate: Metadata is collected.
  m -> Success: Metadata collection is not supported.
  m -> Failed: Metadata collection is supported but collection failed.
  FetchCertificate -> TriggerMeasurements: Certificate is fetched successfully
  TriggerMeasurements: Request all available measurements\n for the target 
  TriggerMeasurements -> PollMeasurements: Start Polling
  PollMeasurements -> PollMeasurements: Until task is finished
  PollMeasurements --> Verification: Task is completed and measurements are received
  PollMeasurements -> Failed: Task is failed
}

state AttestationCompleted #skyblue {
  state Success {
  }
  state Failed {
  }
}
 
state Verification #DarkSeaGreen {
    state VerifyWithVerifier {
    }
  state c <<choice>>
  [*] -> VerifyWithVerifier: Send data to verifier\n and wait for result
  VerifyWithVerifier --> Failed: Verifier returns failure
  VerifyWithVerifier --> c: Verifier returns success
  c --> FetchFirmwareVersion : [Component firmware version policy is set]
  c --> Success: [Component firmware version policy is NOT set]
}

state ApplyEvidenceAppraisalPolicy {
  state ApplyAppraisalPolicy {
  }
  ApplyAppraisalPolicy --> Success: Appraisal policy validation passed
  ApplyAppraisalPolicy --> Failed: Appraisal policy validation failed
}

[*] --> CheckIfAttestationSupported
CheckIfAttestationSupported: Check if `redfish/v` url contains endpoint ComponentIntegrity?
state x <<choice>>
CheckIfAttestationSupported --> x
x --> FetchAttestationTargets: [ComponentIntegrity endpoint exists]
x --> Success: [ComponentIntegrity endpoint does not exist]
FetchAttestationTargets : Query redfish ComponentIntegrity url\n and fetch all available SPDM targets\nUpdate db with supported components\nand updated nonce as well.
state a <<choice>>
state b <<choice>>
FetchAttestationTargets --> a: Targets are known now
a -> FetchMeasurements: [First time attestation is triggered]
a -> ValidateExistingTargetList: [Target list already exists]
ValidateExistingTargetList -> b
b -up-> FetchMeasurements: [Existing target list matched]
b -> Failed: [Existing target list NOT matched]
AttestationCompleted -up-> FetchAttestationTargets: On attestation request
```

## ManagedHost State Machine Updates
The managed host state machine must initiate attestation at key stages throughout the managed host's lifecycle.

- During Ingestion
    - During DPU ingestion
    - During Host ingestion
- During Instance Allocation
- During Instance Deletion
    When deleting an instance, the host may still be running an older firmware version. In this scenario, validation must be performed against the firmware version that was active at the time the instance was created. To this effect, the appraisal policy will contain multiple firmware versions per device, each marked as `Active` or `Deprecated`. If an instance is running an outdated firmware version, marked as `Deprecated`, attestation should succeed, but the machine must be placed in a maintenance state until the firmware is updated.

Attestation should be executed on the same network context in which the triggering event occurs. For example:
- When attestation is initiated during instance creation, it must be performed on the admin network.
- When attestation is initiated during instance deletion, it must be performed on the tenant network.

```plantuml
state a <<choice>>
state b <<choice>>
[*] --> ManagedHostState #DarkSeaGreen 
ManagedHostState: HostInit/Measuring, \nReady (Measuring),\n Preassigned Measuring, \n PostAssignedMeasuring
ManagedHostState --> a
a -> [*]: [Attestation is NOT supported]
a -> RaiseAttestationRequest #skyblue: [Attestation is supported] 
RaiseAttestationRequest: Update db with request parameter.\nAttestation state machine is monitoring\ndb for the requests.
RaiseAttestationRequest --> WaitingForAttestationCompletion #pink: Request is sent to Attestation State Machine
WaitingForAttestationCompletion -> WaitingForAttestationCompletion: Attestation result is not yet updated
WaitingForAttestationCompletion --> b: Attestation is completed
b --> [*]: [Attestation is successful]
b -> Failed #grey: [Attestation is failed]
Failed: Move to failed state\n as per parent state
state b <<choice>>
```

## Data Model
For every attestation request, create a record in the `spdm_machine_attestation` table for a machine_id, if the record already exists, update the time in the field `requested_at`. The managed host should periodically check the attestation status until it transitions to `completed`.

Each attestation request will have an expiration time (e.g., 60 minutes). If the attestation is not completed within this period (result marked as expired), the state machine should attempt to restart the attestation process. After two retries, if the attestation still fails to complete, it will be marked as failed and an appropriate error message will be recorded.

**table**: `spdm_machine_attestation`

**fields**:
```sql
    machine_id VARCHAR UNIQUE NOT NULL,
    requested_at TIMESTAMPTZ NOT NULL,
    started_at TIMESTAMPTZ,
    canceled_at TIMESTAMPTZ,
    state JSONB NOT NULL,
    -- If state version time is less than requested_time, indicates that a new request is received.
    -- State machine should reset the state and start over
    state_version VARCHAR NOT NULL,
    state_outcome JSONB,
    attestation_status spdm_attestation_status_t NOT NULL DEFAULT 'not_started',

    CONSTRAINT fk_machine_id
        FOREIGN KEY (machine_id)
        REFERENCES machines(id)
        ON DELETE CASCADE  -- force-delete handling.
        ON UPDATE CASCADE  -- Update predicted host id to permanent host id scenario.
```

Each machine contains multiple devices e.g. CPU, GPU, CX7, BMCs for attestation. Following table will store information for all these devices.

**table**: `spdm_machine_devices_attestation`

**fields**:

```sql
    machine_id VARCHAR NOT NULL,
    device_id VARCHAR NOT NULL,
    nonce UUID NOT NULL DEFAULT gen_random_uuid(),
    state JSONB,
    -- last_known_metadata will be swapped with current_metadata after appraisal policy validation.
    last_known_metadata JSONB,
    current_metadata JSONB,
    ca_certificate_link VARCHAR,
    -- matches with the struct CaCertificate from libredfish
    ca_certificate JSONB,
    evidence_target VARCHAR,
    -- matches with the struct Evidence from libredfish.
    evidence JSONB,
    PRIMARY KEY (machine_id, device_id),
    CONSTRAINT fk_machine_id
        FOREIGN KEY (machine_id)
        REFERENCES machines(id)
        ON DELETE CASCADE  -- force-delete handling.
        ON UPDATE CASCADE  -- Update predicted host id to permanent host id scenario.
```

This table records each state transition in the attestation process. The `snapshot` field captures the machine's state at the time of the transition, including the `machine_id`, current state, and a map of all devices with their corresponding certificates, evidences, and states.

**table**: `spdm_machine_attestation_history`

**fields**:

```sql
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    machine_id VARCHAR NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL,
    snapshot JSONB NOT NULL,  -- This field contains a hashmap for all components, evidences and certificates, and machine information.
    state_version VARCHAR NOT NULL,
    attestation_status spdm_attestation_status_t NOT NULL
```

## Nonce Generation
Carbide should generate the nonce which is cryptographically safe. As per the Rust UUID crate:
> Specifically, when you use Uuid::new_v4(), the uuid crate internally relies on the getrandom crate. The getrandom crate, in turn, utilizes the underlying operating system's mechanisms for generating random numbers, which are typically cryptographically secure. This ensures that the generated UUIDs are sufficiently random and unpredictable for security-sensitive applications.

Therefore, using the Rust `uuid` crate (specifically `Uuid::new_v4()`) is a secure and appropriate method for generating nonces.

# Devices
Here is the list of devices we aim to support.

## GPU
GPU attestation is performed via host BMC. *Provide some Redfish info.*

NRAS endpoint for attestation is ... Documentation can be found ...

*Some relevant notes.*

## CPU
*TBD*
## CX10
*TBD*
## DPU
DPU supports attestation for following targets:
- Bluefield_DPU_IRoT – The BlueField IRoT (Initial Root of Trust), a Platform Security Controller (PSC) that stores measurements related to the Arm and NIC components
- Bluefield_ERoT – The BlueField BMC ERoT (Endpoint Root of Trust), which contains measurements related to the DPU BMC

DPU attestation should be performed from DPU BMC.

# External Interactions

## Evidence
### Measurement Collection
Below is an example of how to collect GPU measurements from the host BMC.

Trigger the measurement collection on GPU.
```json
POST https://<HOST_BMC_IP>/redfish/v1/ComponentIntegrity/HGX_IRoT_GPU_0/Actions/ComponentIntegrity.SPDMGetSignedMeasurements 
DATA '{"Nonce": "d42a0594c5cd5743ee08fe5ec3cf884b1fac4f106879cda98b7d1c51652b04b7"}'
{
  "@odata.id": "/redfish/v1/TaskService/Tasks/HGX_2",
  "@odata.type": "#Task.v1_4_3.Task",
  "Id": "HGX_2",
  "TaskState": "Running",
  "TaskStatus": "OK"
}
```

Wait until Task is completed.
```json
GET https://<HOST_BMC_IP>/redfish/v1/TaskService/Tasks/HGX_2
{
  "PercentComplete": 100,
  "StartTime": "2025-10-14T04:47:49+00:00",
  "TaskMonitor": "/redfish/v1/TaskService/Tasks/HGX_2/Monitor",
  "TaskState": "Completed",
  "TaskStatus": "OK"
}
```

Collect measurements
```json
GET https://<HOST_BMC_IP>/redfish/v1/ComponentIntegrity/HGX_IRoT_GPU_0/Actions/ComponentIntegrity.SPDMGetSignedMeasurements/data
{
  "HashingAlgorithm": "TPM_ALG_SHA_384",
  "SignedMeasurements": "EeAB//onuMjAhG+xkAnEGtpXXXXXXXXXXXXXXXXXXXXXXXXXXXXX0MgcAAQEHAIMEAAAAAAECAQQAggEA/wMBMwACMAASAMPLEDATA2T4Abi8C7NwQWnzJMXw4r02kpQnV6MSw==",
  "SigningAlgorithm": "TPM_ALG_ECDSA_ECC_NIST_P384",
  "Version": "1.1.0"
}
```

Evidences are stored in `spdm_machine_devices_attestation` table.

### Certificate Collection

The certificate can be retrieved using the URL provided in the following structure.

```json
"SPDM": {
    "IdentityAuthentication": {
      "ResponderAuthentication": {
        "ComponentCertificate": {
          "@odata.id": "/redfish/v1/Chassis/HGX_IRoT_GPU_0/Certificates/CertChain"
        }
      }
    },
    "Requester": {
      "@odata.id": "/redfish/v1/Managers/HGX_BMC_0"
    }
  }
```

To retrieve the certificate, perform a GET request to the certificate's URI.
```json
GET https://10.217.182.134/redfish/v1/Chassis/HGX_IRoT_GPU_0/Certificates/CertChain
{
  "@odata.id": "/redfish/v1/Chassis/HGX_IRoT_GPU_0/Certificates/CertChain",
  "@odata.type": "#Certificate.v1_5_0.Certificate",
  "CertificateString": "-----BEGIN CERTIFICATE-----\nMIIDdDCCAvqgAwIBAgIUdgzUdmT3058TdKflDS6w/mMWggcwCgYIKoZIzj0EAwMw\nZDEbMBkGA1UEBRMSNDBDNUJBMzc4QUZERURBNEU2MQswCQYDVQQGEwJVUzEbMBkG\nA1UECgwSTlZJRElBIENvcnBvcmF0aW9uMRswGQYDVQQDDBJHQjEwMCBBMDEgRlNQ\nIEJST00wIBcNMjMwNjIwMDAwMDAwWhgPOTk5OTEyMz********************WYoDBvs2wPAH1Gw4LbcwUN8QCff8bFmV4ZxjCRr4WXTLFHBKjbfneGSBWwA=\n-----END CERTIFICATE-----\n",
  "CertificateType": "PEMchain",
  "CertificateUsageTypes": [
    "Device"
  ],
  "Id": "CertChain",
  "Name": "HGX_IRoT_GPU_0 Certificate Chain",
  "SPDM": {
    "SlotId": 0
  }
}
```

## Machines/Devices Integrity
Currently, SKU already provides machine integrity check for DPU, GPU and CPU. CX10 (PCIe NIC) is in the plans. That SKU integrity check makes sure that a machine contains a certain list of devices, and that the list remains unchanged. It allows to flag incorrectly configured machines, or missing components, which can be important during the initial site rollout. The check is performed during machine ingestion, after a firmware update and after a machine has been released by a customer. The list of devices is compiled from the topology information, site explorer information, infiniband capabilities information.

The issue with this approach is that topologies information, for example, is supplied by scout agent running on our internal OS, which is not cryptographically secure. `site-explorer` information, however, is obtained via BMC.

As a result, a more cryptographically robust system needs to be developed, which is based upon obtaining information directly from BMC. That solution would essentially replace and supersede the existing SKU. It would provide the following functionality: detection of bad devices and devices infected with malware, detection of unexpected firmwares and compromised versions of firmwares, verification of the supply chain integrity (i.e. that the devices we receive have been manufactured by the original manufacturer - this depends on the necessary information supplied by the manufacturer) as well as machine component integrity as a whole.

Unfortunately, such project cannot be completed by the DS9 deadline without attracting additional development resources. Therefore for the purpose of M0, we'll rely on existing SKU functionality to do machine integrity check. SPDM attestation, in turn, will be responsible for making sure that a device is a genuine one and that it runs an allowed (from NRAS standpoint) version of firmware.

Replacing existing SKU with BMC based version of machine integrity check is a project in its own right, and will need to be executed as such. In an ideal scenario, all the SKU and its component measurements shall me made available to Carbide out-of-band during site bringup/machine injestion phase. Unfortunately, not all SKU's and its firmware component versions and measurements are not available out of band. So as a stop gap, we shall rely on Trust-on-first-use (TOFU) scheme creating a baseline by taking a snapshot of devices, versions and its measurements via BMC during machine introduction into a site (or machine ingestion). Subsequent periodic snapshots (attestation) are taken throughout machine lifecycle and compared them against previous snapshots to detect drifts. This would give us possibility to log any changes performed to a machine SKU. In future, we would be able to rely on using existing (or extended with e.g. fw version info) SKU configuration provided in `deployments` repo to compare our snapshots against, and thus supplant the current SKU as a whole.

## Component Firmware Versions and other Metadata Collection
SKU will perform firmware version integrity check, i.e. that only allowed firmware versions are deployed on a site. NRAS will perform another check: to make sure that a firmware version is valid in general (i.e. wider scope than SKU). Below sections describe firmware version collection for the purpose of NRAS check.

FirmwareInventory represents firmware version information for each component on the machine. Carbide can use this service to fetch the firmware version from the host. Please see [Attestation](#attestation) Section for the corresponding sequence diagram.

In practice, the device identifiers used within the ComponentIntegrity collection often do not directly correspond to the identifiers found in the FirmwareInventory. Therefore, during attestation, Carbide must be able to determine the correct firmware version for each device under evaluation. The relationship between ComponentIntegrity entries and their associated FirmwareInventory records is vendor-specific and requires a mapping. 

Example of FirmwareInventory:
```json
{
    "@odata.id": "/redfish/v1/UpdateService/FirmwareInventory",
    "@odata.type": "#SoftwareInventoryCollection.SoftwareInventoryCollection",
    "Members": [
        {
            "@odata.id": "/redfish/v1/UpdateService/FirmwareInventory/CX7_0"
        },
        {
            "@odata.id": "/redfish/v1/UpdateService/FirmwareInventory/FW_BMC_0"
        },
        {
            "@odata.id": "/redfish/v1/UpdateService/FirmwareInventory/FW_CPLD_0"
        },
        {
            "@odata.id": "/redfish/v1/UpdateService/FirmwareInventory/FW_ERoT_BMC_0"
        },
        {
            "@odata.id": "/redfish/v1/UpdateService/FirmwareInventory/HGX_FW_BMC_0"
        },
        {
            "@odata.id": "/redfish/v1/UpdateService/FirmwareInventory/HGX_FW_CPU_0"
        },
        {
            "@odata.id": "/redfish/v1/UpdateService/FirmwareInventory/HGX_FW_CPU_1"
        },
        {
            "@odata.id": "/redfish/v1/UpdateService/FirmwareInventory/HGX_FW_ERoT_BMC_0"
        },
        {
            "@odata.id": "/redfish/v1/UpdateService/FirmwareInventory/HGX_FW_ERoT_CPU_0"
        },
        {
            "@odata.id": "/redfish/v1/UpdateService/FirmwareInventory/HGX_FW_ERoT_FPGA_0"
        },
        {
            "@odata.id": "/redfish/v1/UpdateService/FirmwareInventory/HGX_FW_FPGA_0"
        },
        {
            "@odata.id": "/redfish/v1/UpdateService/FirmwareInventory/HGX_FW_GPU_0"
        },
        {
            "@odata.id": "/redfish/v1/UpdateService/FirmwareInventory/HGX_InfoROM_GPU_0"
        },
        {
            "@odata.id": "/redfish/v1/UpdateService/FirmwareInventory/HGX_PCIeSwitchConfig_0"
        }
    ],
    "Members@odata.count": 34,
    "Name": "Software Inventory Collection"
}
```

For example, the table below demonstrates the required mapping for GB200 devices.

| FirmwareInventory  | ComponentIntegrity |
|--------------------|--------------------|
| CX7_3              | IRoT_CX7_3         |
| HGX_FW_GPU_1       | HGX_IRoT_GPU_1     |
| HGX_FW_ERoT_CPU_0  | HGX_ERoT_CPU_0     |
| HGX_FW_ERoT_FPGA_0 | HGX_ERoT_FPGA_0    |
| HGX_FW_FPGA_0      | HGX_ERoT_FPGA_0    |

For each device, Carbide must refer to the vendor-specific mapping table to determine the correct FirmwareInventory endpoint corresponding to the ComponentIntegrity entry. Using this endpoint, Carbide can retrieve the firmware version required for attestation.

Certain components may require additional metadata fields in order to be attested using NRAS. The specific metadata requirements for these components are still to be determined and will be provided once available.

## NRAS-Verifier (Client)
Currently, NVIDIA attestation team offers two solutions: the fully remote attestation service, named NRAS, and a semi-remote attestation service, distributed as an SDK.

NRAS documentation is available here - https://docs.attestation.nvidia.com/api-docs/nras.html. It offers attestation for evidence obtained via in-band methods (e.g. via Scout). It is made out of three components: Verifier, RIM Service and a Revocation Service.

Semi-remote attestation service is made out of a Verifier, which is executed locally, and a remote RIM Service and a remote Revocation Service. The latter two are shared with NRAS. In case when a fully airgapped solution is required, a cache, named Trust Outpost Service, is placed between a Verifier and remote services. That cache synchronizes with the remote RIM Service and the remote Revocation Service, and then replaces them in their absence.

Verifier can be executed locally in two modes: as a linked library, and as standalone executable. In the latter scenario, the communication with that standalone executable is performed via CLI only.

For the purpose of M0, it was decided that we'll use the new out of band NRAS. First, the attestation team will provide the new API with stubs, which will be followed by a full out of band NRAS. In subsequent milestones, we might use a semi-remote out of band attestation service.

### NRAS Attestation Results
NRAS expects three pieces of information in every request: (1) nonce, (2) an array of evidence, certificate and firmware version for each device in the request, and (3) architecture type.

The return structure contains attestation result for the entire request as well as a more detailed attestation report for each device specified in the request. Both are encoded as JWT tokens so that the authenticity of the sender could be verified.

The overall attestation result is conveyed using a boolean field `x-nvidia-overall-att-result`. It will be mapped to the `attestation_status` field in the `spdm_machine_attestation` table.

# Appraisal Policy for Attestation Results
*Appraisal policy will be defined as a result of collaboration with NRAS team.*

# Milestones
- M0 - Due by July 2026
  - Carbide fetches Firmware, Evidence and Certificates from GPUs only via host BMC SPDM Redfish
  - Carbide submits data for verification to NRAS and fetches the results
  - Carbide analyses the NRAS response according to Appraisal Policy and transitions a machine into a Failed state or not.
  - Appraisal policy is defined that allows to perform simple actions based on NRAS response: quarantine the managed host, move the ingestion forward, log the result.
  - There is a mechanism for a DC Operator to handle Failed state: e.g. re-do attestation, or transition the machine forward.
  - Devices supported are: Vera CPU, Rubin GPU, BF4 DPU (Optional), CX10 (NIC card in the PCI slot).

- M1
  - More complex appraisal policy, allowing to perform e.g. automatic firmware update

# Limitations
  - We primarily target YTL and GB200 machines. If a machine type is not supporting attestations, we treat it as "Not supported".
  - Currently, carbide does not support network isolation feature. The best it can do is mark a certain managed host as Failed. Manual intervention will then be required to remediate the situation.

# Dependencies
  - We rely on SKU to do component integrity check and firmware version integrity check.
  - We rely on NRAS to do firmware version check.
  - NRAS supporting attestation of Vera CPU, Rubin GPU, BF4 DPU (Optional), CX10 (NIC card in the PCI slot) as a remote service.
  - When an OOB firmware patch/update is required, this will be handled by means outside of the scope of this project.
  - A test setup is available for use.
