Every project needs to know if there backend services are running up and healthy. And if your project is a distributed system where multiples services and multiple instances of services are running, you need a metric collector which monitors these services and reports if they are down.

### Kube Pulse

A distributed health check & metrics aggregator is the perfect balance. Though this blog post only covers how to create a simple distributed health checker pattern using Go programming language and gRPC Health Check Protocol.
The goal is to built a system where a "Central Collector" service polls several "Agent" services via gRPC to monitor their health or system stats (CPU/Memory), then exposes a summary via a REST endpoint. This blog post also does not covers how to exposes a REST endpoint, and I will plan to add it later and update this blog post.

#### Tools

- Go programming language
- gRPC Health Check Protocol library

As this blog post is part of the project I am working on, you can find the ongoing project [here](https://github.com/vinaymanala/NationPulse) for reference.

At the time of writing this blog post, the project consists of a `bff`, `reporting`, `ingestion`, `cronjob` services. And to monitor these services I have created `kubepulse` service.

#### Implementation

To create a kube pulse service, create a new go binary project. Inside `cmd/main.go` paste the code shown below.

```go
package main

import (
	"fmt"
	"os"
	"time"

	"github.com/kubepulse/internal"
)

func main() {

	// List all service to health check
	services := map[string]string{
		"BFF-Service":   os.Getenv("BFF_ADDR"),
		"Reporting-Svc": os.Getenv("REPORTING_ADDR"),
		"Ingestion-Svc": os.Getenv("INGESTION_ADDR"),
		"Cronjob-Svc":   os.Getenv("CRONJOB_ADDR"),
	}

	for {
		fmt.Println("\n--- Pulse check:", time.Now().Format(time.Kitchen), "---")
		for name, addr := range services {
			go func() {
				status := internal.CheckHealth(addr)
				fmt.Printf("[%s]  Status:%s\n", name, status)
			}()
		}
		time.Sleep(10 * time.Second)
	}
}
```

The code registers all services in a map and loop overs each services in a new goroutine to check the health status concurrently every 10 seconds.

To implement `CheckHealth` function, inside `internal/healthcheck.go` paste the code shown below.

```go
package internal

import (
	"context"
	"time"

	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials/insecure"
	"google.golang.org/grpc/health/grpc_health_v1"
)

func CheckHealth(addr string) string {
	ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
	defer cancel()
	// Connect to the service
	conn, err := grpc.DialContext(ctx, addr, grpc.WithTransportCredentials(insecure.NewCredentials()))
	if err != nil {
		return "OFFLINE (Connection Error)"
	}
	defer conn.Close()

	// Use the standard health client
	client := grpc_health_v1.NewHealthClient(conn)
	resp, err := client.Check(ctx, &grpc_health_v1.HealthCheckRequest{Service: ""})
	if err != nil {
		return "UNHEALTHY (RPC ERROR)"
	}
	return resp.GetStatus().String()
}
```

This code creates a context timeout to make sure the check health should complete under 2 seconds or it cancels the context and releases the resources holding the program to wait.
The gRPC connection is made to the service `addr` passed as an argument in the function and handles the error in case connection to the service fails.
After the connection is successful, a health check is performed for the service requested and handles the error case if the service is not serving or unhealthy.
This completes the client Kube pulse service implementation.

Now at the server side, the backend service also needs to use the health check protocol to respond to the kube pulse requests, by passing the health status. I have used `errgroup` library which provides synchronization, error propagation, and Context cancellation for groups of goroutines working on subtasks of a common task.
This standard definition is exactly what we need to manage all our dependency health checks. Lets see the code for this implementation.

To implement the health check connection, use the below code

```go
// ...other configuration

//create the errgroup context
g, ctx := errgroup.WithContext(ctx)

//grpc server setup
grpcSrv := grpc.NewServer()
healthSrv := health.NewServer()
grpc_health_v1.RegisterHealthServer(grpcSrv, healthSrv)
```

To start the gRPC health check server, use the below code

```go
// start grpc server
        g.Go(func() error {
		lis, err := net.Listen("tcp", ":50051")
		if err != nil {
			return err
		}
		log.Printf("gRPC health server listening at %v\n", lis.Addr())

		// goroutine to stop the gRPC server when the context is cancelled
		go func() {
			<-ctx.Done()
			grpcSrv.GracefulStop()
		}()
		return grpcSrv.Serve(lis)
	})
```

You can use the same setup of errgroup to start a HTTP server for serving web requests and handle graceful shutdown with ease.

Let's see the dependency health checker for postgres and redis code:

```go
g.Go(func() error {
		ticker := time.NewTicker(10 * time.Second)
		defer ticker.Stop()
		for {
			select {
			case <-ctx.Done():
				return nil
			case <-ticker.C:
				status := grpc_health_v1.HealthCheckResponse_SERVING

				//check redis & postgres
				if err := db.Client.Ping(ctx); err != nil {
					logger.Error("DB down", zap.Error(err))
					status = grpc_health_v1.HealthCheckResponse_NOT_SERVING
				}
				if err := rds.Client.Ping(ctx).Err(); err != nil {
					logger.Error("Redis down", zap.Error(err))
					status = grpc_health_v1.HealthCheckResponse_NOT_SERVING
				}

				healthSrv.SetServingStatus("", status)
			}
		}
	})
```

This code pings the postgres and redis instances running and executes what is call an empty operation like a preflight request to check if the instance is healthy and running.

Finally, the main goroutine needs to wait until all operations/functions have returned, and return the first non-nil error (if any).

The setup to add this into kubernetes cluster is not the scope of this blog post. However you can find all the manifests for deployments and infra-config used [here](https://github.com/vinaymanala/NationPulse/tree/main/k8s)

You can find more about the setup and the project [here](https://github.com/vinaymanala/NationPulse). This project is still under development and would appreciate your support or contribution in helping this project in any way possible.
