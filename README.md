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

RabbitMQ is a stateful application. Unlike stateless services — which process each request independently and hold no memory between interactions — RabbitMQ actively maintains internal state that is critical to its operation.


---

#### 2. Implications of Running RabbitMQ Without Persistent Storage

The RabbitMQ Deployment in `algonquin-pet-store-all-in-one.yaml` does **not** define any `volumes` or `volumeMounts`. 

This means:

- All data is stored inside the container is destroyed whenever the container stops.
- RabbitMQ is deployed as a `Deployment`** rather than a `StatefulSet`. A `Deployment` does not guarantee stable network identity or per-Pod persistent storage, both of which RabbitMQ requires in production.
- Credentials are hardcoded as plain-text environment variables (`RABBITMQ_DEFAULT_USER` / `RABBITMQ_DEFAULT_PASS`), exposing a configuration risk, though not the primary storage concern.


---

#### 3. What Happens When the RabbitMQ Pod Is Deleted or Restarted?

When the RabbitMQ Pod is terminated, all messages are Permanently lost.


---

#### 4. Potential Solutions


- Convert the Deployment to a StatefulSet.
- Replace RabbitMQ with a Managed Message Broker like Azure Service Bus.


---

#### 5. Does Azure Service Bus Solve the Issues Identified with the RabbitMQ Configuration?

Yes, it saves from Data and Queue messages loss when pod restarts.

#### What Azure Service Bus Resolves

Azure Service Bus is a **fully managed PaaS service**.


---

## Challenges and Learnings (Optional)


---

## Acknowledgments

[Optional: Credit any resources, documentation, or people who helped you]
 

---