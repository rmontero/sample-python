# sample-python
Just a tiny sample: microservices integrations via REST/gRPC APIs and via messaging patterns (events/queues)

To design and deliver robust microservices integrations using REST/gRPC APIs and messaging patterns (events/queues), I'll provide examples in both Python and Kotlin. Let's start with a simple microservice in each language that communicates with another microservice via REST, gRPC, and event/queue-based messaging.

These examples demonstrate basic REST, gRPC, and messaging patterns using Python and Kotlin.

# Python Example
## Microservice 1: REST API with Flask

```
# app.py
from flask import Flask, jsonify, request

app = Flask(__name__)

@app.route('/api/data', methods=['GET'])
def get_data():
    data = {"message": "Hello from Microservice 1!"}
    return jsonify(data)

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

## Microservice 2: gRPC with gRPCio
First, define the gRPC service in a .proto file:

```
// service.proto
syntax = "proto3";

package service;

service DataService {
    rpc GetData (Empty) returns (DataResponse);
}

message Empty {}

message DataResponse {
    string message = 1;
}
```

Generate the gRPC code using protoc:

`python -m grpc_tools.protoc -I. --python_out=. --grpc_python_out=. service.proto`

Then, implement the server:

```
# server.py
from concurrent import futures
import grpc
import service_pb2
import service_pb2_grpc

class DataService(service_pb2_grpc.DataServiceServicer):
    def GetData(self, request, context):
        return service_pb2.DataResponse(message="Hello from gRPC Microservice 1!")

def serve():
    server = grpc.server(futures.ThreadPoolExecutor(max_workers=10))
    service_pb2_grpc.add_DataServiceServicer_to_server(DataService(), server)
    server.add_insecure_port('[::]:50051')
    server.start()
    server.wait_for_termination()

if __name__ == '__main__':
    serve()
```

Messaging with RabbitMQ
Producer:

```
# producer.py
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()
channel.queue_declare(queue='hello')

channel.basic_publish(exchange='', routing_key='hello', body='Hello from Microservice 1!')
print(" [x] Sent 'Hello from Microservice 1!'")
connection.close()
```

Consumer:

```
# consumer.py
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()
channel.queue_declare(queue='hello')

def callback(ch, method, properties, body):
    print(" [x] Received %r" % body)

channel.basic_consume(queue='hello', on_message_callback=callback, auto_ack=True)
print(' [*] Waiting for messages. To exit press CTRL+C')
channel.start_consuming()
```

#Kotlin example
## Microservice 1: REST API with Ktor

```
// build.gradle.kts
plugins {
    kotlin("jvm") version "1.5.21"
    id("io.ktor.plugin") version "1.5.4"
}

repositories {
    mavenCentral()
}

dependencies {
    implementation("io.ktor:ktor-server-netty:1.5.4")
    implementation("io.ktor:ktor-jackson:1.5.4")
    testImplementation("io.ktor:ktor-server-tests:1.5.4")
    testImplementation(kotlin("test-junit"))
}
```

```
// Application.kt
import io.ktor.application.*
import io.ktor.features.ContentNegotiation
import io.ktor.jackson.jackson
import io.ktor.response.*
import io.ktor.request.*
import io.ktor.routing.*
import io.ktor.server.engine.*
import io.ktor.server.netty.*

fun main() {
    embeddedServer(Netty, port = 8080) {
        install(ContentNegotiation) {
            jackson { }
        }
        routing {
            get("/api/data") {
                call.respond(mapOf("message" to "Hello from Microservice 1!"))
            }
        }
    }.start(wait = true)
}
```
## Microservice 2: gRPC with Kotlin
First, define the gRPC service in a .proto file (same as above):

```
// service.proto
syntax = "proto3";

package service;

service DataService {
    rpc GetData (Empty) returns (DataResponse);
}

message Empty {}

message DataResponse {
    string message = 1;
}
```

Generate the gRPC code using protoc:

`protoc --proto_path=src/main/proto --java_out=src/main/java --grpc-java_out=src/main/java src/main/proto/service.proto
`

Then, implement the server:

```
// build.gradle.kts
dependencies {
    implementation("io.grpc:grpc-kotlin-stub:1.0.0")
    implementation("io.grpc:grpc-netty-shaded:1.39.0")
    implementation("com.google.protobuf:protobuf-java:3.17.3")
}

```

```
// Server.kt
import io.grpc.Server
import io.grpc.ServerBuilder
import io.grpc.stub.StreamObserver
import service.DataResponse
import service.Empty
import service.DataServiceGrpc

class DataServiceImpl : DataServiceGrpc.DataServiceImplBase() {
    override fun getData(request: Empty, responseObserver: StreamObserver<DataResponse>) {
        val response = DataResponse.newBuilder().setMessage("Hello from gRPC Microservice 1!").build()
        responseObserver.onNext(response)
        responseObserver.onCompleted()
    }
}

fun main() {
    val server: Server = ServerBuilder.forPort(50051)
        .addService(DataServiceImpl())
        .build()
        .start()
    server.awaitTermination()
}

```

Messaging with RabbitMQ
Producer:

```
// build.gradle.kts
dependencies {
    implementation("com.rabbitmq:amqp-client:5.12.0")
}

```

```
// Producer.kt
import com.rabbitmq.client.ConnectionFactory

fun main() {
    val factory = ConnectionFactory()
    factory.host = "localhost"
    factory.newConnection().use { connection ->
        connection.createChannel().use { channel ->
            channel.queueDeclare("hello", false, false, false, null)
            val message = "Hello from Microservice 1!"
            channel.basicPublish("", "hello", null, message.toByteArray())
            println(" [x] Sent '$message'")
        }
    }
}

```

Consumer:
```
// Consumer.kt
import com.rabbitmq.client.ConnectionFactory

fun main() {
    val factory = ConnectionFactory()
    factory.host = "localhost"
    factory.newConnection().use { connection ->
        connection.createChannel().use { channel ->
            channel.queueDeclare("hello", false, false, false, null)
            val deliverCallback = { _: String?, delivery: com.rabbitmq.client.Delivery ->
                val message = String(delivery.body, charset("UTF-8"))
                println(" [x] Received '$message'")
            }
            channel.basicConsume("hello", true, deliverCallback, { _ -> })
            println(" [*] Waiting for messages. To exit press CTRL+C")
            Thread.sleep(10000)  // Keep the consumer running for 10 seconds for demonstration
        }
    }
}

```

# Infrastructure Requirements
Python Environment:

Install Python 3.8 or higher.
Install dependencies:
sh
Copy code
pip install -r requirements.txt
gRPC Code Generation:

Generate gRPC code:
sh
Copy code
python -m grpc_tools.protoc -I. --python_out=. --grpc_python_out=. microservice2/service.proto
RabbitMQ:

Install RabbitMQ server.
Start RabbitMQ server:
sh
Copy code
rabbitmq-server
Run Services:

Start REST API:
sh
Copy code
cd microservice1
python app.py
Start gRPC server:
sh
Copy code
cd microservice2
python server.py
Start Producer:
sh
Copy code
cd messaging
python producer.py
Start Consumer:
sh
Copy code
cd messaging
python consumer.py
