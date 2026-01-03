# cyberark-conjur-lab
IAM implementation using CyberArk Conjur, Docker, and Policy-as-Code.
# Enterprise Secrets Management Lab (CyberArk Conjur)

## 🚩 Project Objective
To eliminate "Secret Sprawl" and hard-coded credentials in DevOps pipelines by architecting a centralized Secrets Management solution. This project simulates an enterprise Identity Access Management (IAM) environment using **CyberArk Conjur Open Source**.

## 🏗️ Architecture & Tech Stack
* **Core Engine:** CyberArk Conjur (Open Source v1.x)
* **Persistence Layer:** PostgreSQL 14 (Upgraded from v10 to resolve syntax incompatibility)
* **Orchestration:** Docker Compose (Containerized Microservices)
* **Protocol:** HTTP/JSON (REST API)
* **Client Interface:** Conjur CLI (Ruby-based)

## 🔐 Key Security Controls Implemented
| Control | Implementation Detail |
| :--- | :--- |
| **Least Privilege** | Defined declarative policies (`policy.yml`) to restrict variable access to specific roles. |
| **Network Segmentation** | Isolated the database tier; only the Conjur Server can communicate with Postgres. |
| **Secure Storage** | Migrated credentials (`db/password`) from plain-text config files into the encrypted Vault. |
| **Audit Trail** | All access requests (Login, Retrieve Secret) are logged by the Conjur engine. |

## 🛠️ Implementation Challenges & Troubleshooting
During deployment, I encountered and resolved three critical engineering hurdles:

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

## 📄 Policy-as-Code Example
The following YAML policy was deployed to define the variable structure:

```yaml
- !variable db/password
