## Internal Service Communication
![Internal Service Communication](hld/service-communication.svg)

This diagram focuses on how the communication between internal services looks like to achive functionality of stirzz app. The scope of this diagram does not include infrastructure, routing or security.

## Infrastructure
![Infrastructure](hld/infrastructure.svg)

This diagram focuses on infrastructre and security it includes following components.
- **Api Gateway**: responsible for authentication (delegated to auth-provider) and authorization.
- **Auth Provider**: responsible for user authentication and user authentication token issuance.
- **Ingress**: responsible for routing to k8s services.
- **K8S Services**: for load-balancing incoming requests to micro-service pods.
- **Pod Replicaset**: Pods replicas for each micro-service.
- **Rabbit MQ Server**: Rabbit MQ Server for handling asynchronous communication between services using rabbitmq exchanges and queues.
- **Database Cluster**: Database for each micro-service.

*Important!* all pods for one service can connect to it's database directly hence connection isolation should be carefully implemented.