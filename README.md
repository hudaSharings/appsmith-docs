# Appsmith Re-Architecture — Monolith (Java) → Microservices (.NET 10)


## The one-paragraph version

Appsmith is a low-code platform: users drag widgets onto a canvas, connect them to datasources, write queries and JS, and publish an internal tool. Today that is a **Spring WebFlux reactive Java monolith (~163k LOC)** over **MongoDB**, plus a separate Node.js "RTS" process and a **React SPA (~708k LOC)**, all packed into **one Docker container** by supervisord. We are replacing it with **8 deployables on .NET 10** — an API Gateway/BFF plus 7 domain services — each owning its own **PostgreSQL** database, communicating synchronously over REST where a user is waiting and asynchronously over RabbitMQ everywhere else, with **Redis** for locks, sessions, and the SignalR backplane, fronted by a new **Angular** client.

```mermaid
flowchart LR
    subgraph TODAY["Today — one container"]
        direction TB
        C1[React SPA]
        C2[Spring WebFlux monolith<br/>163k LOC Java]
        C3[RTS<br/>Node/Express/Socket.IO]
        C4[(MongoDB)]
        C5[(Redis)]
        C1 --> C2 --> C4
        C2 --> C3
        C2 --> C5
    end

    subgraph TARGET["Target — 8 deployables"]
        direction TB
        T0[Angular SPA]
        T1[API Gateway / BFF]
        T2[Identity &amp; Access]
        T3[Application]
        T4[Datasource]
        T5[Query Execution]
        T6[Git Versioning]
        T7[Realtime]
        T8[Notifications]
        T0 --> T1
        T1 --> T2 & T3 & T4
        T3 --> T5 & T6
        T0 -.WebSocket.-> T7
        T2 & T3 & T4 -.events.-> T8
    end

    TODAY ==>|re-architecture| TARGET
```


