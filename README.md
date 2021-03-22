# cygwinTinyWebServer
A small networking program that develops a functioning web server called TINY, that uses process controls, Unix I/O, the sockets interface and HTTP, though lacks standard security protocols. The web server, serves both static and dynamic content to real web browsers. 

![Web Server](https://raw.githubusercontent.com/kiddjsh/cygwinTinyWebServer/main/images/rainingMarquee.PNG)


# My C Solution tiny.c

```c
//Tiny web server code
#include <stdlib.h>
#include <stdio.h>
#include <arpa/inet.h>
#include <netdb.h>
#include <errno.h>
#include <string.h>
#include <sys/stat.h>
#include <sys/mman.h>
#include <sys/types.h>
#include <fcntl.h>
#include <unistd.h>


#define MAXLINE 8192 //max text line length
#define MAXBUF   8192  /* max I/O buffer size */
typedef struct sockaddr SA; //socaddrdef
#define LISTENQ  1024 


#define RIO_BUFSIZE 8192
typedef struct {
    int rio_fd;                /* descriptor for this internal buf */
    int rio_cnt;               /* unread bytes in internal buf */
    char *rio_bufptr;          /* next unread byte in internal buf */
    char rio_buf[RIO_BUFSIZE]; /* internal buffer */
} rio_t;



//Wrapper for the unix read() function that 
//transfers min(n, rio_cnt) bytes from an internal buffer
//to a user buffer

static ssize_t rio_read(rio_t *rp, char *usrbuf, size_t n)
{
    int cnt;

    while (rp->rio_cnt <= 0) {  /* refill if buf is empty */
	rp->rio_cnt = read(rp->rio_fd, rp->rio_buf, 
			   sizeof(rp->rio_buf));
	if (rp->rio_cnt < 0) {
	    if (errno != EINTR) /* interrupted by sig handler return */
		return -1;
	}
	else if (rp->rio_cnt == 0)  /* EOF */
	    return 0;
	else 
	    rp->rio_bufptr = rp->rio_buf; /* reset buffer ptr */
    }

    /* Copy min(n, rp->rio_cnt) bytes from internal buf to user buf */
    cnt = n;          
    if (rp->rio_cnt < n)   
	cnt = rp->rio_cnt;
    memcpy(usrbuf, rp->rio_bufptr, cnt);
    rp->rio_bufptr += cnt;
    rp->rio_cnt -= cnt;
    return cnt;
}

//wrapper for the unix read() function that transfers min(n, rio_cnt) bytes
//from an internal buffer to a user buffer

ssize_t rio_writen(int fd, void *usrbuf, size_t n) 
{
    size_t nleft = n;
    ssize_t nwritten;
    char *bufp = usrbuf;

    while (nleft > 0) {
	if ((nwritten = write(fd, bufp, nleft)) <= 0) {
	    if (errno == EINTR)  /* interrupted by sig handler return */
		nwritten = 0;    /* and call write() again */
	    else
		return -1;       /* errorno set by write() */
	}
	nleft -= nwritten;
	bufp += nwritten;
    }
    return n;
}

//robustly read a text line (buffered)
ssize_t rio_readlineb(rio_t *rp, void *usrbuf, size_t maxlen) 
{
    int n, rc;
    char c, *bufp = usrbuf;

    for (n = 1; n < maxlen; n++) { 
	if ((rc = rio_read(rp, &c, 1)) == 1) {
	    *bufp++ = c;
	    if (c == '\n')
		break;
	} else if (rc == 0) {
	    if (n == 1)
		return 0; /* EOF, no data read */
	    else
		break;    /* EOF, some data was read */
	} else
	    return -1;	  /* error */
    }
    *bufp = 0;
    return n;
}

//Associate a descriptor with a read buffer and reset buffer

void rio_readinitb(rio_t *rp, int fd) 
{
    rp->rio_fd = fd;  
    rp->rio_cnt = 0;  
    rp->rio_bufptr = rp->rio_buf;
}

//open connection to server at <hostname, port>
//and return a socket descriptor ready for reading and writing
int open_clientfd(char *hostname, int port) 
{
    int clientfd;
    struct hostent *hp;
    struct sockaddr_in serveraddr;

    if ((clientfd = socket(AF_INET, SOCK_STREAM, 0)) < 0)
	return -1; /* check errno for cause of error */

    /* Fill in the server's IP address and port */
    if ((hp = gethostbyname(hostname)) == NULL)
	return -2; /* check h_errno for cause of error */
    bzero((char *) &serveraddr, sizeof(serveraddr));
    serveraddr.sin_family = AF_INET;
    bcopy((char *)hp->h_addr_list[0], 
	  (char *)&serveraddr.sin_addr.s_addr, hp->h_length);
    serveraddr.sin_port = htons(port);

    /* Establish a connection with the server */
    if (connect(clientfd, (SA *) &serveraddr, sizeof(serveraddr)) < 0)
	return -1;
    return clientfd;
}

//Open and return a listening socket on port

int open_listenfd(int port) 
{
    int listenfd, optval=1;
    struct sockaddr_in serveraddr;
  
    /* Create a socket descriptor */
    if ((listenfd = socket(AF_INET, SOCK_STREAM, 0)) < 0)
	return -1;
 
    /* Eliminates "Address already in use" error from bind. */
    if (setsockopt(listenfd, SOL_SOCKET, SO_REUSEADDR, 
		   (const void *)&optval , sizeof(int)) < 0)
	return -1;

    /* Listenfd will be an endpoint for all requests to port
       on any IP address for this host */
    bzero((char *) &serveraddr, sizeof(serveraddr));
    serveraddr.sin_family = AF_INET; 
    serveraddr.sin_addr.s_addr = htonl(INADDR_ANY); 
    serveraddr.sin_port = htons((unsigned short)port); 
    if (bind(listenfd, (SA *)&serveraddr, sizeof(serveraddr)) < 0)
	return -1;

    /* Make it a listening socket ready to accept connection requests */
    if (listen(listenfd, LISTENQ) < 0)
	return -1;
    return listenfd;
}

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
/* $end tinymain */

/*
 * doit - handle one HTTP request/response transaction
 */
/* $begin doit */
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
/* $end doit */

/*
 * read_requesthdrs - read and parse HTTP request headers
 */
/* $begin read_requesthdrs */
void read_requesthdrs(rio_t *rp) 
{
    char buf[MAXLINE];

    rio_readlineb(rp, buf, MAXLINE);
    printf("%s", buf);
    while(strcmp(buf, "\r\n")) {
	rio_readlineb(rp, buf, MAXLINE);
	printf("%s", buf);
    }
    return;
}
/* $end read_requesthdrs */

/*
 * parse_uri - parse URI into filename and CGI args
 *             return 0 if dynamic content, 1 if static
 */
/* $begin parse_uri */
int parse_uri(char *uri, char *filename, char *cgiargs) 
{
    char *ptr;

    if (!strstr(uri, "cgi-bin")) {  /* Static content */
	strcpy(cgiargs, "");
	strcpy(filename, ".");
	strcat(filename, uri);
	if (uri[strlen(uri)-1] == '/')
	    strcat(filename, "home.html");
	return 1;
    }
    else {  /* Dynamic content */
	ptr = index(uri, '?');
	if (ptr) {
	    strcpy(cgiargs, ptr+1);
	    *ptr = '\0';
	}
	else 
	    strcpy(cgiargs, "");
	strcpy(filename, ".");
	strcat(filename, uri);
	return 0;
    }
}
/* $end parse_uri */

/*
 * serve_static - copy a file back to the client 
 */
/* $begin serve_static */
void serve_static(int fd, char *filename, int filesize) 
{
    int srcfd;
    char *srcp, filetype[MAXLINE], buf[MAXBUF];
 
    /* Send response headers to client */
    get_filetype(filename, filetype);
    sprintf(buf, "HTTP/1.0 200 OK\r\n");
    sprintf(buf, "%sServer: Tiny Web Server\r\n", buf);
    sprintf(buf, "%sContent-length: %d\r\n", buf, filesize);
    sprintf(buf, "%sContent-type: %s\r\n\r\n", buf, filetype);
    rio_writen(fd, buf, strlen(buf));

    /* Send response body to client */
    srcfd = open(filename, O_RDONLY, 0);
    srcp = mmap(0, filesize, PROT_READ, MAP_PRIVATE, srcfd, 0);
    close(srcfd);
    rio_writen(fd, srcp, filesize);
    munmap(srcp, filesize);
}

/*
 * get_filetype - derive file type from file name
 */
void get_filetype(char *filename, char *filetype) 
{
    if (strstr(filename, ".html"))
	strcpy(filetype, "text/html");
    else if (strstr(filename, ".gif"))
	strcpy(filetype, "image/gif");
    else if (strstr(filename, ".jpg"))
	strcpy(filetype, "image/jpeg");
    else
	strcpy(filetype, "text/plain");
}  
/* $end serve_static */

/*
 * serve_dynamic - run a CGI program on behalf of the client
 */
/* $begin serve_dynamic */
void serve_dynamic(int fd, char *filename, char *cgiargs) 
{
    char buf[MAXLINE], *emptylist[] = { NULL };

    /* Return first part of HTTP response */
    sprintf(buf, "HTTP/1.0 200 OK\r\n");
    rio_writen(fd, buf, strlen(buf));
    sprintf(buf, "Server: Tiny Web Server\r\n");
    rio_writen(fd, buf, strlen(buf));
  
    if (fork() == 0) { /* child */
	/* Real server would set all CGI vars here */
	setenv("QUERY_STRING", cgiargs, 1); 
	dup2(fd, STDOUT_FILENO);         /* Redirect stdout to client */
	execve(filename, emptylist, environ); /* Run CGI program */
    }
    wait(NULL); /* Parent waits for and reaps child */
}
/* $end serve_dynamic */

/*
 * clienterror - returns an error message to the client
 */
/* $begin clienterror */
void clienterror(int fd, char *cause, char *errnum, 
		 char *shortmsg, char *longmsg) 
{
    char buf[MAXLINE], body[MAXBUF];

    /* Build the HTTP response body */
    sprintf(body, "<html><title>Tiny Error</title>");
    sprintf(body, "%s<body bgcolor=""ffffff"">\r\n", body);
    sprintf(body, "%s%s: %s\r\n", body, errnum, shortmsg);
    sprintf(body, "%s<p>%s: %s\r\n", body, longmsg, cause);
    sprintf(body, "%s<hr><em>The Tiny Web server</em>\r\n", body);

    /* Print the HTTP response */
    sprintf(buf, "HTTP/1.0 %s %s\r\n", errnum, shortmsg);
    rio_writen(fd, buf, strlen(buf));
    sprintf(buf, "Content-type: text/html\r\n");
    rio_writen(fd, buf, strlen(buf));
    sprintf(buf, "Content-length: %d\r\n\r\n", (int)strlen(body));
    rio_writen(fd, buf, strlen(buf));
    rio_writen(fd, body, strlen(body));
}
/* $end clienterror */
```

# My HTML Solution home.html

```html
<html>
   <head>
		<meta http-equiv="Content-Type" content="text/html; charset=utf-8">
		<title>Tiny Web Server</title>
   </head>
	
<body>

	<body style="background-color:black;">
	
	<h1><marquee style="z-index:2;position:absolute;left:20;top:8;font-family:Cursive;font-size:6pt;color:00ff00;
		height:650;"scrollamount="5" direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />
		よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />
		こ<br />そ<br />マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br />
		マ<br />ト<br />リ<br />ッ<br />ク<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:25;top:2;font-family:Cursive;font-size:8pt;color:00ff00;
		height:650;"scrollamount="4" direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />
		よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />
		こ<br />そ<br />マ<br />ト<br />リ<br />ッ<br />ク<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:35;top:2;font-family:Cursive;font-size:5pt;color:00ff00;
		height:650;"scrollamount="8" direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />
		よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />
		こ<br />そ<br />マ<br />ト<br />リ<br />ッ<br />ク<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:40;top:8;font-family:Cursive;font-size:5pt;color:00ff00;
		height:650;"scrollamount="7" direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />
		よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />
		こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:45;top:3;font-family:Cursive;font-size:6pt;color:00ff00;
		height:650;"scrollamount="6" direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />
		よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />
		こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:85;top:4;font-family:Cursive;font-size:7pt;color:00ff00;
		height:650;"scrollamount="4" direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />
		よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:90;top:5;font-family:Cursive;font-size:5pt;color:00ff00;
		height:650;"scrollamount="6" direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />
		よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:95;top:6;font-family:Cursive;font-size:9pt;color:00ff00;
		height:650;"scrollamount="5" direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />
		よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />
		こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:120;top:3;font-family:Cursive;font-size:6pt;color:00ff00;
		height:650;"scrollamount="6" direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />
		よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:125;top:5;font-family:Cursive;font-size:7pt;color:00ff00;
		height:650;"scrollamount="5" direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />
		よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:130;top:8;font-family:Cursive;font-size:20pt;color:00ff00;
		height:650;"scrollamount="6" direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />
		よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />
		こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:140;top:5;font-family:Cursive;font-size:35pt;color:00ff00;
		height:650;"scrollamount="4" direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ
		<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:145;top:6;font-family:Cursive;font-size:6pt;color:00ff00;
		height:650;"scrollamount="7" direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />
		よ<br />う<br />こ<br />そ<br /> </marquee></h1>	
	<h1><marquee style="z-index:2;position:absolute;left:165;top:4;font-family:Cursive;font-size:7pt;color:00ff00;
		height:650;"scrollamount="4" direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ
		<br />う<br />こ<br />そ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ
		<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ<br />ク
		<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:170;top:5;font-family:Cursive;font-size:5pt;color:00ff00;
		height:650;"scrollamount="6" direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />
		よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:175;top:6;font-family:Cursive;font-size:5pt;color:00ff00;
		height:650;"scrollamount="5" direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ
		<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:190;top:3;font-family:Cursive;font-size:6pt;color:00ff00;
		height:650;"scrollamount="6" direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />
		よ<br />う<br />こ<br />そ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br />マ<br />ト<br />
		リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ<br />
		ク<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:195;top:5;font-family:Cursive;font-size:4pt;color:00ff00;
		height:650;"scrollamount="7" direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />
		よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:200;top:8;font-family:Cursive;font-size:8pt;color:00ff00;
		height:650;"scrollamount="5" direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />
		よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:205;top:5;font-family:Cursive;font-size:5pt;color:00ff00;
		height:650;"scrollamount="4" direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />
		よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:210;top:6;font-family:Cursive;font-size:6pt;color:00ff00;
		height:650;"scrollamount="6" direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />
		よ<br />う<br />こ<br />そ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br />マ<br />ト<br />
		リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ<br />
		ク<br /> </marquee></h1>	
	<h1><marquee style="z-index:2;position:absolute;left:225;top:2;font-family:Cursive;font-size:7pt;color:00ff00;
		height:650;"scrollamount="2" direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />
		よ<br />う<br />こ<br />そ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ
		<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ<br />ク
		<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:230;top:5;font-family:Cursive;font-size:15pt;color:00ff00;
		height:650;"scrollamount="4" direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ
		<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:235;top:6;font-family:Cursive;font-size:3pt;color:00ff00;
		height:650;"scrollamount="5" direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />
		よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:245;top:4;font-family:Cursive;font-size:20pt;color:00ff00;
		height:650;"scrollamount="3" direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ
		<br />う<br />こ<br />そ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ
		<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ<br />ク
		<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:250;top:2;font-family:Cursive;font-size:25pt;color:00ff00;
		height:650;"scrollamount="7" direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ
		<br />う<br />こ<br />そ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ
		<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ<br />ク
		<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:255;top:8;font-family:Cursive;font-size:8pt;color:00ff00;
		height:650;"scrollamount="5" direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />
		よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:265;top:5;font-family:Cursive;font-size:50pt;color:00ff00;
		height:650;"scrollamount="4" direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ
		<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:275;top:6;font-family:Cursive;font-size:6pt;color:00ff00;
		height:650;"scrollamount="6" direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />
		よ<br />う<br />こ<br />そ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ
		<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ<br />ク
		<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:295;top:4;font-family:Cursive;font-size:7pt;color:00ff00;
		height:650;"scrollamount="2" direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />
		よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:300;top:5;font-family:Cursive;font-size:5pt;color:00ff00;
		height:650;"scrollamount="6" direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />
		よ<br />う<br />こ<br />そ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br />マ<br />ト<br />
		リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ<br />
		ク<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:305;top:6;font-family:Cursive;font-size:5pt;color:00ff00;
		height:650;"scrollamount="4" direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />
		よ<br />う<br />こ<br />そ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ
		<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ<br />ク
		<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:310;top:3;font-family:Cursive;font-size:3pt;color:00ff00;
		height:650;"scrollamount="3" direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />
		よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:330;top:5;font-family:Cursive;font-size:4pt;color:00ff00;
		height:650;"scrollamount="7" direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />
		よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:335;top:8;font-family:Cursive;font-size:20pt;color:00ff00;
		height:650;"scrollamount="5" direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />
		よ<br />う<br />こ<br />そ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ
		<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ<br />ク
		<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:340;top:2;font-family:Cursive;font-size:15pt;color:00ff00;
		height:650;"scrollamount="3" direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />
		よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:350;top:6;font-family:Cursive;font-size:20pt;color:00ff00;
		height:650;"scrollamount="2" direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ
		<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ
		<br />そ<br /></marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:350;top:3;font-family:Cursive;font-size:3pt;color:00ff00;
		height:650;"scrollamount="5" direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />
		よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />
		こ<br />そ<br /></marquee></h1>
	
	<h1><marquee style="z-index:2;position:absolute;left:355;top:8;font-family:Cursive;font-size:6pt;color:00ff00;
		height:650;"scrollamount="5" direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />
		よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />
		こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:365;top:2;font-family:Cursive;font-size:8pt;color:00ff00;
		height:650;"scrollamount="4" direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />
		よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />
		こ<br />そ<br />マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br /> 
		</marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:370;top:2;font-family:Cursive;font-size:5pt;color:00ff00;
		height:650;"scrollamount="8" direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ
		<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:375;top:8;font-family:Cursive;font-size:5pt;color:00ff00;
		height:650;"scrollamount="7" direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />
		よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:380;top:3;font-family:Cursive;font-size:16pt;color:00ff00;
		height:650;"scrollamount="6" direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />
		よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:415;top:4;font-family:Cursive;font-size:7pt;color:00ff00;
		height:650;"scrollamount="4" direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />
		よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:420;top:5;font-family:Cursive;font-size:5pt;color:00ff00;
		height:650;"scrollamount="6" direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />
		よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />
		こ<br />そ<br />マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br /> 
		</marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:425;top:6;font-family:Cursive;font-size:9pt;color:00ff00;
		height:650;"scrollamount="5" direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />
		よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:450;top:3;font-family:Cursive;font-size:16pt;color:00ff00;
		height:650;"scrollamount="6" direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />
		よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:455;top:5;font-family:Cursive;font-size:7pt;color:00ff00;
		height:650;"scrollamount="5" direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />
		よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ
		<br />そ<br />マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br /> 
		</marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:465;top:8;font-family:Cursive;font-size:4pt;color:00ff00;
		height:650;"scrollamount="6" direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />
		よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:470;top:5;font-family:Cursive;font-size:5pt;color:00ff00;
		height:650;"scrollamount="4" direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />
		よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:475;top:6;font-family:Cursive;font-size:6pt;color:00ff00;
		height:650;"scrollamount="7" direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />
		よ<br />う<br />こ<br />そ<br /> </marquee></h1>	
	<h1><marquee style="z-index:2;position:absolute;left:495;top:4;font-family:Cursive;font-size:7pt;color:00ff00;
		height:650;"scrollamount="4" direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />
		よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ
		<br />そ<br />マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br /> 
		</marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:500;top:5;font-family:Cursive;font-size:15pt;color:00ff00;
		height:650;"scrollamount="6" direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ
		<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:505;top:6;font-family:Cursive;font-size:5pt;color:00ff00;
		height:650;"scrollamount="5" direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />
		よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:520;top:3;font-family:Cursive;font-size:6pt;color:00ff00;
		height:650;"scrollamount="6" direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />
		よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ
		<br />そ<br />マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br /> 
		</marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:525;top:5;font-family:Cursive;font-size:4pt;color:00ff00;
		height:650;"scrollamount="7" direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />
		よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ
		<br />そ<br />マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br /> 
		</marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:530;top:8;font-family:Cursive;font-size:8pt;color:00ff00;
		height:650;"scrollamount="5" direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />
		よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />
		こ<br />そ<br />マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う	<br />こ<br />そ<br /> 
		</marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:535;top:5;font-family:Cursive;font-size:5pt;color:00ff00;
		height:650;"scrollamount="4" direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />
		よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:540;top:6;font-family:Cursive;font-size:6pt;color:00ff00;
		height:650;"scrollamount="6" direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />
		よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />
		こ<br />そ<br />マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br /> 
		</marquee></h1>	
	<h1><marquee style="z-index:2;position:absolute;left:555;top:2;font-family:Cursive;font-size:7pt;color:00ff00;
		height:650;"scrollamount="2" direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />
		よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:560;top:5;font-family:Cursive;font-size:5pt;color:00ff00;
		height:650;"scrollamount="4" direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />
		よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ
		<br />そ<br />マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br /> 
		</marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:565;top:6;font-family:Cursive;font-size:3pt;color:00ff00;
		height:650;"scrollamount="5" direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />
		よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:575;top:4;font-family:Cursive;font-size:6pt;color:00ff00;
		height:650;"scrollamount="3" direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />
		よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:580;top:2;font-family:Cursive;font-size:4pt;color:00ff00;
		height:650;"scrollamount="7" direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />
		よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />
		こ<br />そ<br />マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br /> 
		</marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:585;top:8;font-family:Cursive;font-size:40pt;color:00ff00;
		height:650;"scrollamount="5" direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ
		<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:595;top:5;font-family:Cursive;font-size:5pt;color:00ff00;
		height:650;"scrollamount="4" direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />
		よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ
		<br />そ<br />マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br /> 
		</marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:605;top:6;font-family:Cursive;font-size:15pt;color:00ff00;
		height:650;"scrollamount="6" direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />
		よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:625;top:4;font-family:Cursive;font-size:7pt;color:00ff00;
		height:650;"scrollamount="2" direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />
		よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:630;top:5;font-family:Cursive;font-size:5pt;color:00ff00;
		height:650;"scrollamount="6" direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />
		よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:635;top:6;font-family:Cursive;font-size:5pt;color:00ff00;
		height:650;"scrollamount="4" direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />
		よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />
		こ<br />そ<br />マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う
		<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:640;top:3;font-family:Cursive;font-size:3pt;color:00ff00;
		height:650;"scrollamount="3" direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />
		よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:660;top:5;font-family:Cursive;font-size:10pt;color:00ff00;
		height:650;"scrollamount="7" direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />
		よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />
		こ<br />そ<br />マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br /> 
		</marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:665;top:8;font-family:Cursive;font-size:8pt;color:00ff00;
		height:650;"scrollamount="5" direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />
		よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ
		<br />そ<br />マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br /> 
		</marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:670;top:2;font-family:Cursive;font-size:5pt;color:00ff00;
		height:650;"scrollamount="3" direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />
		よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:680;top:6;font-family:Cursive;font-size:8pt;color:00ff00;
		height:650;"scrollamount="2" direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />
		よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />
		こ<br />そ<br /></marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:680;top:3;font-family:Cursive;font-size:3pt;color:00ff00;
		height:650;"scrollamount="5" direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />
		よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />
		こ<br />そ<br /></marquee></h1>

	<h1><marquee style="z-index:2;position:absolute;left:685;top:8;font-family:Cursive;font-size:6pt;color:00ff00;height:650;"scrollamount="5" 
		direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ
		<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:695;top:2;font-family:Cursive;font-size:8pt;color:00ff00;height:650;"scrollamount="4" 
		direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ
		<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:700;top:2;font-family:Cursive;font-size:15pt;color:00ff00;height:650;"scrollamount="8" 
		direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ
		<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う
		<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:705;top:8;font-family:Cursive;font-size:5pt;color:00ff00;height:650;"scrollamount="7" 
		direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ
		<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:710;top:3;font-family:Cursive;font-size:16pt;color:00ff00;height:650;"scrollamount="6" 
		direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:745;top:4;font-family:Cursive;font-size:7pt;color:00ff00;height:650;"scrollamount="4" 
		direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:750;top:5;font-family:Cursive;font-size:5pt;color:00ff00;height:650;"scrollamount="6" 
		direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ
		<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:755;top:6;font-family:Cursive;font-size:9pt;color:00ff00;height:650;"scrollamount="5" 
		direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ
		<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う
		<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:775;top:3;font-family:Cursive;font-size:6pt;color:00ff00;height:650;"scrollamount="6" 
		direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:780;top:5;font-family:Cursive;font-size:17pt;color:00ff00;height:650;"scrollamount="5" 
		direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ
		<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ<br />ク<br />
	ス<br />へ<br />よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:790;top:8;font-family:Cursive;font-size:4pt;color:00ff00;height:650;"scrollamount="6" 
		direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ
		<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:795;top:5;font-family:Cursive;font-size:15pt;color:00ff00;height:650;"scrollamount="4" 
		direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:815;top:6;font-family:Cursive;font-size:6pt;color:00ff00;height:650;"scrollamount="7" 
		direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ
		<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br /> </marquee></h1>	
	<h1><marquee style="z-index:2;position:absolute;left:820;top:4;font-family:Cursive;font-size:7pt;color:00ff00;height:650;"scrollamount="4" 
		direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ
		<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う
		<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:825;top:5;font-family:Cursive;font-size:5pt;color:00ff00;height:650;"scrollamount="6" 
		direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:840;top:6;font-family:Cursive;font-size:5pt;color:00ff00;height:650;"scrollamount="5" 
		direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ
		<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う
		<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:845;top:3;font-family:Cursive;font-size:6pt;color:00ff00;height:650;"scrollamount="6" 
		direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ
		<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br /> </marquee></h1>		
	<h1><marquee style="z-index:2;position:absolute;left:850;top:5;font-family:Cursive;font-size:4pt;color:00ff00;height:650;"scrollamount="7" 
		direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:855;top:8;font-family:Cursive;font-size:8pt;color:00ff00;height:650;"scrollamount="5" 
		direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ
		<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ
		<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:860;top:5;font-family:Cursive;font-size:15pt;color:00ff00;height:650;"scrollamount="4" 
		direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:875;top:6;font-family:Cursive;font-size:6pt;color:00ff00;height:650;"scrollamount="6" 
		direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ
		<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br /> </marquee></h1>	
	<h1><marquee style="z-index:2;position:absolute;left:880;top:2;font-family:Cursive;font-size:7pt;color:00ff00;height:650;"scrollamount="2" 
		direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:885;top:5;font-family:Cursive;font-size:5pt;color:00ff00;height:650;"scrollamount="4" 
		direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ
		<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:895;top:6;font-family:Cursive;font-size:13pt;color:00ff00;height:650;"scrollamount="5" 
		direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:900;top:4;font-family:Cursive;font-size:6pt;color:00ff00;height:650;"scrollamount="3" 
		direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ
		<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う
		<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:905;top:2;font-family:Cursive;font-size:4pt;color:00ff00;height:650;"scrollamount="7" 
		direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ
		<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:915;top:8;font-family:Cursive;font-size:8pt;color:00ff00;height:650;"scrollamount="5" 
		direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:925;top:5;font-family:Cursive;font-size:5pt;color:00ff00;height:650;"scrollamount="4" 
		direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ
		<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:945;top:6;font-family:Cursive;font-size:6pt;color:00ff00;height:650;"scrollamount="6" 
		direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:950;top:4;font-family:Cursive;font-size:17pt;color:00ff00;height:650;"scrollamount="2" 
		direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:955;top:5;font-family:Cursive;font-size:5pt;color:00ff00;height:650;"scrollamount="6" 
		direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ
		<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:960;top:6;font-family:Cursive;font-size:15pt;color:00ff00;height:650;"scrollamount="4" 
		direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:980;top:3;font-family:Cursive;font-size:3pt;color:00ff00;height:650;"scrollamount="3" 
		direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ
		<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う
		<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:985;top:5;font-family:Cursive;font-size:14pt;color:00ff00;height:650;"scrollamount="7" 
		direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ
		<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:990;top:8;font-family:Cursive;font-size:8pt;color:00ff00;height:650;"scrollamount="5" 
		direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:995;top:2;font-family:Cursive;font-size:5pt;color:00ff00;height:650;"scrollamount="3" 
		direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:1005;top:6;font-family:Cursive;font-size:28pt;color:00ff00;height:650;"scrollamount="2" 
		direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ
		<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br /></marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:1005;top:3;font-family:Cursive;font-size:3pt;color:00ff00;height:650;"scrollamount="5" 
		direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ
		<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br /></marquee></h1>

	<h1><marquee style="z-index:2;position:absolute;left:1010;top:8;font-family:Cursive;font-size:16pt;color:00ff00;height:650;"scrollamount="5" 
		direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ
		<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う
		<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:1020;top:2;font-family:Cursive;font-size:8pt;color:00ff00;height:650;"scrollamount="4" 
		direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ
		<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う
		<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:1025;top:2;font-family:Cursive;font-size:5pt;color:00ff00;height:650;"scrollamount="8" 
		direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ
		<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:1030;top:8;font-family:Cursive;font-size:5pt;color:00ff00;height:650;"scrollamount="7" 
		direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:1035;top:3;font-family:Cursive;font-size:26pt;color:00ff00;height:650;"scrollamount="6" 
		direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ
		<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う
		<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:1070;top:4;font-family:Cursive;font-size:17pt;color:00ff00;height:650;"scrollamount="4" 
		direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ
		<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:1075;top:5;font-family:Cursive;font-size:5pt;color:00ff00;height:650;"scrollamount="6" 
		direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:1080;top:6;font-family:Cursive;font-size:9pt;color:00ff00;height:650;"scrollamount="5" 
		direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:1100;top:3;font-family:Cursive;font-size:6pt;color:00ff00;height:650;"scrollamount="6" 
		direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ
		<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:1105;top:5;font-family:Cursive;font-size:7pt;color:00ff00;height:650;"scrollamount="5" 
		direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:1115;top:8;font-family:Cursive;font-size:4pt;color:00ff00;height:650;"scrollamount="6" 
		direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ
		<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う
		<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:1120;top:5;font-family:Cursive;font-size:5pt;color:00ff00;height:650;"scrollamount="4" 
		direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:1130;top:6;font-family:Cursive;font-size:6pt;color:00ff00;height:650;"scrollamount="7" 
		direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ
		<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う
		<br />こ<br />そ<br /> </marquee></h1>	
	<h1><marquee style="z-index:2;position:absolute;left:1135;top:4;font-family:Cursive;font-size:7pt;color:00ff00;height:650;"scrollamount="4" 
		direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:1140;top:5;font-family:Cursive;font-size:5pt;color:00ff00;height:650;"scrollamount="6" 
		direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ
		<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:1155;top:6;font-family:Cursive;font-size:5pt;color:00ff00;height:650;"scrollamount="5" 
		direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:1160;top:3;font-family:Cursive;font-size:6pt;color:00ff00;height:650;"scrollamount="6" 
		direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ<br />ク
		<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br /> </marquee></h1>		
	<h1><marquee style="z-index:2;position:absolute;left:1165;top:5;font-family:Cursive;font-size:4pt;color:00ff00;height:650;"scrollamount="7" 
		direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:1170;top:8;font-family:Cursive;font-size:8pt;color:00ff00;height:650;"scrollamount="5" 
		direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ
		<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う
		<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:1175;top:5;font-family:Cursive;font-size:5pt;color:00ff00;height:650;"scrollamount="4" 
		direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:1190;top:6;font-family:Cursive;font-size:6pt;color:00ff00;height:650;"scrollamount="6" 
		direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br /> </marquee></h1>	
	<h1><marquee style="z-index:2;position:absolute;left:1195;top:2;font-family:Cursive;font-size:7pt;color:00ff00;height:650;"scrollamount="2" 
		direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ
		<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:1200;top:5;font-family:Cursive;font-size:5pt;color:00ff00;height:650;"scrollamount="4" 
		direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:1210;top:6;font-family:Cursive;font-size:3pt;color:00ff00;height:650;"scrollamount="5" 
		direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ
		<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う
		<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:1215;top:4;font-family:Cursive;font-size:16pt;color:00ff00;height:650;"scrollamount="3" 
		direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:1220;top:2;font-family:Cursive;font-size:4pt;color:00ff00;height:650;"scrollamount="7" 
		direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ
		<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:1230;top:8;font-family:Cursive;font-size:8pt;color:00ff00;height:650;"scrollamount="5" 
		direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:1240;top:5;font-family:Cursive;font-size:25pt;color:00ff00;height:650;"scrollamount="4" 
		direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ
		<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:1260;top:6;font-family:Cursive;font-size:6pt;color:00ff00;height:650;"scrollamount="6" 
		direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:1265;top:4;font-family:Cursive;font-size:7pt;color:00ff00;height:650;"scrollamount="2" 
		direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ
		<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:1270;top:5;font-family:Cursive;font-size:5pt;color:00ff00;height:650;"scrollamount="6" 
		direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:1275;top:6;font-family:Cursive;font-size:5pt;color:00ff00;height:650;"scrollamount="4" 
		direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ
		<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う
		<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:1295;top:3;font-family:Cursive;font-size:3pt;color:00ff00;height:650;"scrollamount="3" 
		direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:1300;top:5;font-family:Cursive;font-size:4pt;color:00ff00;height:650;"scrollamount="7" 
		direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ
		<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:1305;top:8;font-family:Cursive;font-size:8pt;color:00ff00;height:650;"scrollamount="5" 
		direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:1310;top:2;font-family:Cursive;font-size:5pt;color:00ff00;height:650;"scrollamount="3" 
		direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ
		<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う
		<br />こ<br />そ<br /> </marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:1320;top:6;font-family:Cursive;font-size:8pt;color:00ff00;height:650;"scrollamount="2" 
		direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ
		<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br /></marquee></h1>
	<h1><marquee style="z-index:2;position:absolute;left:1320;top:3;font-family:Cursive;font-size:3pt;color:00ff00;height:650;"scrollamount="5" 
		direction="down"> マ<br />ト<br />リ<br />ッ<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br />マ<br />ト<br />リ<br />ッ
		<br />ク<br />ス<br />へ<br />よ<br />う<br />こ<br />そ<br /></marquee></h1>


	<span style="position:absolute;top:400px"></span> 
	<h1><font color="00ff00"> GSP215::Tiny Web Server </font></h1>
	<br></br>
	<br></br>
	<br></br>
	<br></br>
	<br></br>
	<p><font color="00ff00">My name is Joshua Kidder</font></p>
	<p><font color="00ff00">This is a simple raining marquee</font></p>
	<br></br>
	<br></br>
	<br></br>
	<br></br>
	<br></br>
	<br></br>
	<br></br>
	<p><font color="00ff00">The matrix has you...</font></p>">

</body>
</html>
```

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
 * Tiny doit: Handles one HTTP transaction.
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

# The clienterror Function

Tiny lacks many of the error handling features of a real server. However, it does check for some obvious errors and reports them to the client. The clienterror function sends an HTTP response to the client with the appropriate status code and status message in the response line, along with an HTML file in the response body that explains the error to the browser’s user.

```c
/*
 * Tiny clienterror: Sends an error message to the client.
 */

void clienterror(int fd, char *cause, char *errnum, 
		 char *shortmsg, char *longmsg) 
{
    char buf[MAXLINE], body[MAXBUF];

    /* Build the HTTP response body */
    sprintf(body, "<html><title>Tiny Error</title>");
    sprintf(body, "%s<body bgcolor=""ffffff"">\r\n", body);
    sprintf(body, "%s%s: %s\r\n", body, errnum, shortmsg);
    sprintf(body, "%s<p>%s: %s\r\n", body, longmsg, cause);
    sprintf(body, "%s<hr><em>The Tiny Web server</em>\r\n", body);

    /* Print the HTTP response */
    sprintf(buf, "HTTP/1.0 %s %s\r\n", errnum, shortmsg);
    rio_writen(fd, buf, strlen(buf));
    sprintf(buf, "Content-type: text/html\r\n");
    rio_writen(fd, buf, strlen(buf));
    sprintf(buf, "Content-length: %d\r\n\r\n", (int)strlen(body));
    rio_writen(fd, buf, strlen(buf));
    rio_writen(fd, body, strlen(body));
}
```

# The read_requesthdrs Function

Tiny does not use any of the information in the request headers. Tiny reads and ignores them by calling the read_requesthdrs function, the empty text line that terminates the request headers consists of a carriage return and line feed pair.

```c
/*
 * Tiny read_requesthdrs: Reads and ignores request headers.
 */
 
 void read_requesthdrs(rio_t *rp) 
{
    char buf[MAXLINE];

    rio_readlineb(rp, buf, MAXLINE);
    printf("%s", buf);
    while(strcmp(buf, "\r\n")) {
	rio_readlineb(rp, buf, MAXLINE);
	printf("%s", buf);
    }
    return;
}
```

# The parse_uri Function

Tiny assumes that the home directory for static content is its current directory, and that the home directory for executables is ./cgi-bin. Any URI that contains the string cgi-bin is assumed to denote a request for dynamic content. The default file name is ./home.html.

The parse_uri function implements these policies. The function parses the URI into a file name and an optional CGI argument string.

---If the request is for static content, the function clears the CGI argument string and then converts the URI into a relative Unix pathname such as ./index.html. 
---If the URI ends with a ‘/’ character, the function appends the default file name. 
---If the request is for dynamic content, the function extracts any CGI arguments and converts the remaining portion of the URI to a relative Unix file name.

```c
/*
 * Tiny parse_uri: Parses an HTTP URI and CGI ARGS
 *                 Returns 0 if dynamic content, 1 if static
 */
 
int parse_uri(char *uri, char *filename, char *cgiargs) 
{
    char *ptr;

    if (!strstr(uri, "cgi-bin")) {  /* Static content */
	strcpy(cgiargs, "");
	strcpy(filename, ".");
	strcat(filename, uri);
	if (uri[strlen(uri)-1] == '/')
	    strcat(filename, "home.html");
	return 1;
    }
    else {  /* Dynamic content */
	ptr = index(uri, '?');
	if (ptr) {
	    strcpy(cgiargs, ptr+1);
	    *ptr = '\0';
	}
	else 
	    strcpy(cgiargs, "");
	strcpy(filename, ".");
	strcat(filename, uri);
	return 0;
    }
}
```

# The serve_static Function

Tiny serves four different types of static content: HTML files, unformatted text files, and images encoded in GIF and JPEG formats. The serve_static function sends an HTTP response whose body contains the contents of a local file. The function determines the file type by inspecting the suffix in the file name and then send the response line and response headers to the client. A blank line terminates the headers.

```c
/*
 * Tiny serve_static: Serves static content to a client (copies a file back to client). 
 */

void serve_static(int fd, char *filename, int filesize) 
{
    int srcfd;
    char *srcp, filetype[MAXLINE], buf[MAXBUF];
 
    /* Send response headers to client */
    get_filetype(filename, filetype);
    sprintf(buf, "HTTP/1.0 200 OK\r\n");
    sprintf(buf, "%sServer: Tiny Web Server\r\n", buf);
    sprintf(buf, "%sContent-length: %d\r\n", buf, filesize);
    sprintf(buf, "%sContent-type: %s\r\n\r\n", buf, filetype);
    rio_writen(fd, buf, strlen(buf));

    /* Send response body to client */
    srcfd = open(filename, O_RDONLY, 0);
    srcp = mmap(0, filesize, PROT_READ, MAP_PRIVATE, srcfd, 0);
    close(srcfd);
    rio_writen(fd, srcp, filesize);
    munmap(srcp, filesize);
}
```

# The serve_dynamic Function

Tiny serves any type of dynamic content by forking a child process, and then running a CGI program in the context of the child. The serve_dynamic function begins by sending a response line indicating success to the client, along with an informational Server header. The CGI program is responsible for sending the rest of the response. 

```c
/*
 * Tiny serve_dynamic: Serves dynamic content to a client (runs a CGI program on behalf of the client).
 */

void serve_dynamic(int fd, char *filename, char *cgiargs) 
{
    char buf[MAXLINE], *emptylist[] = { NULL };

    /* Return first part of HTTP response */
    sprintf(buf, "HTTP/1.0 200 OK\r\n");
    rio_writen(fd, buf, strlen(buf));
    sprintf(buf, "Server: Tiny Web Server\r\n");
    rio_writen(fd, buf, strlen(buf));
  
    if (fork() == 0) { /* child */
	/* Real server would set all CGI vars here */
	setenv("QUERY_STRING", cgiargs, 1); 
	dup2(fd, STDOUT_FILENO);         /* Redirect stdout to client */
	execve(filename, emptylist, environ); /* Run CGI program */
    }
    wait(NULL); /* Parent waits for and reaps child */
}
```

# DNS mapping, program compilation, and executing listener

![cygwinExecution](https://raw.githubusercontent.com/kiddjsh/cygwinTinyWebServer/main/images/cygwinExecution.PNG)

# E:\cygwin64\home\KidderJoshua

![directoryFiles](https://raw.githubusercontent.com/kiddjsh/cygwinTinyWebServer/main/images/directoryFiles.PNG)
