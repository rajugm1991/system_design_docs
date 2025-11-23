Resliance 4j [Understand the usecase and configuration. Flow of each and order of execution. State tansiation ]
 Circuit breaker
 Retry
 Timeout
 Fallback 
 Bulkahead pattern

Microservice communcication
 RestTemplate
 RestClient
 WebClient 
 WebFlux
 Event driven use case

 | Requirement                         | Choose                 |
| ----------------------------------- | ---------------------- |
| Synchronous & simple                | RestClient             |
| Calling 5 microservices in parallel | WebClient + Futures    |
| Streaming (SSE/WebSockets)          | WebFlux                |
| Background workflow                 | Kafka                  |
| Retry + guaranteed delivery         | Kafka                  |
| Business transaction pipeline       | Kafka + Outbox         |
| External API call                   | RestClient / WebClient |


3) Understand the useage of completableFuture async [ Use only for parallel services and non blocking ] 

  