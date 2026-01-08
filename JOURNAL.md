# 🛠️ Engineering Journal: CyberArk Conjur & Wazuh Integration

This journal documents the technical execution, troubleshooting, and validation steps taken during the deployment of the Enterprise Secrets Management Lab.

## 📓 Phase 1: Infrastructure Deployment & Database Remediation
**Objective:** Deploy a stable PostgreSQL 14 backend for CyberArk Conjur.

* **The Challenge:** Initial deployment failed with `PostgreSQL syntax errors` due to version mismatch between the Conjur image and the DB schema.
* **The Fix:** 1. Manually purged the corrupted Docker volumes.
    2. Modified `docker-compose.yml` to force `postgres:14-alpine`.
    3. Verified connectivity using `docker exec -it <db_container> psql -U conjur`.
* **Result:** Achieved a stable persistent data layer for secrets storage (NIST SC-28).

## 📓 Phase 2: CLI & Network Protocol Alignment
**Objective:** Establish secure communication between the Conjur CLI and Server.

* **The Challenge:** Encountered `Connection Refused: 443` when initializing the CLI.
* **Diagnosis:** Analysis via `netstat -tulpn` revealed the containerized application was binding to Port 80, while the client defaulted to HTTPS.
* **The Fix:** ```bash
    # Re-initializing the client to use the correct HTTP protocol for the lab environment
    conjur init -u http://conjur-server -a my-org
    ```
* **Result:** Successful trust chain initialization without protocol overhead for the dev environment.

## 📓 Phase 3: SIEM Ingestion & Custom Rule Engineering
**Objective:** Ingest and parse Conjur audit logs into Wazuh.

* **The Challenge:** Wazuh was failing to parse Conjur authentication logs because Ubuntu 24.04’s `journald` was prepending non-standard headers.
* **The Fix (The "Trojan Horse" Strategy):**
    1. Modified `/var/ossec/etc/rules/local_rules.xml` to include a custom decoder.
    2. Hooked into the standard SSHD alert tag to force the analysis engine to evaluate the payload.
    3. Used `chown wazuh:wazuh local_rules.xml` to fix permission conflicts that were causing the Wazuh Manager boot loop.
* **Verification:**
    ```bash
    # Testing the rule logic against raw log samples
    /var/ossec/bin/wazuh-logtest
    ```
* **Result:** Successfully captured 5 unauthorized access hits for `conjur_admin` (NIST AU-2, SI-4).

## 📓 Phase 4: Control IA-2 Validation (Machine Identity)
**Objective:** Prove non-human identity (NHI) authentication via Host Factory.

* **Command:**
    ```bash
    # Generating a unique machine token for an application host
    conjur hostfactory tokens create --host-id app-host-01
    ```
* **Result:** Validated that secret retrieval requires a unique machine-level token rather than static passwords, satisfying Least Privilege requirements.
