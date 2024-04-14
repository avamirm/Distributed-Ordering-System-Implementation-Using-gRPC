# Distributed Ordering System Implementation Using gRPC

## Overview
This project implements a distributed ordering system using gRPC (Google Remote Procedure Call), consisting of a server and a client side. The server is responsible for responding to client requests for items, querying a database to check for item availability, and sending the requested items back to the client. Communication between the client and server can be configured to be bidirectional or server-streaming.

## Prerequisite & Dependencies
These project requires the following tools to be installed on your system:
- Go version 1.19 or later
- Protocol Buffers (protoc) compiler ([more info](https://grpc.io/docs/languages/go/quickstart/#prerequisites))
- Go gRPC plugins

## How to run?

### Windows:
1. Run the following command to build the necessary files:

   ```bash
   go build pkg/*.go
   ```
2. Run the Server:

   ```bash
   go run server/server.go
   ```
3. Run the Client:
	
   ```bash
   go run client/client.go
   ```

### Linux:
<!-- run make command. it will generate output files in bin directory. -->
1. Run `make` command. It will generate output files in the bin directory.
2. Run the Server:

   ```bash
   ./bin/server
   ```
3. Run the Client:

   ```bash
   ./bin/client
   ```

## gRPC Communication Modes
In gRPC, there are four types of communication modes that can be used to define the interaction between the client and the server:
* Unary RPC: In a unary RPC, the client sends a single request to the server and receives a single response from the server. This is the simplest form of RPC and is similar to a traditional remote procedure call.
![Unary RPC](./Documents/bb5ced3a-image1.png)
* Server Streaming RPC: In a server streaming RPC, the client sends a single request to the server, and the server responds with a stream of messages asynchronously. This pattern is useful for scenarios where the server needs to send multiple messages to the client in response to a single request. For example, retrieving a list of items that match a search query.
![Server Streaming RPC](./Documents/c6826dd4-image3.png)
* Client Streaming RPC: In a client streaming RPC, the client sends a stream of messages to the server, and the server responds with a single response. This pattern is useful for scenarios where the client needs to send a large amount of data to the server. For example, uploading a file or sending a batch of requests.
![Client Streaming RPC](./Documents/74a4c5f9-image4.png)
* Bidirectional Streaming RPC: In a bidirectional streaming RPC, both the client and the server send a stream of messages to each other asynchronously. This pattern is useful for scenarios where both the client and the server need to send and receive multiple messages during the lifetime of the RPC. This pattern is wildly used in chat applications, real-time data processing, and collaborative editing.
![Bidirectional RPC](./Documents/414e501c-image5.png)





# Server

At first we need to import the necessary packages.

```go
import (
  "io"
  "log"
  "net"
  "time"
  "strconv"
  handler "user/ordersystem/src/handler"
  pb "user/ordersystem/src/proto"

  "google.golang.org/grpc"
)
```

Then we need to define the server struct and implement the interface. The interface is generated by the protobuf compiler. The interface is used to define the service and the methods.

```go
type server struct {
  pb.UnimplementedOrderServiceServer
}
```

### Server Streaming RPC:

Server streaming is a gRPC feature that allows the server to send multiple messages to the client in response to a single client request. This is useful when the server has a potentially large or indefinite amount of data to send back to the client, and it's not practical or efficient to send it all at once.  
In the provided code below, we have implemented the server streaming method. This method implements the `GetOrderServerStreaming` method of the OrderManagement gRPC service interface. The method receives two parameters:

- req \*pb.OrderRequest: This parameter contains the request message sent by the client. In this case, it's an OrderRequest message, which is an order.
- stream pbOrderManagement_GetOrderServerStreamingServer: This parameter is a server-side stream that the method uses to send multiple OrderResponse messages back to the client.
  At the beginning of the method, we call the `FindOrderByItemName` method to retrieve the orders that match the item name specified in the request. We then iterate over the matching orders and send each one back to the client using the stream.Send method.

```go
func (s *server) GetOrderServerStreaming(req *pb.OrderRequest, stream pb.OrderManagement_GetOrderServerStreamingServer) error {
  found, orders := handler.FindOrderByItemName(req.Items)
  if found {
    for _, order := range orders {
      if err := stream.Send(&pb.OrderResponse{ItemName: order, TimeStamp: strconv.Itoa(time.Now().Second())}); err != nil {
        return err
      }
    }
  }
  return nil
}
```

Also the `FindOrderByItemName` method implementation is as below:

```go
var ServerOrders []string = []string{"banana", "apple", "orange", "grape", "red apple", "kiwi", "mango", "pear", "cherry", "green apple"}

func FindOrderByItemName(itemName string) (bool, []string) {
	var orders []string
	found := false
	for _, serverOrder := range ServerOrders {
		if strings.Contains(serverOrder, itemName) {
			orders = append(orders, serverOrder)
			found = true
		}
	}
	return found, orders
}
```

The `FindOrderByItemName` method takes an item name as a parameter and returns a boolean value indicating whether any orders match the item name and a slice of strings containing the matching orders. It iterates over the ServerOrders slice and checks if each order contains the specified item name. If a match is found, the order is added to the orders slice, and the found flag is set to true. The method then returns the found flag and the orders slice.

### Bidirectional Streaming RPC:

Bidirectional streaming is a gRPC feature that allows both the client and the server to send multiple messages to each other in any order. This is useful when both the client and the server need to send and receive multiple messages during the lifetime of the RPC.
In the provided code below, we have implemented the bidirectional streaming method. This

```go
func (s *server) GetOrderBidirectional(stream pb.OrderManagement_GetOrderBidirectionalServer) error {
  for {
    req, err := stream.Recv()
    if err == io.EOF {
      return nil
    }
    if err != nil {
      return err
    }
    found, orders := handler.FindOrderByItemName(req.Items)
    if found {
      for _, order := range orders {
        if err := stream.Send(&pb.OrderResponse{ItemName: order, TimeStamp: strconv.Itoa(time.Now().Second())}); err != nil {
          return err
        }
      }
    }
  }
}
```

The function `GetOrderBidirectional` is defined with a single parameter `stream`, which is a bidirectional stream. This stream allows both sending messages to the client (stream.Send()) and receiving messages from the client (stream.Recv()). The method starts an infinite loop using for{}. This loop will continue until the client closes the stream or an error occurs. Inside the loop, the method receives a message from the client using `stream.Recv()`. If the client has closed the stream, the method returns nil to indicate that the RPC has completed successfully. If an error occurs while receiving the message, the method returns the error to the client. If the message is received successfully, the method calls the `FindOrderByItemName` method to retrieve the orders that match the item name specified in the request. It then iterates over the matching orders and sends each one back to the client using the `stream.Send()` method.

```go
func main() {
  lis, err := net.Listen("tcp", ":50051")
  if err != nil {
    log.Fatalf("failed to listen: %v", err)
  }

  s := grpc.NewServer()

  pb.RegisterOrderManagementServer(s, &server{})
  log.Println("Server started")
  if err := s.Serve(lis); err != nil {
    log.Fatalf("failed to serve: %v", err)
  }
}
```

This part of the code is the entry point for our gRPC server application. `net.Listen("tcp", ":50051")` creates a TCP network listener on port 50051. This listener will accept incoming connections from clients attempting to communicate with our gRPC server. `grpc.NewServer()` creates a new gRPC server instance. This server will handle incoming gRPC requests from clients. `pb.RegisterOrderManagementServer(s, &server{})` registers your implementation of the gRPC service defined in your .proto file with the gRPC server. This tells the server to route incoming requests to the appropriate methods in your implementation. `RegisterOrderManagementServer` is an auto-generated function that registers your service implementation with the gRPC server. `s.Serve(lis)` starts the gRPC server and begins listening for incoming connections on the listener (lis) created earlier. If an error occurs during server startup, the application will log the error and exit.


# Client

At first we need to import the necessary packages.

```go
import (
	"bufio"
	"context"
	"fmt"
	"io"
	"log"
	"os"
	pb "user/ordersystem/src/proto"

	"google.golang.org/grpc"
)
```

### Server Streaming RPC:

In Server Streaming RPC, the client sends a single request to the server, and the server responds with a stream of messages asynchronously. This pattern is useful for scenarios where the client needs to receive a large amount of data or a sequence of messages from the server in response to a single request.

The `GetInputServerStreaming` function reads the user's input from the command line and returns the entered order as a string.

```go
func GetInputServerStreaming() string {
	fmt.Println("Enter order for Server streaming:")
	scanner := bufio.NewScanner(os.Stdin)
	scanner.Scan()
	if err := scanner.Err(); err != nil {
		log.Fatalf("Error reading input: %v", err)
	}
	return scanner.Text()
}
```

The `ServerStreaming` prompts the user to enter an order for server streaming RPC. It achieves this by calling the GetInputServerStreaming() function, which reads the user's input from the command line and returns the entered order as a string. Once the user enters the order, the function constructs an OrderRequest message containing the order and sends it to the server using the GetOrderServerStreaming RPC method of the provided client. This initiates the server streaming RPC, where the server will asynchronously send multiple responses to the client based on the received order. Then in for loop receive responses from the server via getOrderClient.Recv() and print the response.

```go
func ServerStreaming(client pb.OrderManagementClient) {
	orderRequest := &pb.OrderRequest{Items: GetInputServerStreaming()}
	getOrderClient, err := client.GetOrderServerStreaming(context.Background(), orderRequest)
	if err != nil {
		log.Fatalf("Error calling GetOrderServerStreaming: %v", err)
	}
	for {
		orderResponse, err := getOrderClient.Recv()
		if err == io.EOF {
			break
		}
		if err != nil {
			log.Fatalf("Error receiving response: %v", err)
		}
		log.Printf("Order: %s", orderResponse)
	}
}
```

### Bidirectional Streaming RPC:

In Bidirectional Streaming RPC, the client and server can send a stream of messages to each other asynchronously. This pattern is useful for scenarios where the client and server need to send and receive multiple messages in parallel.

The `GetInputBidirectionalStreaming` function reads the user's input from the command line and returns the entered orders as an string array.

```go
func GetInputBidirectional() []string {
	var orders []string
	fmt.Println("Enter orders for Bidirectional streaming:")
	scanner := bufio.NewScanner(os.Stdin)
	for scanner.Scan() {
		text := scanner.Text()
		if text == "" {
			break
		}
		orders = append(orders, text)
	}

	if err := scanner.Err(); err != nil {
		log.Fatalf("Error reading input: %v", err)
	}

	return orders
}
```

The `BidirectionalStreaming` prompts the user to enter multiple orders for bidirectional streaming RPC. It achieves this by calling the GetInputBidirectional() function, which reads the user's input from the command line and returns the entered orders as a string array. Once the user enters the orders, the function constructs an OrderRequest message containing the orders and sends it to the server using the ProcessOrdersBidirectional RPC method of the provided client. This initiates the bidirectional streaming RPC, where the client and server will asynchronously send and receive multiple messages in parallel. Then in for loop receive responses from the server via processOrderClient.Recv() and print the response.

```go
func BidirectionalStreaming(client pb.OrderManagementClient) {
	orderRequests := GetInputBidirectional()
	getOrderClient, err := client.GetOrderBidirectional(context.Background())
	if err != nil {
		log.Fatalf("Error calling GetOrderBidirectional: %v", err)
	}
	for _, orderRequest := range orderRequests {
		request := &pb.OrderRequest{Items: orderRequest}
		if err := getOrderClient.Send(request); err != nil {
			log.Fatalf("Error sending request: %v", err)
		}
	}

	for {
		orderResponse, err := getOrderClient.Recv()
		if err == io.EOF {
			break
		}
		if err != nil {
			log.Fatalf("Error receiving response: %v", err)
		}
		log.Printf("Order: %s", orderResponse)
	}
}
```

The `ConnectToServer` function is used to establish a connection to the gRPC server on localhost at port 50051. It creates a new gRPC client instance using the pb.NewOrderManagementClient method and returns the client and the connection.
```go
func ConnectToServer() (pb.OrderManagementClient, *grpc.ClientConn) {
	conn, err := grpc.Dial("localhost:50051", grpc.WithTransportCredentials(insecure.NewCredentials()))
	if err != nil {
		log.Fatalf("Failed to connect: %v", err)
	}
	client := pb.NewOrderManagementClient(conn)
	return client, conn
}
```

# Proto File
```proto
syntax = "proto3";
```
This line specifies that this .proto file is written using the Protobuf version 3 syntax. The option `go_package` specifies the Go package name that will be used when generating Go code from this .proto file. In this case, the Go package name is set to user/ordersystem.The option `java_multiple_files` tells the Protobuf compiler to generate separate Java files for each message and service defined in this .proto file. By default, Protobuf generates a single .java file containing all messages and services.
```proto
package distributedOrderingSystem;
```
This line specifies the package namespace for the messages and services defined in this .proto file.
- Service Declaration
```proto
service OrderManagement {
    rpc getOrderServerStreaming(OrderRequest) returns (stream OrderResponse);
    rpc getOrderBidirectional(stream OrderRequest) returns (stream OrderResponse);
}
```
This block defines a gRPC service named `OrderManagement``. It contains two RPC methods: `getOrderServerStreaming` which is a server streaming RPC. It takes an OrderRequest message as input and returns a stream of OrderResponse messages.
getOrderBidirectional: The `getOrderBidirectional` method which is a bidirectional streaming RPC. It takes a stream of OrderRequest messages as input and returns a stream of OrderResponse messages.
- Message Definitions
```proto
message OrderRequest {
    string items = 1;
}

message OrderResponse {
    string itemName = 1;
    string timeStamp = 2;
}
```
These blocks define two message types:

OrderRequest: This message type represents a request sent to the server. It contains a single field named items, which is of type string. The field number for items is 1.
OrderResponse: This message type represents a response sent by the server. It contains two fields:
itemName: A field of type string representing the name of an item.
timeStamp: A field of type string representing a timestamp. The field numbers for itemName and timeStamp are 1 and 2, respectively.  
n Protocol Buffers (protobuf), the numbers associated with message fields and service methods are called field numbers or tag numbers. These numbers serve several purposes: 
- Uniquely Identifying Fields:
- Forward and Backward Compatibility:
- Efficient Encoding:
- compact binary format
These field numbers are used during message serialization and deserialization to identify and parse the fields correctly. They play a crucial role in ensuring the interoperability and efficiency of communication between different systems using Protocol Buffers.