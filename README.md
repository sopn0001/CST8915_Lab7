# CST8915 Lab 7: Full-stack Cloud-native Development: Introduction to Kubernetes Basics

**Student Name**: IDRIS JOVIAL SOP NWABO

**Student ID**: 041199877

**Course**: CST8915 Full Stack Cloud Native Development

**Semester**: Winter 2026


---

## Demo Video

🎥 [Watch Demo Video](https://www.youtube.com/watch?v=gqqueIYIMZk)

---

## Reflection Questions
    

### RabbitMQ Configuration Analysis

#### 1. Is RabbitMQ a Stateless or Stateful Application?

RabbitMQ is a **stateful** application. Unlike stateless services — which process each request independently and hold no memory between interactions — RabbitMQ actively maintains internal state that is critical to its operation.


---

#### 2. Implications of Running RabbitMQ Without Persistent Storage

The RabbitMQ Deployment in `algonquin-pet-store-all-in-one.yaml` does **not** define any `volumes` or `volumeMounts`, and does not use a `PersistentVolumeClaim` (PVC). This means:

- **All data is stored inside the container's ephemeral (writable) layer**, which is destroyed whenever the container stops.
- **RabbitMQ is deployed as a `Deployment`** rather than a `StatefulSet`. A `Deployment` does not guarantee stable network identity or per-Pod persistent storage, both of which RabbitMQ requires in production.
- **Credentials are hardcoded as plain-text environment variables** (`RABBITMQ_DEFAULT_USER` / `RABBITMQ_DEFAULT_PASS`), exposing a secondary configuration risk, though not the primary storage concern.


---

#### 3. What Happens When the RabbitMQ Pod Is Deleted or Restarted?

When the RabbitMQ pod is deleted, stopped or restarted, — either manually (`kubectl delete pod`), due to a node failure, an OOMKill event, or a rolling update — the following occurs:

| Event | Result |
|---|---|
| Container stops | The entire writable container filesystem is destroyed |
| Kubernetes reschedules the pod | A brand-new container starts with a **clean, empty** RabbitMQ installation |
| Queued messages | **Permanently lost** — even messages declared as `durable` in application code, because durability requires an intact disk to write to |
| Queue / exchange definitions | **Lost** — all topology must be re-declared by producers/consumers on reconnect |
| In-flight messages (unacknowledged) | **Lost** — no record of their existence remains |
| Connected consumers | Disconnected; must reconnect and re-subscribe to the new broker instance |
| Order Service connection | Fails with a connection refused error until the new pod is Ready, causing orders placed during the downtime window to be dropped |

In the context of the Algonquin Pet Store, a pod restart means that any orders submitted between the old pod dying and the new pod becoming healthy are **silently lost** — the Store Front receives no error, the Order Service cannot publish, and the customer never receives a confirmation.

---

#### 4. Potential Solutions


— Convert the Deployment to a StatefulSet

A `StatefulSet` is the correct Kubernetes workload type for stateful applications like RabbitMQ. It provides:

- **Stable, predictable pod names** (`rabbitmq-0`, `rabbitmq-1`, …) required for RabbitMQ cluster node identity.
- **Per-pod PVCs via `volumeClaimTemplates`**, automatically provisioning and binding a dedicated disk to each replica.
- **Ordered, graceful rolling updates**, ensuring the broker is not restarted abruptly.


— Replace RabbitMQ with a Managed Message Broker (e.g., Azure Service Bus)

Offloading the message broker entirely to a cloud-managed service removes the in-cluster storage problem at the cost of moving to a different protocol and pricing model (see Section 5 below).

---

#### 5. Does Azure Service Bus Solve the Issues Identified with the RabbitMQ Configuration?

**Yes — partially and significantly — but with important trade-offs.**

#### What Azure Service Bus Resolves

| RabbitMQ Problem | Azure Service Bus Behaviour |
|---|---|
| Ephemeral storage (no PVC) | No in-cluster storage at all; Microsoft manages all persistence on Azure-backed infrastructure |
| Data loss on pod restart | Messages are stored durably in the Azure Service Bus namespace, completely independent of the Kubernetes cluster |
| Queue/topic definitions lost on restart | Queues and topics exist as Azure resources; they survive any pod or cluster lifecycle event |
| StatefulSet / PVC configuration complexity | Eliminated; no Kubernetes storage objects to provision or manage |
| Operational burden (upgrades, backups, HA) | Fully managed by Azure; geo-redundancy and automatic failover are built-in features of the Premium tier |

Because Azure Service Bus is a **fully managed PaaS service**, the broker state is entirely external to the Kubernetes cluster. Deleting or restarting any pod in the application has zero effect on the message store — orders in-flight remain in the Service Bus queue and will be delivered once the Order Service reconnects.

#### What Azure Service Bus Does NOT Solve (Remaining Considerations)

1. **Protocol change**: RabbitMQ uses AMQP 0-9-1 natively. Azure Service Bus uses AMQP 1.0, a different protocol version. The Order Service connection string (`amqp://myuser:mypassword@rabbitmq:5672/`) and the SDK used in the application code must be updated to use the Azure Service Bus SDK or an AMQP 1.0-compatible client.

2. **Hardcoded credentials**: Migrating to Service Bus requires storing a connection string (SAS key or Managed Identity). If credentials are still passed as plain-text environment variables, the original security concern persists. The correct fix is to use a Kubernetes `Secret` combined with Azure Workload Identity / Managed Identity to avoid embedding credentials in YAML at all.

3. **Cost**: Azure Service Bus charges per million operations and by tier. For a low-throughput student demo this is negligible, but it introduces an ongoing cost that the self-hosted RabbitMQ pod does not.

4. **Vendor lock-in**: The application becomes coupled to Azure-specific APIs, making it harder to run the same stack on a different cloud provider or locally without a Service Bus emulator.

#### Summary

Azure Service Bus **directly and completely solves** the core problem identified in this lab — the loss of messages and queue definitions caused by running RabbitMQ without persistent storage in Kubernetes. It achieves this by externalising the message broker entirely from the cluster, making message durability a platform-level guarantee rather than a configuration responsibility. For production workloads on Azure, it is a sound architectural choice. The remaining action items are updating the application SDK and securing credentials via Kubernetes Secrets or Managed Identity.

---

## Challenges and Learnings (Optional)


---

## Acknowledgments

[Optional: Credit any resources, documentation, or people who helped you]
 

---