# WhatsApp Bot Project

This project is a WhatsApp bot built using the Go programming language and the WhatsMeow library. It uses PostgreSQL as its database, and it's containerized using Docker.

## Prerequisites

- Go (version 1.16 or later)
- Docker and Docker Compose
- PostgreSQL (version 13 or later)

## How to Run

1. Clone the repository to your local machine.
2. Navigate to the project directory.
3. Build the Docker images using Docker Compose with the following command:

```bash
docker-compose up --build
```

This command will start the PostgreSQL database and the application.

## How It Works
The bot connects to WhatsApp using the WhatsMeow library. If it's the first time running, it will generate a QR code that you need to scan using your phone to authenticate.
Once authenticated, the bot listens for incoming messages.
When a message is received, it sends a reply back to the sender.
The bot uses a PostgreSQL database to store device information.
The database connection details are configured in the docker-compose.yml file.

## Stopping the Application

To stop the application, you can use the following command:

```bash
docker-compose down
```


In this blog post, I'll walk you through the steps to create a simple WhatsApp chatbot using the Go programming language. We'll use the whatsmeow library to connect to WhatsApp and handle incoming messages. Additionally, we will containerize the application using Docker and Docker Compose.

# 1. Introduction
Chatbots have become a popular tool for businesses and developers to automate interactions with users. WhatsApp, being one of the most widely used messaging platforms, is an excellent choice for deploying chatbots. In this tutorial, we'll build a basic chatbot that responds to incoming messages and containerize it for easy deployment.
# 2. Setting Up Your Development Environment
Before we start coding, make sure you have the following installed on your machine:
- Go (version 1.15 or higher)
- PostgreSQL (for storing session data)
- Docker and Docker Compose

You'll also need to install some Go packages. Run the following command to get the required packages:
```go
go mod init whatsapp-chat-bot
```
```go
go get github.com/lib/pq github.com/mdp/qrterminal go.mau.fi/whatsmeow google.golang.org/protobuf/proto
```
# 3. Connecting to WhatsApp
First, let's set up the connection to WhatsApp. Create a new Go file, main.go, and add the following code:
```go
package main

import (
    "context"
    "fmt"
    "os"
    "os/signal"
    "syscall"

    _ "github.com/lib/pq"
    "github.com/mdp/qrterminal"
    "go.mau.fi/whatsmeow"
    waProto "go.mau.fi/whatsmeow/binary/proto"
    "go.mau.fi/whatsmeow/store/sqlstore"
    "go.mau.fi/whatsmeow/types/events"
    waLog "go.mau.fi/whatsmeow/util/log"
    "google.golang.org/protobuf/proto"
)

func main() {
    dbLog := waLog.Stdout("Database", "DEBUG", true)
    dsn := "user=your_username password=your_password dbname=your_dbname host=your_host port=your_port sslmode=disable"
    container, err := sqlstore.New("postgres", dsn, dbLog)
    if err != nil {
        panic(err)
    }

    deviceStore, err := container.GetFirstDevice()
    if err != nil {
        panic(err)
    }

    clientLog := waLog.Stdout("Client", "INFO", true)
    client := whatsmeow.NewClient(deviceStore, clientLog)
    client.AddEventHandler(GetEventHandler(client))

    if client.Store.ID == nil {
        qrChan, _ := client.GetQRChannel(context.Background())
        err = client.Connect()
        if err != nil {
            panic(err)
        }
        for evt := range qrChan {
            if evt.Event == "code" {
                qrterminal.GenerateHalfBlock(evt.Code, qrterminal.L, os.Stdout)
            } else {
                fmt.Println("Login event:", evt.Event)
            }
        }
    } else {
        err = client.Connect()
        if err != nil {
            panic(err)
        }
    }

    c := make(chan os.Signal, 1)
    signal.Notify(c, os.Interrupt, syscall.SIGTERM)
    <-c

    client.Disconnect()
}
```
Replace your_username, your_password, your_dbname, and your_host with your PostgreSQL credentials and host information.
# 4. Handling Incoming Messages
Next, let's add the functionality to handle incoming messages. We will define an event handler that processes messages and sends a response.
Add the following function above the main function:
```go
func GetEventHandler(client *whatsmeow.Client) func(interface{}) {
    return func(evt interface{}) {
        switch v := evt.(type) {
        case *events.Message:
            var messageBody = v.Message.GetConversation()
            fmt.Printf("Received message: %s\n", messageBody)

            client.SendMessage(context.Background(), v.Info.Chat, &waProto.Message{
                Conversation: proto.String("Received: " + messageBody),
            })
        }
    }
}
```
This function will print the received message to the console and send a response back to the sender.
# 5. Running the Bot
To run the bot locally, execute the following command in your terminal:
go run main.go
go run main.go
If this is the first time you're running the bot, you'll see a QR code in the terminal. Scan this QR code with your WhatsApp to link the bot to your WhatsApp account.
# 6. Dockerizing the Application
To make deployment easier, let's containerize the application using Docker. Here are the necessary Docker and Docker Compose configurations.
# Dockerfile
Create a file named Dockerfile and add the following content:
```dockerfile

FROM --platform=$BUILDPLATFORM golang:1.22.2-alpine AS builder

WORKDIR /code

ENV CGO_ENABLED 0
ENV GOPATH /go
ENV GOCACHE /go-build

# Cache go mod dependencies
COPY go.mod go.sum ./
RUN --mount=type=cache,target=/go/pkg/mod/cache \
    go mod download

# Copy the source code
COPY . .

# Build the Go binary
RUN --mount=type=cache,target=/go/pkg/mod/cache \
    --mount=type=cache,target=/go-build \
    go build -o bin/backend main.go

# Development environment setup
FROM builder AS dev-envs

RUN apk update && apk add git

# Create a non-root user
RUN addgroup -S docker && adduser -S --shell /bin/bash --ingroup docker vscode

# Install Docker tools (cli, buildx, compose)
COPY --from=gloursdocker/docker / /

# Run the application in development mode
CMD ["go", "run", "main.go"]

# Production image
FROM scratch
COPY --from=builder /code/bin/backend /usr/local/bin/backend
CMD ["/usr/local/bin/backend"]
```
# Docker Compose Configuration
Create a file named docker-compose.yml and add the following content:
## Running the Bot with Docker Compose
```dockerfile
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
      target: builder
    ports:
      - "8080:8080"
    depends_on:
      db:
        condition: service_healthy
    environment:
      DB_USER: galibot
      DB_PASSWORD: your_secure_password
      DB_NAME: galibot
      DB_HOST: db
      DB_PORT: 5432
  db:
    image: postgres:13
    environment:
      POSTGRES_PASSWORD: your_secure_password
      POSTGRES_DB: galibot
      POSTGRES_USER: galibot
    volumes:
      - db-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U galibot"]
      timeout: 20s
      retries: 10

volumes:
  db-data:
```
To run the bot using Docker Compose, execute the following command in your terminal:
```shell
docker-compose up --build
```
Docker Compose will build the Docker images, set up the PostgreSQL database, and start the chatbot application.
# 7. Conclusion
In this tutorial, we've built a simple WhatsApp chatbot using Go and containerized it with Docker. We've covered setting up the development environment, connecting to WhatsApp, handling incoming messages, and running the bot both locally and in a containerized environment. This is a basic implementation, and you can expand it by adding more sophisticated message handling and integrating with external APIs.
Feel free to share your thoughts and improvements in the comments section below. Happy coding!

Github
https://github.com/galileoluna/chatbot-whatsapp
#golang #codingtips 
