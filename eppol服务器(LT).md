### server
```c
#include <sys/epoll.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
void ProcessConnect(int listen_fd, int epoll_fd) {
struct sockaddr_in client_addr;
socklen_t len = sizeof(client_addr);
int connect_fd = accept(listen_fd, (struct sockaddr*)&client_addr, &len);
if (connect_fd < 0) {
perror("accept");
return;
}
printf("client %s:%d connect\n", inet_ntoa(client_addr.sin_addr), ntohs(client_a
epoll的使⽤用场景
epoll中的惊群问题(选学)
epoll⽰示例: epoll服务器
比特科技
ddr.sin_port));
struct epoll_event ev;
ev.data.fd = connect_fd;
ev.events = EPOLLIN;
int ret = epoll_ctl(epoll_fd, EPOLL_CTL_ADD, connect_fd, &ev);
if (ret < 0) {
perror("epoll_ctl");
return;
}
return;
}
void ProcessRequest(int connect_fd, int epoll_fd) {
char buf[1024] = {0};
ssize_t read_size = read(connect_fd, buf, sizeof(buf) - 1);
if (read_size < 0) {
perror("read");
return;
}
if (read_size == 0) {
close(connect_fd);
epoll_ctl(epoll_fd, EPOLL_CTL_DEL, connect_fd, NULL);
printf("client say: goodbye\n");
return;
}
printf("client say: %s", buf);
write(connect_fd, buf, strlen(buf));
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
比特科技
}
ret = listen(listen_fd, 5);
if (ret < 0) {
perror("listen");
return 1;
}
int epoll_fd = epoll_create(10);
if (epoll_fd < 0) {
perror("epoll_create");
return 1;
}
struct epoll_event event;
event.events = EPOLLIN;
event.data.fd = listen_fd;
ret = epoll_ctl(epoll_fd, EPOLL_CTL_ADD, listen_fd, &event);
if (ret < 0) {
perror("epoll_ctl");
return 1;
}
for (;;) {
struct epoll_event events[10];
int size = epoll_wait(epoll_fd, events, sizeof(events) / sizeof(events[0]), -1
);
if (size < 0) {
perror("epoll_wait");
continue;
}
if (size == 0) {
printf("epoll timeout\n");
continue;
}
for (int i = 0; i < size; ++i) {
if (!(events[i].events & EPOLLIN)) {
continue;
}
if (events[i].data.fd == listen_fd) {
// 处理 listen_fd
ProcessConnect(listen_fd, epoll_fd);
} else {
// 处理 connect_fd
ProcessRequest(events[i].data.fd, epoll_fd);
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
比特科技
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
