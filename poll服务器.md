### server
```c
#include <poll.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <unistd.h>
#include <stdlib.h>
#include <stdio.h>
#include <string.h>
typedef struct pollfd pollfd;
void Init(pollfd* fd_list, int size) {
for (int i = 0; i < size; ++i) {
fd_list[i].fd = -1;
fd_list[i].events = 0;
fd_list[i].revents = 0;
}
}
void Add(int fd, pollfd* fd_list, int size) {
for (int i = 0; i < size; ++i) {
if (fd_list[i].fd == -1) {
fd_list[i].fd = fd;
fd_list[i].events = POLLIN;
break;
}
}
}
int main(int argc, char* argv[]) {
if (argc != 3) {
printf("usage ./server [ip] [port]\n");
return 1;
}
struct sockaddr_in addr;
addr.sin_family = AF_INET;
addr.sin_addr.s_addr = inet_addr(argv[1]);
addr.sin_port = htons(atoi(argv[2]));
int listen_fd = socket(AF_INET, SOCK_STREAM, 0);
if (listen_fd < 0) {
perror("socket");
return 1;
}
int ret = bind(listen_fd, (struct sockaddr*)&addr, sizeof(addr));
if (ret < 0) {
perror("bind");
return 1;
}
ret = listen(listen_fd, 5);
if (ret < 0) {
perror("listen");
return 1;
}
pollfd fd_list[1024];
Init(fd_list, sizeof(fd_list) / sizeof(pollfd));
Add(listen_fd, fd_list, sizeof(fd_list) / sizeof(pollfd));
for (;;) {
int ret = poll(fd_list, sizeof(fd_list) / sizeof(pollfd), 1000);
if (ret < 0) {
perror("poll");
continue;
}
if (ret == 0) {
printf("poll timeout\n");
continue;
}
for (size_t i = 0; i < sizeof(fd_list) / sizeof(pollfd); ++i) {
if (fd_list[i].fd == -1) {
continue;
}
if (!(fd_list[i].revents & POLLIN)) {
continue;
}
if (fd_list[i].fd == listen_fd) {
// 处理 listen_fd 的情况.
struct sockaddr_in client_addr;
socklen_t len = sizeof(client_addr);
int connect_fd = accept(listen_fd, (struct sockaddr*)&client_addr, &len);
if (connect_fd < 0) {
perror("accept");
continue;
}
Add(connect_fd, fd_list, sizeof(fd_list) / sizeof(pollfd));
} else {
// 处理 connect_fd 的情况.
char buf[1024] = {0};
ssize_t read_size = read(fd_list[i].fd, buf, sizeof(buf) - 1);
if (read_size < 0) {
perror("read");
continue;
}
if (read_size == 0) {
printf("client say: goodbye\n");
close(fd_list[i].fd);
fd_list[i].fd = -1;
}
printf("client say: %s\n", buf);
write(fd_list[i].fd, buf, strlen(buf));
}
}
}
return 0;
}
```

### client
```c
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
void Usage() {
printf("usage: ./client [ip] [port]\n");
}
int main(int argc, char* argv[]) {
if (argc != 3) {
Usage();
return 1;
}
struct sockaddr_in addr;
addr.sin_family = AF_INET;
addr.sin_addr.s_addr = inet_addr(argv[1]);
addr.sin_port = htons(atoi(argv[2]));
int fd = socket(AF_INET, SOCK_STREAM, 0);
if (fd < 0) {
perror("socket");
return 1;
}

int ret = connect(fd, (struct sockaddr*)&addr, sizeof(addr));
if (ret < 0) {
perror("connect");
return 1;
}
for (;;) {
printf("> ");
fflush(stdout);
char buf[1024] = {0};
read(0, buf, sizeof(buf) - 1);
ssize_t write_size = write(fd, buf, strlen(buf));
if (write_size < 0) {
perror("write");
continue;
}
ssize_t read_size = read(fd, buf, sizeof(buf) - 1);
if (read_size < 0) {
perror("read");
continue;
}
if (read_size == 0) {
printf("server close\n");
break;
}
printf("server say: %s", buf);
}
close(fd);
return 0;
}
```
