---
title: "Let's talk about Health Checks"
date: 2019-01-12
tags: ["go", "microservices", "services", "redis"]
---

According to the azure documentation in this [excellent article](https://docs.microsoft.com/en-us/azure/architecture/patterns/health-endpoint-monitoring), they state that.

*"It's a good practice, and often a business requirement, to monitor web applications and back-end services, to ensure they're available and performing correctly. However, it's more difficult to monitor services running in the cloud than it is to monitor on-premises services."*

*"There are many factors that affect cloud-hosted applications such as network latency, the performance, and availability of the underlying compute and storage systems and the network bandwidth between them. The service can fail entirely or partially due to any of these factors. Therefore, you must verify at regular intervals that the service is performing correctly to ensure the required level of availability."*

When we work with multiple microservices deployed in a container orchestrator, we have a problem which is "How to detect that a running microservice instance is unable to handle requests?".

## Solution

Implement health monitoring by sending requests to an endpoint on the application. The application should perform the necessary checks, and return an indication of its status "health checks" normally returns 200 if all ok and 503 if the service is failing.

This will be a basic introduction to health checks.

## What are health checks?

Health checks are basically endpoints provided by a microservice (e.g. HTTP /health) to check whether the service is running properly.

## Why should we use health checks?

All microservices should implement health checks. These checks can be used by orchestration tools "as a K8s" to kill an instance or raise an alert to monitoring tool in case of a failing health check.

## What can we check in health checks?
Everything will depend on what do out service or what is our requirements.

For example, if our service using PostgreSQL to persist data or use Redis to cache, we need to ensure that our service can communicate with our storage services "as a PostgreSQL or Redis" because our logic depends on these storage services. if we can't communicate with the database our service cannot work.

Another example is if our microservice receives files, such as images, stores them on disk, we need to check that we have available space on disk. Otherwise, our microservice we will not work.

The most important cases to check are:
* the status of the connections to the infrastructure services used by the service instance
* the status of the others microservices, if it is required.
* the status of the host, e.g. disk space
* application specific logic

I have seen a lot of projects that just implementing health check to return a response with status 200, without doing any checks. For example:

```golang
func HealthCheck(w http.ResponseWriter, r *http.Request) {
	w.WriteHeader(http.StatusOK)
	return
}
```

***Please, don't do this!***

The health check is a powerful ally that allows us to do a lot of things to check the behavior of our microservice and it can avoid us headaches if is combined with some monitoring tool.

I will use my last articles where I wrote a service to look for a driver like uber to implement an example of the health check.

[Tracking Service](https://dev.to/douglasmakey/tracking-service-with-go-and-redis-43mm)

[Tracking Service V2](https://dev.to/douglasmakey/tracking-service-v2-28fd)


We need to create a handler for healthcheck, in this handler we will implement all the checks that we want to do.

If you already read my past articles you have seen that my microservice use redis for register location of drivers and also it uses redis to search drivers with the command 'georadius' function of redis, so my microservice depends 100% on redis, means that I need to check my microservice can communicate with redis.

For this task I use the command Ping of redis, this command is used to test if a connection is still alive, or to measure latency.

File handler/healthcheck.go

```go
func health(w http.ResponseWriter, r *http.Request) {
	if r.Method != "GET" {
		w.WriteHeader(http.StatusMethodNotAllowed)
		return
	}

	// Get instance redis client
	redis := storages.GetRedisClient()
	// Checks that the communication with redis is alive.
	if err := redis.Ping().Err(); err != nil {
		// Put yours logs HERE
		log.Printf("redis unaccessible error: %v ", err)
		w.WriteHeader(http.StatusServiceUnavailable)
	} else {
		w.WriteHeader(http.StatusOK)
	}

	return
}

```

So next we need to add our healtcheck to router.
```go
func NewHandler() *http.ServeMux {
	mux := http.NewServeMux()
        // Add healthcheck
	mux.HandleFunc("/health", health)
        // ....
	mux.HandleFunc("/tracking", tracking)
	mux.HandleFunc("/search", search)

	// V2
	mux.HandleFunc("/v2/search", v2.SearchV2)
	mux.HandleFunc("/v2/cancel", v2.CancelRequest)
	return mux
}
```

Now we run service and run a container on docker with redis.

```bash
docker run -p 6379:6379 -d redis
go run main.go
```

Use curl to consume our new endpoint health to know our service is OK.

```bash
curl -X GET -I localhost:8000/health                                                                                                              1212:50:15
HTTP/1.1 200 OK
Date: Sat, 12 Jan 2019 15:50:23 GMT
Content-Length: 0
```


But if we stop the container with redis and try to hit to '/health' the healthcheck response should be 503 Service Unavailable because our service cant communicates with redis "for we stopped the container with Redis."

```bash
curl -X GET -I localhost:8000/health                                                                                                              1313:38:24
HTTP/1.1 503 Service Unavailable
Date: Sat, 12 Jan 2019 16:38:37 GMT
Content-Length: 0
```

For this microservice just need to check the connection with redis because this service is very simple but depends on your microservice, you need to applied different checks.

[Github Repo](https://github.com/douglasmakey/tracking)