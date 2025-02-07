# epoll echo server example

- epoll.c includes both server-end and client-end code.
- please refer to [epoll](http://man7.org/linux/man-pages/man7/epoll.7.html) for details.

# Build a Simple Echo Service with Epoll

## Features

### Original
 - When typing messages and send them in the clients, the server can receive the same messages, log, and return them.
 - Edge-triggered `epoll` file descriptor is used to monitor I/O events of sockets for the connections of clients.

### Added
 - Connections from different hosts
 - Add standard input into epoll listened-on events
 - Type `exit` can close file descriptors correctly and exit program
 - Type `%date%` and `%time%` on clients to get the date and time of server

## Build Executable

### Compile from Source Code

```sh=
git clone https://github.com/Axisflow/epoll-example.git
cd epoll-example

gcc -o epoll epoll.c
```

Usage: `epoll [-cs] [-a address] [-p port]`

### Run as Server for Example

```sh=
./epoll -s -a 0.0.0.0 -p 9090
```

### Run as Client for Example

```sh=
./epoll -c -a 127.0.0.1 -p 9090
```

## Run Server in Docker

##Todo##
Demo Server is now available on: `125.229.40.240:9090`.

## Edited Code Explaination

### Include headers

```c=
#include <sys/types.h>
#include <sys/socket.h>
#include <netdb.h>
#include <string.h>
#include <unistd.h>
#include <fcntl.h>
#include <sys/epoll.h>
#include <errno.h>
#include <stdio.h>
#include <stdlib.h>
#include <time.h> // Add this to use the time function

// Add these to avoid the warning of implicit declaration
#include <arpa/inet.h>
#include <getopt.h>
```

### Define constant

```c=
#define DEFAULT_ADDR    INADDR_ANY // Server Address
#define DEFAULT_PORT    9090       // Server Port Number
#define MAX_CONN        16         // Maximum number of clients
#define MAX_EVENTS      32         // Maximum number of epoll listen-on events
#define BUF_SIZE        16         // Maximum size of server I/O buffer
#define MAX_LINE        256        // Maximum size of client I/O buffer

// set the default address and port number
in_addr_t address = DEFAULT_ADDR;
unsigned short port = DEFAULT_PORT;
```
### Entry point

We decide to run server or client, and process custom address/port number here. `char getopt(int argc, char *argv[], char options[])` returns the current option and `optarg` refers the value of it.

```c=
void server_run();
void client_run();

// Determine run as server or client
int main(int argc, char *argv[]) {
    int opt;
    char role = 's'; // Run as a server default

    // To get the argument (not a standard UNIX function)
    while ((opt = getopt(argc, argv, "csa:p:")) != -1) {
        switch (opt) {
            case 'c':
                role = 'c'; // Run as a client
                break;
            case 's':
                break;
            case 'a':
                address = inet_addr(optarg); // Convert the address from text to binary
                printf("address: %s -> %x\n", optarg, address);
                if (address == INADDR_NONE) { // Check if the address is valid
                    perror("Cannot convert the address\n");
                    return EXIT_FAILURE;
                }

                break;
            case 'p':
                port = atoi(optarg); // Convert the port number from text to numeric
                printf("port: %u\n", port);
                if (port == 0) { // Check if the port number is valid
                    perror("Cannot convert the port number\n");
                    return EXIT_FAILURE;
                }

                break;
            default: // Print usage when being given the error arguments
                printf("usage: %s [-cs] [-a address] [-p port]\n", argv[0]);
                return EXIT_FAILURE;
        }
    }

    if (role == 's') {
        server_run();
    } else {
        client_run();
    }

    return EXIT_SUCCESS;
}
```

### Register a file descriptor to epoll instance

We construct an `epoll_event` structure to store the file descriptor and events needed to be listened on, and use `epoll_ctl` to register it in the epoll instance.

```c=
static void epoll_ctl_add(int epfd, int fd, uint32_t events) {
    // Create an epoll event struct
    // events: a bit mask specifying the events the application is interested in for the file descriptor
    // data: a union that can hold a file descriptor or a pointer
    struct epoll_event ev;
    ev.events = events;
    ev.data.fd = fd;

    // Add the file descriptor to the epoll instance
    if (epoll_ctl(epfd, EPOLL_CTL_ADD, fd, &ev) == -1) {
        perror("epoll_ctl()\n");
        exit(EXIT_FAILURE);
    }
}
```

### Configure server information

Help construct the structure `sockaddr_in` of the server information where to be listened on new connections for server and where to connect it for client.

```c=
static void set_sockaddr(struct sockaddr_in *addr) {
    // Not a standard UNIX function
    // Set the memory of addr to 0s (same as memset(addr, 0, sizeof(struct sockaddr_in)))
    // This is to avoid the garbage value in the memory
    bzero((char *)addr, sizeof(struct sockaddr_in));

    addr->sin_family = AF_INET; // IPv4
    addr->sin_addr.s_addr = address;
    addr->sin_port = htons(port); // Convert the port number with network byte order (big-endian)
}
```

### Set file descriptor to non-blocking mode

Use `fcntl(int fd, F_GETFL, NULL)` to get now cofigurations of the specific file descriptor, then use `fcntl(int fd, F_SETFL, flags | O_NONBLOCK)` to make its I/O non-block.

```c=
static int setnonblocking(int sockfd) {
    // Set the file descriptor to non-blocking mode (combine current flags with O_NONBLOCK)
    // F_GETFL: Get the file status flags and file access modes
    // F_SETFL: Set the file status flags to the value specified by arg
    // O_NONBLOCK: Non-blocking mode
    if (fcntl(sockfd, F_SETFL, fcntl(sockfd, F_GETFL, 0) | O_NONBLOCK) == -1) {
        return -1;
    }
    
    return 0;
}
```

### Server mode

```c=
void server_run() {
    int i;
    int n;
    int epfd;
    int nfds;
    int listen_sock;
    int conn_sock;
    int socklen;
    char buf[BUF_SIZE];
    struct sockaddr_in srv_addr;
    struct sockaddr_in cli_addr;
    struct epoll_event events[MAX_EVENTS];

    time_t t;
    struct tm *tm;

    // Create a socket using TCP protocol in IPv4 domain & get the file descriptor
    if((listen_sock = socket(AF_INET, SOCK_STREAM, 0)) < 0) {
        perror("[!] Cannot create socket file descriptor\n");
        exit(EXIT_FAILURE);
    }

    // Set the socket address & port information for the server
    set_sockaddr(&srv_addr);
    
    // Give the socket fd the local address with the size of the info struct
    if(bind(listen_sock, (struct sockaddr *)&srv_addr, sizeof(srv_addr)) < 0) {
        perror("[!] Cannot bind the socket\n");
        exit(EXIT_FAILURE);
    }

    // Set the file descriptor to non-blocking mode because of using epoll edge-triggered mode
    if(setnonblocking(listen_sock) < 0) {
        perror("[!] Cannot set the socket to non-blocking mode\n");
        exit(EXIT_FAILURE);
    }

    // Set the socket to listen mode with at most 16 connections
    if(listen(listen_sock, MAX_CONN) < 0) {
        perror("[!] Cannot listen on the socket\n");
        exit(EXIT_FAILURE);
    }

    // Create an epoll file descriptor (size parameter is deprecated but still required to be larger than 0)
    if((epfd = epoll_create(1)) == -1) {
        perror("[!] Cannot create epoll file descriptor\n");
        exit(EXIT_FAILURE);
    }

    // Add the listen socket to the events queue in epoll file descriptor
    // EPOLLIN: The associated file is available for read(2) operations
    // EPOLLOUT: The associated file is available for write(2) operations
    // EPOLLET: Sets the Edge Triggered behavior for the associated file descriptor
    epoll_ctl_add(epfd, listen_sock, EPOLLIN | EPOLLOUT | EPOLLET);\

    // Add stdin to the events queue in epoll file descriptor
    epoll_ctl_add(epfd, STDIN_FILENO, EPOLLIN | EPOLLET);

    // The size of the client address info struct
    socklen = sizeof(cli_addr);

    // Start to handle the events
    for (;;) {
        // Wait for events on an epoll instance
        // nfds: the number of file descriptors ready for the requested I/O operations (triggered events)
        // events: the buffer where the triggered events are stored
        nfds = epoll_wait(epfd, events, MAX_EVENTS, -1);
        for (i = 0; i < nfds; i++) {
            if (events[i].data.fd == listen_sock) { // The listen socket is ready for read
                /* handle new connection */
                // Accept the incoming connection and get the file descriptor of the new client socket fd
                conn_sock = accept(listen_sock, (struct sockaddr *)&cli_addr, &socklen);

                // Convert the IP address from binary to text
                inet_ntop(AF_INET, (char *)&(cli_addr.sin_addr), buf, sizeof(cli_addr));
                printf("[+] connected with %s:%d\n", buf, ntohs(cli_addr.sin_port));

                setnonblocking(conn_sock); // Set the client socket to non-blocking mode

                // Add the client socket to the events queue
                // EPOLLRDHUP: Stream socket peer closed connection, or shut down writing half of connection
                // EPOLLHUP: Hang up happened on the associated file descriptor
                epoll_ctl_add(epfd, conn_sock, EPOLLIN | EPOLLET | EPOLLRDHUP | EPOLLHUP);
            } else if (events[i].data.fd == STDIN_FILENO) { // The stdin is ready for read
                /* handle stdin */
                for (;;) {
                    bzero(buf, sizeof(buf)); // Set the buffer to 0s

                    // Read the data from the stdin to the buffer
                    // EAGAIN: Try read again because of resource is temporarily unavailable (non-blocking mode)
                    if ((n = read(STDIN_FILENO, buf, sizeof(buf))) <= 0 /* || errno == EAGAIN */ ) {
                        break;
                    } else {
                        if(strcmp(buf, "exit\n") == 0) { // Check if the input is "exit"
                            close(listen_sock); // Close the listen socket
                            close(epfd); // Close the epoll file descriptor
                            exit(EXIT_SUCCESS);
                        } else {
                            printf("[+] stdin (%d bytes): %s\n", n, buf);
                        }
                    }
                }
            } else if (events[i].events & EPOLLIN) { // The client socket is ready for read
                /* handle EPOLLIN event */
                for (;;) {
                    bzero(buf, sizeof(buf)); // Set the buffer to 0s
                    
                    // Read the data from the client socket to the buffer
                    if ((n = read(events[i].data.fd, buf, sizeof(buf))) <= 0 /* || errno == EAGAIN */ ) {
                        break;
                    } else {
                        printf("[+] data (%d bytes): %s", n, buf);

                        /* Echo function */
                        if(strcmp(buf, "%date%") == 0) { // Check if the input is "%%date%%"
                            t = time(NULL);
                            tm = localtime(&t);
                            strftime(buf, sizeof(buf), "%x", tm);
                        } else if(strcmp(buf, "%time%") == 0) { // Check if the input is "%%time%%"
                            t = time(NULL);
                            tm = localtime(&t);
                            strftime(buf, sizeof(buf), "%X", tm);
                        }

                        // Write the data back to the client socket
                        if(write(events[i].data.fd, buf, strlen(buf)) < 0) {
                            perror("[!] write()");
                        }

                        printf(" -> %s\n", buf);
                    }
                }
            } else {
                printf("[+] unexpected\n");
            }
            
            /* check if the connection is closing */
            if (events[i].events & (EPOLLRDHUP | EPOLLHUP)) {
                printf("[+] connection closed\n");

                // Remove the file descriptor from the events queue
                epoll_ctl(epfd, EPOLL_CTL_DEL, events[i].data.fd, NULL);

                // Close the file descriptor of the client socket
                close(events[i].data.fd);
            }
        }
    }
}
```

### Client mode

```c=
void client_run() {
    int n;
    int c;
    int sockfd;
    char buf[MAX_LINE];
    struct sockaddr_in srv_addr;

    // Create a socket using TCP protocol in IPv4 domain & get the file descriptor
    sockfd = socket(AF_INET, SOCK_STREAM, 0);

    // Set the socket address & port information for the server
    set_sockaddr(&srv_addr);

    // Connect to the server
    if (connect(sockfd, (struct sockaddr *)&srv_addr, sizeof(srv_addr)) < 0) {
        perror("cannot connect to the server\n");
        exit(1);
    }

    // Start the communication with the server
    for (;;) {
        printf("input: ");
        fgets(buf, sizeof(buf), stdin); // Get the input from the user
        c = strlen(buf) - 1;
        buf[c] = '\0'; // Replace the newline character with the null character

        if (strcmp(buf, "exit") == 0) { // Check if the input is "exit"
            break;
        }

        write(sockfd, buf, c + 1); // Send the input to the server

        bzero(buf, sizeof(buf)); // Reset the buffer

        // Receive the data from the server
        while (errno != EAGAIN && (n = read(sockfd, buf, sizeof(buf))) > 0) {
            printf("echo: %s\n", buf);
            bzero(buf, sizeof(buf));

            if ((c -= n) <= 0) { // Check if the data is fully received
                break;
            }
        }
    }

    // Close the file descriptor
    close(sockfd);
}
```

## Example

Open 5 terminals: `t1`, `t2`, `t3`, `t4` and `t5`. `t1` is used for starting echo server, `t2` and `t3` are used for starting clients, `t4` and `t5` are used for monitoring the connections.

### t1

```shell
manager@AF-Workstation:/mnt/d/docker-data$ ./epoll -s -p 9090
port: 9090
[+] connected with 127.0.0.1:44733
[+] data (12 bytes): hello world -> hello world
[+] data (7 bytes): %time% -> 16:31:29
[+] connected with 114.33.245.20:33010
[+] data (7 bytes): %time% -> 16:31:44
[+] data (7 bytes): %date% -> 02/07/25
[+] data (16 bytes): Hello! I'm Axisf -> Hello! I'm Axisf
[+] data (16 bytes): low. Nice to mee -> low. Nice to mee
[+] data (7 bytes): t you~ -> t you~
[+] connection closed
[+] connection closed
exit
```

### t2

```shell
manager@AF-Workstation:/mnt/d/docker-data$ ./epoll -c -p 9090
port: 9090
input: hello world
echo: hello world
input: %time%
echo: 16:31:29
input: exit
```

### t3

```shell
user@Axisflow-ROG:~/epoll-example$ ./epoll -c -a 125.229.40.240 -p 9090
address: 125.229.40.240 -> f028e57d
port: 9090
input: %time%
echo: 16:31:44
input: %date%
echo: 02/07/25
input: Hello! I'm Axisflow. Nice to meet you~ 
echo: Hello! I'm Axisf
echo: low. Nice to meet you~
input: exit
```

### t4

```shell
manager@AF-Workstation:~$ netstat --inet -n | grep 9090
tcp        0      0 127.0.0.1:9090          127.0.0.1:44733         ESTABLISHED
tcp        0      0 127.0.0.1:44733         127.0.0.1:9090          ESTABLISHED
tcp        0      0 125.229.40.240:9090     114.33.245.20:33010     ESTABLISHED
```

### t5

```shell
user@Axisflow-ROG:~/epoll-example$ netstat --inet -n | grep 9090
tcp        0      0 172.22.38.15:45060      125.229.40.240:9090     ESTABLISHED
```
