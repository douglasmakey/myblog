---
title: "Understanding Unix Domain Sockets in Golang"
date: 2022-12-05
tags: ["go", "linux", "socket", "network"]
cover:
    image: "/img/unix-domain-sockets-by-julia-evans.png"
---

In Golang, a socket is a communication endpoint that allows a program to send and receive data over a network. There are two main types of sockets in Golang: Unix domain sockets (AF_UNIX) and network sockets (AF_INET|AF_INET6). This blog post will explore some differences between these two types of sockets.

Unix domain sockets, a.k.a., local sockets, are used for communication between processes on the same machine. They use a file-based interface and can be accessed using the file system path, just like regular files. In some cases, Unix domain sockets are faster and more efficient than network sockets as they do not require the overhead of network protocols and communication. They are commonly used for interprocess communication (IPC) and communication between services on the same machine. The data is transmitted between processes using the file system as the communication channel.

The file system provides a reliable and efficient mechanism for transmitting data between processes. Only the kernel is involved in the communication between processes. The processes communicate by reading and writing to the same socket file, which is managed by the kernel. The kernel is responsible for handling the communication details, such as synchronization, buffering and error handling, and ensures that the data is delivered reliably and in the correct order.

This is different from network sockets, where the communication involves the kernel, the network stack, and the network hardware. The processes use network protocols, such as TCP/UDP, in network sockets to establish connections and transfer data over the network. The kernel and the network stack handle the communication details, such as routing, addressing, and error correction. The network hardware handles the physical transmission of the data over the network.

In Golang, Unix domain sockets are created using the `net.Dial` "client" or `net.Listen` "server" functions, with the `unix` network type. For example, the following code creates a Unix domain socket and listens for incoming connections:

```go
// Create a Unix domain socket and listen for incoming connections.
socket, err := net.Listen("unix", "/tmp/mysocket.sock")
if err != nil {
    panic(err)
}

```

Let's look at an example of how to use Unix domain sockets in Golang. The following code creates a simple echo server using a Unix domain socket:

```go
package main
import (...)

func main() {
	// Create a Unix domain socket and listen for incoming connections.
	socket, err := net.Listen("unix", socketPath)
	if err != nil {
		log.Fatal(err)
	}

	// Cleanup the sockfile.
	c := make(chan os.Signal, 1)
	signal.Notify(c, os.Interrupt, syscall.SIGTERM)
	go func() {
		<-c
		os.Remove("/tmp/echo.sock")
		os.Exit(1)
	}()

	for {
		// Accept an incoming connection.
		conn, err := socket.Accept()
		if err != nil {
			log.Fatal(err)
		}

		// Handle the connection in a separate goroutine.
		go func(conn net.Conn) {
			defer conn.Close()
			// Create a buffer for incoming data.
			buf := make([]byte, 4096)

			// Read data from the connection.
			n, err := conn.Read(buf)
			if err != nil {
				log.Fatal(err)
			}

			// Echo the data back to the connection.
			_, err = conn.Write(buf[:n])
			if err != nil {
				log.Fatal(err)
			}
		}(conn)
	}
}

```

Let's test the above echo server using `netcat`, you can use the `-U` option to specify the socket file. This allows you to connect to the socket and send and receive data through it.

```bash
$ echo "I'm a Kungfu Dev" | nc -U /tmp/echo.sock
I'm a Kungfu Dev
```

Network sockets, on the other hand, are used for communication between processes on different machines. They use network protocols, such as TCP and UDP. Network sockets are more versatile than Unix domain sockets, as they can be used to communicate with processes on any machine that is connected to the network. They are commonly used for client-server communication, such as web servers and client applications.

In Golang, network sockets are created using the `net.Dial` or `net.Listen` functions, with a network type such as `TCP` or UDP. For example, the following code creates a TCP socket and listens for incoming connections:

```go
// Create a TCP socket and listen for incoming connections.
socket, err := net.Listen("tcp", ":8000")
if err != nil {
    panic(err)
}
```

### Basic Profiling from a client perspective

Client pprof for network socket

```text
Type: cpu
...
(pprof) list main.main
...
ROUTINE ======================== main.main in /Users/douglasmakey/go/src/github.com/douglasmakey/go-sockets-uds-network-pprof/server_echo_network_socket/client/main.go
         0      530ms (flat, cum) 70.67% of Total
         .          .     16:
         .          .     17:   pprof.StartCPUProfile(f)
         .          .     18:   defer pprof.StopCPUProfile()
         .          .     19:
         .          .     20:   for i := 0; i < 10000; i++ {
         .      390ms     21:          conn, err := net.Dial("tcp", "localhost:3000")
         .          .     22:          if err != nil {
         .          .     23:                  log.Fatal(err)
         .          .     24:          }
         .          .     25:
         .          .     26:          msg := "Hello"
         .       40ms     27:          if _, err := conn.Write([]byte(msg)); err != nil {
         .          .     28:                  log.Fatal(err)
         .          .     29:          }
         .          .     30:
         .          .     31:          b := make([]byte, len(msg))
         .      100ms     32:          if _, err := conn.Read(b); err != nil {
         .          .     33:                  log.Fatal(err)
         .          .     34:          }
         .          .     35:   }
         .          .     36:}
```

Client pprof for unix socket

```text
Type: cpu
...
(pprof) list main.main
...
ROUTINE ======================== main.main in /Users/douglasmakey/go/src/github.com/douglasmakey/go-sockets-uds-network-pprof/server_echo_unix_domain_socket/client/main.go
         0      210ms (flat, cum) 80.77% of Total
         .          .     16:
         .          .     17:   pprof.StartCPUProfile(f)
         .          .     18:   defer pprof.StopCPUProfile()
         .          .     19:
         .          .     20:   for i := 0; i < 10000; i++ {
         .      130ms     21:          conn, err := net.Dial("unix", "/tmp/echo.sock")
         .          .     22:          if err != nil {
         .          .     23:                  log.Fatal(err)
         .          .     24:          }
         .          .     25:
         .          .     26:          msg := "Hello"
         .       40ms     27:          if _, err := conn.Write([]byte(msg)); err != nil {
         .          .     28:                  log.Fatal(err)
         .          .     29:          }
         .          .     30:
         .          .     31:          b := make([]byte, len(msg))
         .       40ms     32:          if _, err := conn.Read(b); err != nil {
         .          .     33:                  log.Fatal(err)
         .          .     34:          }
         .          .     35:   }
         .          .     36:}
```

Things to notice in these basic profiles are:
- Open an unix socket is significantly faster than a network socket.
- Reading from unix socket is significantly faster than reading from a network socket.

[Github Repo](https://github.com/douglasmakey/go-sockets-uds-network-pprof)


### Use Unix domain socket with HTTP Server

To use a Unix domain socket with an HTTP server in Go, you can use the `server.Serve` function and specify the `net.Listener` to listen on.

```go
package main
import (...)

const socketPath = "/tmp/httpecho.sock"
func main() {
	// Create a Unix domain socket and listen for incoming connections.
	socket, err := net.Listen("unix", socketPath)
	if err != nil {
		panic(err)
	}

	// Cleanup the sockfile.
	c := make(chan os.Signal, 1)
	signal.Notify(c, os.Interrupt, syscall.SIGTERM)
	go func() {
		<-c
		os.Remove(socketPath)
		os.Exit(1)
	}()

	m := http.NewServeMux()
	m.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		w.Write([]byte("Hello kung fu developer! "))
	})

	server := http.Server{
		Handler: m,
	}

	if err := server.Serve(socket); err != nil {
		log.Fatal(err)
	}
}

```

Test it

```bash
$ curl -s -N --unix-socket /tmp/httpecho.sock http://localhost/
Hello kung fu developer!
```

Using an HTTP server with a Unix domain socket in Go can provide several advantages, which were named in this article, such as improved security, performance, ease of use, and interoperability. These advantages can make the HTTP server implementation more efficient, reliable, and scalable for processes that have to communicate between the same machine.

### What about security?

Unix domain sockets and network sockets have different security characteristics. In general, Unix domain sockets are considered to be more secure than network sockets, as they are not exposed to the network and are only accessible to processes on the same machine.

One of the main security features of Unix domain sockets is that they use file system permissions to control access. The socket file is created with a specific user and group and can only be accessed by processes that have the correct permissions for that user and group. This means that only authorized processes can connect to the socket and exchange data.

In contrast, network sockets are exposed to the network and are accessible to any machine that is connected to the network. This makes them vulnerable to attacks from malicious actors, such as hackers and malware. Network sockets use network protocols, such as TCP/UDP. These protocols have their own security mechanisms, such as encryption and authentication. However, these mechanisms are not always sufficient to protect against all types of attacks, and network sockets can still be compromised.

### Which to choose?

When deciding between Unix domain sockets and network sockets in Golang, it is important to consider the requirements and constraints of the application. Unix domain sockets are faster and more efficient but are limited to communication between processes on the same machine. Network sockets are more versatile but require more overhead and are subject to network latency and reliability issues. In general, Unix domain sockets are suitable for IPC and communication between services on the same machine, while network sockets are suitable for client-server communication over the network.

You should choose a Unix domain socket over a network socket when you need to communicate between processes on the same host. Unix domain sockets provide a secure and efficient communication channel between processes on the same host and are suitable for scenarios where the processes need to exchange data frequently or in real time.

For example, imagine you have two containers in a K8s pod, which must communicate with each other as quickly as possible!


![Container commmunications](/img/container-com.png)



In conclusion, Unix domain sockets and network sockets are two important types of sockets in Golang and are used for different purposes and scenarios. Understanding the differences and trade-offs between these types of sockets is essential for designing and implementing efficient and reliable networked applications in Golang.

Remember, the communication between processes with a unix socket is handled solely by the kernel, while in a network socket, the communication involves the kernel, the network stack, and the network hardware.