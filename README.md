# cygwinTinyWebServer
A small networking program that develops a functioning web server called TINY, that uses process controls, Unix I/O, the sockets interface and HTTP, though lacks standard security protocols. The web server, serves both static and dynamic content to real web browsers. 

![Web Server](https://raw.githubusercontent.com/kiddjsh/cygwinTinyWebServer/main/images/rainingMarquee.PNG)

![directoryFiles](https://raw.githubusercontent.com/kiddjsh/cygwinTinyWebServer/main/images/directoryFiles.PNG)

![cygwinExecution](https://raw.githubusercontent.com/kiddjsh/cygwinTinyWebServer/main/images/cygwinExecution.PNG)


# The TINY main Routine

A small networking program that develops a functioning web server called TINY, that uses process controls, Unix I/O, the sockets interface and HTTP, though lacks standard security protocols. The web server, serves both a static and dynamic content to real web browsers. 

```c
/*
 * tiny.c - A simple iterative HTTP/1.0 Web Server that uses the 
 *          GET method to serve static and dynamic content.
 */
 
void doit(int fd);
void read_requesthdrs(rio_t *rp);
int parse_uri(char *uri, char *filename, char *cgiargs);
void serve_static(int fd, char *filename, int filesize);
void get_filetype(char *filename, char *filetype);
void serve_dynamic(int fd, char *filename, char *cgiargs);
void clienterror(int fd, char *cause, char *errnum, 
		 char *shortmsg, char *longmsg);

int main(int argc, char **argv) 
{
    int listenfd, connfd, port, clientlen;
    struct sockaddr_in clientaddr;

    /* Check command line args */
    if (argc != 2) {
	fprintf(stderr, "usage: %s <port>\n", argv[0]);
	exit(1);
    }
    port = atoi(argv[1]);

    listenfd = open_listenfd(port);
    while (1) {
	clientlen = sizeof(clientaddr);
	connfd = accept(listenfd, (SA *)&clientaddr, &clientlen);
	doit(connfd);
	close(connfd);
    }
}
```

# The doit Function

The doit function handles one HTTP transaction, reading and parseing the request line using the rio_ readlineb function to read the request line.

```c
/*
 * doit - Handles one HTTP request/response transaction
 */
 
void doit(int fd) 
{
    int is_static;
    struct stat sbuf;
    char buf[MAXLINE], method[MAXLINE], uri[MAXLINE], version[MAXLINE];
    char filename[MAXLINE], cgiargs[MAXLINE];
    rio_t rio;
  
    /* Read request line and headers */
    rio_readinitb(&rio, fd);
    rio_readlineb(&rio, buf, MAXLINE);
    sscanf(buf, "%s %s %s", method, uri, version);
    if (strcasecmp(method, "GET")) { 
       clienterror(fd, method, "501", "Not Implemented",
                "Tiny does not implement this method");
        return;
    }
    read_requesthdrs(&rio);

    /* Parse URI from GET request */
    is_static = parse_uri(uri, filename, cgiargs);
    if (stat(filename, &sbuf) < 0) {
	clienterror(fd, filename, "404", "Not found",
		    "Tiny couldn't find this file");
	return;
    }

    if (is_static) { /* Serve static content */
	if (!(S_ISREG(sbuf.st_mode)) || !(S_IRUSR & sbuf.st_mode)) {
	    clienterror(fd, filename, "403", "Forbidden",
			"Tiny couldn't read the file");
	    return;
	}
	serve_static(fd, filename, sbuf.st_size);
    }
    else { /* Serve dynamic content */
	if (!(S_ISREG(sbuf.st_mode)) || !(S_IXUSR & sbuf.st_mode)) {
	    clienterror(fd, filename, "403", "Forbidden",
			"Tiny couldn't run the CGI program");
	    return;
	}
	serve_dynamic(fd, filename, cgiargs);
    }
}
```
