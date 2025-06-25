# Secure Service Mesh-Enabled Database Platform

This project sets up a secure and automated PostgreSQL database platform on Kubernetes. It uses **Istio** for strong security, **Role-Based Access Control (RBAC)** for permissions, and **Kubernetes CronJobs** for automated tasks like data backups and insertions.

---

## What It Does

This platform is designed to:

* Host a **PostgreSQL database**.
* Allow **privileged access** from specific pods (like CLI tools).
* Use **Istio** to secure all communication with the database via **mutual TLS (mTLS)** and **Authorization Policies**.
* **Automate data management** with scheduled data insertion and backups.
* Ensure all relevant services use **Istio sidecars** for mesh capabilities.

---

## Key Features

* **Secure Database:** PostgreSQL deployment with persistent storage.
* **Automated Backups:** A CronJob regularly backs up the database.
* **Automated Data Insertion:** A CronJob periodically adds data to the database.
* **Database Setup:** An initial job creates the necessary database schema.
* **Strict Access Control:**
    * **Kubernetes RBAC** ties permissions to a `user-dev` Service Account.
    * **Istio AuthorizationPolicy** only allows the `user-dev` Service Account to connect to the database.
    * **Istio mTLS** encrypts all communication within the database namespace.
* **Dedicated CLI Access:** A `user-cli` pod for manual database interaction.

---

## Architecture Overview

The platform is deployed within a dedicated Kubernetes Namespace called `secure-db-platform`, which has **Istio sidecar injection enabled**. This means all pods within this namespace automatically get an Istio proxy, allowing them to be part of the service mesh.

Here's how the main components interact:

* **PostgreSQL Database:** This is deployed as a Kubernetes `Deployment` (`service-mesh-database`) and exposes its service via a `Service` (`service-mesh-database`). It uses a `PersistentVolumeClaim` (`postgres-pvc`) for durable storage and retrieves sensitive credentials from a Kubernetes `Secret` (`postgres-secret`).

* **Service Account (`user-dev`):** A central `ServiceAccount` named `user-dev` is created. This service account is the identity for all privileged operations.

* **RBAC Permissions:** A Kubernetes `Role` (`db-access`) defines permissions (like access to pods and their execution capabilities) which are then granted to the `user-dev` Service Account via a `RoleBinding` (`db-access-binding`).

* **Automated Jobs & CLI Access:**
    * The **`data-backup` CronJob** (for database backups)
    * The **`data-inserter` CronJob** (for inserting test data)
    * The **`db-init` Job** (for initial database schema creation)
    * The **`user-cli` Pod** (for manual command-line access)
    All these components run using the `user-dev` Service Account. They connect to the PostgreSQL database via the `service-mesh-database` Kubernetes Service.

* **Istio Security Policies:**
    * **`PeerAuthentication` (`default`):** Enforces strict mutual TLS (mTLS) for all communication between services within the `secure-db-platform` namespace. This encrypts and authenticates all internal traffic.
    * **`AuthorizationPolicy` (`allow-cli-to-db`):** This policy specifically targets the PostgreSQL database `Service` (`service-mesh-database`). It is configured to **only allow connections from clients that present the `user-dev` Service Account principal** (`cluster.local/ns/secure-db-platform/sa/user-dev`). This ensures that only authorized entities (like our automated jobs and CLI pod) can access the database, even within the mesh.

In essence, all interaction with the PostgreSQL database, whether from automated tasks or the CLI pod, is funneled through the Istio mesh, secured by mTLS, and strictly controlled by the Authorization Policy based on the `user-dev` Service Account.

---

## Getting Started

### Prerequisites

You'll need:

* A **Kubernetes cluster**.
* **`kubectl`** configured for your cluster.
* **Istio** control plane installed on your cluster.
* **`helm`**.

### Deployment

This project is best deployed using **Helm**.

1.  **Ensure you have a `values.yaml`** file that defines the configurations like namespace, image, schedules, etc. (The content you provided already includes these values).

2.  **Make sure all your Kubernetes YAML files** are in your Helm chart's `templates/` directory.

3.  **Deploy the Helm chart** to your cluster:

    ```bash
    helm upgrade --install secure-db-platform . --namespace secure-db-platform --create-namespace
    ```

    This command will:
    * Create the `secure-db-platform` namespace.
    * Enable Istio sidecar injection for the namespace.
    * Deploy all the platform components.

---

## Core Components Explained

* **PostgreSQL Database (`Deployment`):** The main database container. Uses a `PersistentVolumeClaim` for data and a `Secret` for credentials.
* **Automated CronJobs:**
    * `data-backup`: Dumps the database regularly (e.g., daily).
    * `data-inserter`: Inserts test data periodically.
* **Database Initialization (`Job`):** `db-init` sets up the initial `test` table in the database.
* **Service Account (`user-dev`) & RBAC (`Role`/`RoleBinding`):** Defines a specific identity (`user-dev`) for our automated tasks and CLI pod, and grants it the necessary permissions to operate within Kubernetes and connect to the database.
* **CLI Access Pod (`user-cli`):** A pod you can use to manually connect to the database and run commands.
* **Istio Policies:**
    * **`PeerAuthentication` (mTLS):** Forces all communication within the namespace to be encrypted and mutually authenticated by Istio.
    * **`AuthorizationPolicy`:** Specifies that **only** the `user-dev` Service Account can connect to the database service, providing strong access control at the network level.

---

## How to Access the Database (via CLI Pod)

1.  **Get your CLI pod name:**

    ```bash
    kubectl get pods -n secure-db-platform -l app=user-cli
    ```

2.  **Access the pod:**

    ```bash
    kubectl exec -it <user-cli-pod-name> -n secure-db-platform -- bash
    ```

3.  **Connect to PostgreSQL:**

    ```bash
    # These commands fetch credentials from the secret
    PGUSER=$(kubectl get secret postgres-secret -n secure-db-platform -o jsonpath='{.data.postgres-user}' | base64 -d)
    PGPASSWORD=$(kubectl get secret postgres-secret -n secure-db-platform -o jsonpath='{.data.postgres-password}' | base64 -d)
    PGDATABASE=$(kubectl get secret postgres-secret -n secure-db-platform -o jsonpath='{.data.postgres-db}' | base64 -d)

    psql -h service-mesh-database -U "$PGUSER" -d "$PGDATABASE"
    ```

    You are now in the database. Try `SELECT * FROM test;` to see inserted data.

---

## Troubleshooting Tips

* **Check Pod Status:** Use `kubectl get pods -n secure-db-platform` and `kubectl describe pod <pod-name> -n secure-db-platform`.
* **Review Pod Logs:** `kubectl logs <pod-name> -n secure-db-platform`.
* **Verify Istio Injection:** Ensure your pods show `2/2` in the `READY` column when you `kubectl get pods`, indicating the Istio sidecar is running.
* **CronJob Activity:** Check `kubectl get cronjobs -n secure-db-platform` and `kubectl get jobs -n secure-db-platform` to see if they're running as expected.

---

## Need to customize anything?

You can easily adjust settings like the PostgreSQL version, backup schedules, or resource limits by modifying your Helm `values.yaml` file and other related files before deployment.