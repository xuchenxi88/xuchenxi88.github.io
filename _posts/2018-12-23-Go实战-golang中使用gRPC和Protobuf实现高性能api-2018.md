# Go实战--golang中使用gRPC和Protobuf实现高性能api

## 实现流程
[原文地址](https://blog.csdn.net/wangshubo1989/article/details/78739994)

最近在尝试在golang中使用gRPC框架和protobuf序列化工具实现一个简单的高性能api，简单步骤如下：

- 1. 按照protobuf文件格式编写proto文件

> 这边定义了消费者服务，包括一个GetCustomers和CreateCustomer方法，使用了rpc插件

```protobuf
syntax = "proto3";
package customer;


// The Customer service definition.
service Customer {   
  // Get all Customers with filter - A server-to-client streaming RPC.
  rpc GetCustomers(CustomerFilter) returns (stream CustomerRequest) {}
  // Create a new Customer - A simple RPC 
  rpc CreateCustomer (CustomerRequest) returns (CustomerResponse) {}
}

// Request message for creating a new customer
message CustomerRequest {
  int32 id = 1;  // Unique ID number for a Customer.
  string name = 2;
  string email = 3;
  string phone= 4;

  message Address {
    string street = 1;
    string city = 2;
    string state = 3;
    string zip = 4;
    bool isShippingAddress = 5; 
  }

  repeated Address addresses = 5;
}

message CustomerResponse {
  int32 id = 1;
  bool success = 2;
}
message CustomerFilter {    
  string keyword = 1;
}
```
> 这边编写时曾经奇怪，因为学习proto文件格式时记得每个成员变量都是有require,optional,repeated三个选项的，但例子中并不是所有成员都有，这是因为上述三个选项时proto2的特性，proto3已经做了修改，两者的却别详情见[Protobuf 的 proto3 与 proto2 的区别](https://blog.csdn.net/liujiayu2/article/details/77837463)

- 2. 编译proto文件生成\*pb.go文件，因为使用grpc框架，所以要添加插件选项**plugins**

```protobuf
protoc --go_out=plugins=grpc:. *.proto
```

- 3. 实现服务端和客户端，分别编写相应代码，这边直接参考原文代码
 - 3.1 服务端实现
		- a. 使用tcp协议，监听指定端口
		- b. 创建一个grpc服务
		- c. 注册grpc服务端
		- d. 为监听端口获取的指令提供响应，执行相应函数
		
```go
package main

import (
	"log"
	"net"
	"strings"

	"context"
	"google.golang.org/grpc"

	pb "study/go_grpc_protobuf/customer"
	"fmt"
)

const (
	port = ":50051"
)

// server is used to implement customer.CustomerServer.
type server struct {
	savedCustomers []*pb.CustomerRequest
}

// CreateCustomer creates a new Customer
func (s *server) CreateCustomer(ctx context.Context, in *pb.CustomerRequest) (*pb.CustomerResponse, error) {
	s.savedCustomers = append(s.savedCustomers, in)
	fmt.Println("customer add",in.Name)
	return &pb.CustomerResponse{Id: in.Id, Success: true}, nil
}

// GetCustomers returns all customers by given filter
func (s *server) GetCustomers(filter *pb.CustomerFilter, stream pb.Customer_GetCustomersServer) error {
	for _, customer := range s.savedCustomers {
		if filter.Keyword != "" {
			if !strings.Contains(customer.Name, filter.Keyword) {
				continue
			}
		}
		fmt.Println("customer get",customer.Name)
		if err := stream.Send(customer); err != nil {
			return err
		}
	}
	return nil
}

func main() {
	lis, err := net.Listen("tcp", port)
	if err != nil {
		log.Fatalf("failed to listen: %v", err)
	}
	// Creates a new gRPC server
	s := grpc.NewServer()
	pb.RegisterCustomerServer(s, &server{})
	s.Serve(lis)
}
```
> 在注册grpc服务端时定义了一个server结构，里面包括一个消费者请求队列，并实现了CreateCustomer和GetCustomers接口
 
 - 3.2 客户端实现
	- a. 使用grpc的Dial方法连接本地端口，返回一个\*grpc.ClientConn实例
	- b. 创建一个新消费者客户端，参数为步骤一中创建的实例
	- c. 初始化一个消费者实例
	- d. 调用服务端方法创建消费者
	- e. 调用服务端方法获取刚刚创建的消费者信息

```go
package main

import (
	"io"
	"log"

	"golang.org/x/net/context"
	"google.golang.org/grpc"

	pb "study/go_grpc_protobuf/customer"
)

const (
	address = "localhost:50051"
)

// createCustomer calls the RPC method CreateCustomer of CustomerServer
func createCustomer(client pb.CustomerClient, customer *pb.CustomerRequest) {
	resp, err := client.CreateCustomer(context.Background(), customer)
	if err != nil {
		log.Fatalf("Could not create Customer: %v", err)
	}
	if resp.Success {
		log.Printf("A new Customer has been added with id: %d", resp.Id)
	}
}

// getCustomers calls the RPC method GetCustomers of CustomerServer
func getCustomers(client pb.CustomerClient, filter *pb.CustomerFilter) {
	// calling the streaming API
	stream, err := client.GetCustomers(context.Background(), filter)
	if err != nil {
		log.Fatalf("Error on get customers: %v", err)
	}
	for {
		customer, err := stream.Recv()
		if err == io.EOF {
			break
		}
		if err != nil {
			log.Fatalf("%v.GetCustomers(_) = _, %v", client, err)
		}
		log.Printf("Customer: %v", customer)
	}
}
func main() {
	// Set up a connection to the gRPC server.
	conn, err := grpc.Dial(address, grpc.WithInsecure())
	if err != nil {
		log.Fatalf("did not connect: %v", err)
	}
	defer conn.Close()
	// Creates a new CustomerClient
	client := pb.NewCustomerClient(conn)

	customer := &pb.CustomerRequest{
		Id:    101,
		Name:  "Shiju Varghese",
		Email: "shiju@xyz.com",
		Phone: "732-757-2923",
		Addresses: []*pb.CustomerRequest_Address{
			&pb.CustomerRequest_Address{
				Street:            "1 Mission Street",
				City:              "San Francisco",
				State:             "CA",
				Zip:               "94105",
				IsShippingAddress: false,
			},
			&pb.CustomerRequest_Address{
				Street:            "Greenfield",
				City:              "Kochi",
				State:             "KL",
				Zip:               "68356",
				IsShippingAddress: true,
			},
		},
	}

	// Create a new customer
	createCustomer(client, customer)

	// Filter with an empty Keyword
	filter := &pb.CustomerFilter{Keyword: ""}
	getCustomers(client, filter)
}

```
> 客户端同样需要实现GetCustomers和CreateCustomer接口，Create请求需要携带消费者结构，使服务端生成对应数据并储存；Get请求返回一个grpc.ClientStream数据流，可以从中读取数据

执行服务端，程序会一直监听端口，当执行客户端请求时，服务端会进行响应

客户端执行结果：

```
2018/12/22 22:43:34 A new Customer has been added with id: 101
2018/12/22 22:43:34 A new Customer has been added with id: 102
2018/12/22 22:43:34 Customer: id:101 name:"Shiju Varghese" email:"shiju@xyz.com" phone:"732-757-2923" addresses:<street:"1 Mission Street" city:"San Francisco" state:"CA" zip:"94105" > addresses:<street:"Greenfield" city:"Kochi" state:"KL" zip:"68356" isShippingAddress:true > 
2018/12/22 22:43:34 Customer: id:102 name:"Irene Rose" email:"irene@xyz.com" phone:"732-757-2924" addresses:<street:"1 Mission Street" city:"San Francisco" state:"CA" zip:"94105" isShippingAddress:true > 
```

服务端日志结果:

```
customer add Shiju Varghese
customer add Irene Rose
customer get Shiju Varghese
customer get Irene Rose
```

## 遇到问题

在搭建过程中遇到了几个问题，也查找了很多资料才逐个解决，这边总结下，防止后面再掉坑

- 1. grpc的golang库安装，在很多文章上是执行下述指令:
```go get -u google.golang.org/grpc```
但是会出现i/o timeout错误，这是因为grpc现在已经不在google.golang.org维护，已经迁移到github上面了，所以正确的方式应该是从[grpc-go clone](https://github.com/grpc/grpc-go)

- 2. 也是困扰了我很久的问题，在编译过程中报出如下错误:
```
cannot use handler (type func("context".Context, interface {}) (interface {}, error)) as type grpc.UnaryHandler in argument to interceptor
```
后来查了很多网上资料，发现是版本问题，在[GitHub](https://github.com/grpc/grpc-go/issues/711)上有关于该问题的讨论:

>If you're using the new "context" library, then gRPC servers can't match the server to the interface, because it has the wrong type of context.

看起来是版本匹配问题导致，后来将go1.8.3重新安装为go1.11.4，并重新编译proto文件，上述代码测试通过
