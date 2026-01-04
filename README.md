# cyberark-conjur-lab
IAM implementation using CyberArk Conjur, Docker, and Policy-as-Code.

# Enterprise Secrets Management & SIEM Integration Lab

## 🚩 Project Objective
To eliminate "Secret Sprawl" in DevOps pipelines by architecting a centralized Secrets Management solution, and to validate security controls through a real-time **SIEM (Security Information and Event Management)** pipeline. This project simulates an enterprise Identity Access Management (IAM) environment using **CyberArk Conjur** and **Wazuh**.

## 🏗️ Architecture & Tech Stack
* **Core Engine:** CyberArk Conjur (Open Source v1.x)
* **SIEM / Auditing:** Wazuh (Manager & Agent v4.x)
* **Persistence Layer:** PostgreSQL 14 (Upgraded from v10)
* **Orchestration:** Docker Compose (Containerized Microservices)
* **Host OS:** Ubuntu 24.04 LTS (Proxmox VM)
* **Protocol:** HTTP/JSON (REST API) & Syslog/Journald
* **Client Interface:** Conjur CLI (Ruby-based)

## 🔐 Key Security Controls Implemented
| Control | Implementation Detail |
| :--- | :--- |
| **Least Privilege** | Defined declarative policies (`policy.yml`) to restrict variable access to specific roles. |
| **Network Segmentation** | Isolated the database tier; only the Conjur Server can communicate with Postgres. |
| **Secure Storage** | Migrated credentials (`db/password`) from plain-text config files into the encrypted Vault. |
| **Audit Trail (NIST AU-6)** | Integrated Wazuh SIEM to capture and visualize privileged access attempts (e.g., `conjur_admin` login failures). |

## 🛠️ Implementation Challenges & Troubleshooting
During deployment, I encountered and resolved five critical engineering hurdles:

### 1. Database Compatibility Crash
* **Issue:** The Conjur container failed to start with a `syntax error` in the Postgres logs.
* **Diagnosis:** Identified that the legacy Conjur image was incompatible with older Postgres default schemas.
* **Resolution:** Upgraded the database service to `postgres:14` and forced a schema rebuild.

### 2. Container "Crash Loop"
* **Issue:** The Client CLI container would exit immediately upon startup (`Exit Code 0`).
* **Resolution:** Rewrote the `docker-compose.yml` entrypoint to use a "Keep-Alive" strategy:
    ```yaml
    entrypoint: ["tail", "-f", "/dev/null"]
    ```
    This forced the container to remain active, allowing for interactive shell sessions.

### 3. Port & Protocol Mismatch
* **Issue:** The CLI connection was refused (`Connection Refused: 443`).
* **Root Cause:** `netstat` analysis revealed the Application Server was binding to **Port 80 (HTTP)**, while the Client defaulted to **HTTPS (443)**.
* **Fix:** Re-initialized the trust chain using the HTTP protocol to align with the Lab environment's network bindings.

### 4. SIEM Engine Boot Loop (Corrupted State)
* **Issue:** The Wazuh Analysis Engine entered a restart loop, failing to load custom rules.
* **Diagnosis:** Identified a race condition where the engine attempted to read corrupted queue files in the Docker volume, combined with a Linux permission conflict (`root` vs `wazuh` user) on the `local_rules.xml` file.
* **Resolution:** Engineered an atomic recovery command to purge volatile queues and corrected file ownership using `chown wazuh:wazuh` to restore the pipeline.

### 5. Log Parsing & Journald Incompatibility
* **Issue:** Ubuntu 24.04's `Journald` was intercepting application logs, causing them to be ignored by standard Syslog decoders.
* **Fix:** Engineered a custom XML decoder rule using a "Trojan Horse" strategy—hooking into standard SSHD alert tags—to guarantee detection of `conjur_admin` events despite the non-standard log format.

## 📄 Policy-as-Code Example
The following YAML policy was deployed to define the variable structure:

NIST Control,Description,Proof of Implementation
AU-6,Audit Review,Centralized Wazuh dashboard aggregating all Conjur auth events.
AC-2,Account Management,Validated detection of invalid/unauthorized users (conjur_admin).
SI-4,System Monitoring,Real-time ingestion of container logs via Wazuh Agent.

<img width="1280" height="800" alt="Conjur_admin" src="https://github.com/user-attachments/assets/631a54e4-20dc-4946-8df1-fa0117e23801" />


## 📄 Policy-as-Code Example
The following YAML policy was deployed to define the variable structure:

```yaml
- !variable db/password
