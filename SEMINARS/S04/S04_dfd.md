# S04 – DFD

```mermaid
flowchart LR
User[Клиент] -->|JWT / login data| API[API Gateway]
API -->|SQL queries| DB[(Database)]
API -->|HTTP requests| ExternalAPI[External API]
ExternalAPI -->|HTTP responses| API
```
