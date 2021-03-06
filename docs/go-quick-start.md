---
id: go-quick-start
title: Quick Start
---

This topic helps you install the Temporal server and implement a workflow.

## Install Temporal Server Locally
To run samples locally you need to run Temporal server locally using [instructions](installing-server.md). 

## Start with an empty directory

Create directory for the project

```
mkdir tutorial-go-sdk
```

```
cd tutorial-go-sdk
```

## Initialize Go Modules and SDK Package Dependency

Initialize Go modules
```
> go mod init github.com/temporalio/tutorial-go-sdk
go: creating new go.mod: module github.com/temporalio/tutorial-go-sdk
```

Add dependency to Temporal Go SDK
```bash
> go get go.temporal.io/temporal@v0.23.1
go: downloading go.temporal.io/temporal v0.23.1
go: go.temporal.io/temporal upgrade => v0.23.1
```

## Implement Activities

### Get User Activity

Create file get_user.go

```go
package main

import (
	"context"

	"go.temporal.io/temporal/activity"
)

// GetUser is the implementation for Temporal activity
func GetUser(ctx context.Context) (string, error) {
	logger := activity.GetLogger(ctx)
	logger.Info("GetUser activity called")
	return "Temporal", nil
}
```

### Send Greeting Activity

Create file send_greeting.go

```go
package main

import (
	"context"
	"fmt"

	"go.temporal.io/temporal/activity"
)

// SendGreeting is the implementation for Temporal activity
func SendGreeting(ctx context.Context, user string) error {
	logger := activity.GetLogger(ctx)
	logger.Info("SendGreeting activity called")

	fmt.Printf("Greeting sent to user: %v\n", user)
	return nil
}
```

## Implement Greetings Workflow

Create file greetings.go

```go
package main

import (
	"time"

	"go.temporal.io/temporal/workflow"
	"go.uber.org/zap"
)

// Greetings is the implementation for Temporal workflow
func Greetings(ctx workflow.Context) error {
	logger := workflow.GetLogger(ctx)
	logger.Info("Workflow Greetings started")

	ao := workflow.ActivityOptions{
		ScheduleToStartTimeout: time.Hour,
		StartToCloseTimeout:    time.Hour,
	}
	ctx = workflow.WithActivityOptions(ctx, ao)

	var user string
	err := workflow.ExecuteActivity(ctx, GetUser).Get(ctx, &user)
	if err != nil {
		return err
	}

	err = workflow.ExecuteActivity(ctx, SendGreeting, user).Get(ctx, nil)
	if err != nil {
		return err
	}

	logger.Info("Greetings workflow complete", zap.String("user", user))
	return nil
}
```

## Host Workflows and Activities inside Worker

Create file main.go

```go
package main

import (
	"go.uber.org/zap"

	"go.temporal.io/temporal/client"
	"go.temporal.io/temporal/worker"
)

func main() {
	logger, err := zap.NewDevelopment()
	if err != nil {
		panic(err)
	}

	logger.Info("Zap logger created")

	// The client is a heavyweight object that should be created once
	serviceClient, err := client.NewClient(client.Options{
			Logger: logger,
	})

	if err != nil {
		logger.Fatal("Unable to start worker", zap.Error(err))
	}

	worker := worker.New(serviceClient, "tutorial_tl", worker.Options{})

	worker.RegisterWorkflow(Greetings)
	worker.RegisterActivity(GetUser)
	worker.RegisterActivity(SendGreeting)

	err = worker.Start()
	if err != nil {
		logger.Fatal("Unable to start worker", zap.Error(err))
	}

	select {}
}
```

## Start Worker

Run your worker app which hosts workflow and activity implementations

```bash
> go run *.go
2020-04-07T22:44:53.073-0700    INFO    tutorial-go-sdk/main.go:19      Zap logger created
2020-04-07T22:44:53.111-0700    INFO    internal/internal_worker.go:1021        Started Worker  {"Namespace": "default", "TaskList": "tutorial_tl", "WorkerID": "59260@local@"}
```

## Start workflow execution

```bash
> docker run --network=host --rm temporalio/tctl:0.23.1 wf start --tl tutorial_tl -w Greet_Temporal_1 --wt Greetings --et 3600 --dt 10
Started Workflow Id: Greet_Temporal_1, run Id: b4f8957a-565c-40ad-8495-15a41338f8f4
```

## Workflow Completes Execution

```
2020-04-07T22:46:32.424-0700    INFO    workflows/greetings.go:14       Workflow Greetings started      {"Namespace": "default", "TaskList": "tutorial_tl", "WorkerID": "59260@local@", "WorkflowType": "Greetings", "WorkflowID": "Greet_Temporal_1", "RunID": "b4f8957a-565c-40ad-8495-15a41338f8f4"}
2020-04-07T22:46:32.424-0700    DEBUG   internal/internal_event_handlers.go:466 ExecuteActivity {"Namespace": "default", "TaskList": "tutorial_tl", "WorkerID": "59260@local@", "WorkflowType": "Greetings", "WorkflowID": "Greet_Temporal_1", "RunID": "b4f8957a-565c-40ad-8495-15a41338f8f4", "ActivityID": "0", "ActivityType": "GetUser"}
2020-04-07T22:46:32.452-0700    INFO    activities/get_user.go:12       GetUser activity called {"Namespace": "default", "TaskList": "tutorial_tl", "WorkerID": "59260@local@", "ActivityID": "0", "ActivityType": "GetUser", "WorkflowType": "Greetings", "WorkflowID": "Greet_Temporal_1", "RunID": "b4f8957a-565c-40ad-8495-15a41338f8f4"}
2020-04-07T22:46:32.485-0700    DEBUG   internal/internal_event_handlers.go:466 ExecuteActivity {"Namespace": "default", "TaskList": "tutorial_tl", "WorkerID": "59260@local@", "WorkflowType": "Greetings", "WorkflowID": "Greet_Temporal_1", "RunID": "b4f8957a-565c-40ad-8495-15a41338f8f4", "ActivityID": "1", "ActivityType": "SendGreeting"}
2020-04-07T22:46:32.505-0700    INFO    activities/send_greeting.go:13  SendGreeting activity called    {"Namespace": "default", "TaskList": "tutorial_tl", "WorkerID": "59260@local@", "ActivityID": "1", "ActivityType": "SendGreeting", "WorkflowType": "Greetings", "WorkflowID": "Greet_Temporal_1", "RunID": "b4f8957a-565c-40ad-8495-15a41338f8f4"}
Greeting sent to user: Temporal
2020-04-07T22:46:32.523-0700    INFO    workflows/greetings.go:33       Greetings workflow complete     {"Namespace": "default", "TaskList": "tutorial_tl", "WorkerID": "59260@local@", "WorkflowType": "Greetings", "WorkflowID": "Greet_Temporal_1", "RunID": "b4f8957a-565c-40ad-8495-15a41338f8f4", "user": "Temporal"}
```

## Try Go SDK Samples

Check [Go SDK Samples](https://github.com/temporalio/temporal-go-samples)
and try simple Temporal usage scenario. 
