---
layout: post
title: "Understanding Remote Buffer Overflows in a Controlled Lab"
permalink: /writeups/buffer-overflow/
date: 2026-05-14 15:20:46 -0400
categories: exploit-development binary-exploitation homelab
---

# Introduction

In this writeup, I explore classic stack-based buffer overflows within a controlled homelab environment created specifically for exploit development research and defensive learning purposes.

This project focuses on:
- Understanding stack memory layout
- Controlling instruction flow
- Analyzing crashes with a debugger
- Examining modern exploit mitigations
- Learning how insecure memory handling leads to vulnerabilities

---

# Lab Environment

## Target System
- OS: Parrot OS
- Architecture: x86
- Compiler: gcc
- Protections Enabled/Disabled: Disabled

## Tools Used
- GDB
- GCC
- Python
- Custom test binaries

---

# Vulnerable Program

```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <unistd.h>

void vulnerable_function(char *input) {
    char buffer[64];
    printf("RSP: %p\n", __builtin_frame_address(0));  // ADD THIS
    strcpy(buffer, input);
    printf("Buffer content: %s\n", buffer);
}

int main() {
    int port = 4444;
    int sockfd, newsockfd;
    struct sockaddr_in server_addr, client_addr;
    socklen_t client_len = sizeof(client_addr);
    char buffer[256];

    // Create socket
    sockfd = socket(AF_INET, SOCK_STREAM, 0);
    if (sockfd < 0) {
        perror("socket");
        exit(1);
    }

    // Set up server address
    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(port);
    server_addr.sin_addr.s_addr = INADDR_ANY;

    // Bind socket to address
    if (bind(sockfd, (struct sockaddr *)&server_addr, sizeof(server_addr)) < 0) {
        perror("bind");
        exit(1);
    }

    // Listen for connections
    if (listen(sockfd, 1) < 0) {
        perror("listen");
        exit(1);
    }

    printf("Server listening on port %d...\n", port);

    // Accept connection
    newsockfd = accept(sockfd, (struct sockaddr *)&client_addr, &client_len);
    if (newsockfd < 0) {
        perror("accept");
        exit(1);
    }

    printf("Accepted connection from %s:%d\n", inet_ntoa(client_addr.sin_addr), ntohs(client_addr.sin_port));

    // Receive data from client
    recv(newsockfd, buffer, 256, 0);
    vulnerable_function(buffer);

    // Close sockets
    close(newsockfd);
    close(sockfd);

    return 0;
}
```

# Why is this code vulnerable?

- The program uses a stack-based buffer to store user input, which can be overwritten by an attacker.
- The code is vulnerable due to the lack of input sanitization and the use of the `strcpy` function.
- The program does not properly handle the return value of `strcpy`, which can lead to buffer overflows.
- Due to the use of sockets in order to create a server, the program is vulnerable to remote buffer overflows.

---

# Exploitation Process

1. Compile the vulnerable program using GCC.
2. Run the compiled binary on the target system.
3. Connect to the server and send a payload that overflows the buffer.
Payload:
```
A * 100
```
4. Observe the program's behavior and analyze the crash with GDB.
5. Using GNU Debugger (GDB), analyze the crash and identify the source of the vulnerability.
6. Once the return address is either found or calculated via RSP, create a payload that overflows the buffer (for this experiment, I created my own in assembly).
7. Create a python script that will send the payload to the server.
8. Run the python script and observe the program's behavior.
9. If the script doesn't work, the problem is either null bytes in the shellcode, miscalculated NOP sled, or the wrong return address.

---

# Screenshots

![](/assets/images/exploit.png)

---

# Conclusion

Buffer Overflows are a classic means of obtaining Remote Code Execution (or RCE). Despite being more prominent in the 1990's, they still come up every now and then.
