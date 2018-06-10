### server
```c
#include <sys/epoll.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <errno.h>
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
void SetNoBlock(int fd) {
int fl = fcntl(fd, F_GETFL);
if (fl < 0) {
perror("fcntl");
return;
}
fcntl(fd, F_SETFL, fl | O_NONBLOCK);
}
void ProcessConnect(int listen_fd, int epoll_fd) {
struct sockaddr_in client_addr;
socklen_t len = sizeof(client_addr);
int connect_fd = accept(listen_fd, (struct sockaddr*)&client_addr, &len);
if (connect_fd < 0) {
perror("accept");
return;
}
printf("client %s:%d connect\n", inet_ntoa(client_addr.sin_addr), ntohs(client_a
ddr.sin_port));
SetNoBlock(connect_fd);
struct epoll_event ev;
ev.data.fd = connect_fd;
ev.events = EPOLLIN | EPOLLET;
int ret = epoll_ctl(epoll_fd, EPOLL_CTL_ADD, connect_fd, &ev);
if (ret < 0) {
perror("epoll_ctl");
return;
}
return;
}
ssize_t NoBlockRead(int fd, char* buf, int size) {
(void) size;
ssize_t total_size = 0;
for (;;) {
ssize_t cur_size = read(fd, buf + total_size, 1024);
比特科技
total_size += cur_size;
if (cur_size < 1024 || errno == EAGAIN) {
break;
}
}
buf[total_size] = '\0';
return total_size;
}
void ProcessRequest(int connect_fd, int epoll_fd) {
char buf[1024] = {0};
ssize_t read_size = NoBlockRead(connect_fd, buf, sizeof(buf) - 1);
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
}
ret = listen(listen_fd, 5);
if (ret < 0) {
比特科技
perror("listen");
return 1;
}
int epoll_fd = epoll_create(10);
if (epoll_fd < 0) {
perror("epoll_create");
return 1;
}
SetNoBlock(listen_fd);
struct epoll_event event;
event.events = EPOLLIN | EPOLLET;
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
