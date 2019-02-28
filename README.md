# Sockets in C/C++ for Raspberry Pi
This is a quick tutorial on socket programming in C/C++ on a Raspbian Raspberry Pi, based on [Socket programming in C on Linux – tutorial](https://www.binarytides.com/socket-programming-c-linux-tutorial/). I complemented with a part for Raspbian OS, and some changes more likely to my personal taste. 

## Sockets
* *Sockets* are the "virtual" endpoints of any kind of network communications done between 2 hosts over in a network. 
* All along the tutorial there are code snippets to demonstrate some concepts. You can run those code snippets in geany rightaway and test the results to better understand the concepts.

---
## Socket Client

A client is an application that connects to a remote system to fetch or retrieve data. In this case, the steps to run such activities are
1. Create a socket
2. Connect to remote server
3. Send some data
4. Receive a reply

### Create a socket
This first thing to do is create a socket. The socket function does this. For example, 
```C++
#include <stdio.h>
#include <sys/socket.h>

int main(int argc, char &argv[]){
    int socket_desc;
    socket_desc = socket(AF_INET, SOCK_STREAM, 0);
    if (socket == -1) printf("Could not create socket.");
    return 0;
}
```
The function `socket()` creates a socket and returns a descriptor which will be used by other functions. The above code will create a socket with the following properties:
* **Address family** : `AF_INET` (this is IPV4)
* **Type** : `SOCK_STREAM` (this means connection oriented to TCP protocol)
* **Protocol - 0** : or IPPROTO_IP, this is IP protocol

**Quick note**: Apart from `SOCK_STREAM` type of sockets there is another type called `SOCK_DGRAM` which indicates the UDP protocol. This type of socket is non-connection socket. In this tutorial we shall stick to `SOCK_STREAM` or TCP sockets. 

### Connect Socket to a Server
We connect to a remote server on a certain port number, so we need the IP address and port number to connect to.

To connect to a remote server we need to do a couple of things. First is to create a `sockaddr_in` structure with proper values.

```C++
struct sockaddr_in server;
```
So take a look in the following structures. In this case, the `sockaddr_in` has a member called `sin_addr` of type `in_addr` which has a s_addr which is nothing but a long. It contains the IP address in long format.
```C++
// IPv4 AF_INET sockets:
struct sockaddr_in {
    short            sin_family;   // e.g. AF_INET, AF_INET6
    unsigned short   sin_port;     // e.g. htons(3490)
    struct in_addr   sin_addr;     // see struct in_addr, below
    char             sin_zero[8];  // zero this if you want to
};

struct in_addr {
    unsigned long s_addr;          // load with inet_pton()
};

struct sockaddr {
    unsigned short    sa_family;    // address family, AF_xxx
    char              sa_data[14];  // 14 bytes of protocol address
};
```

Function `inet_addr` is a very handy function to convert an IP address to a long format. This is how you do it :
```C++
server.sin_addr.s_addr = inet_addr("74.125.235.20");
```

So you need to know the IP address of the remote server you are connecting to. Here we used the ip address of google.com as a sample. [Appendix 1](#appendix-1) section shows how to find out the ip address of a given domain name.

The last thing needed is the `connect` function. It needs a `socket` and a `sockaddr` structure to connect to. Here is a code sample.

```C++
#include<stdio.h>
#include<sys/socket.h>
#include<arpa/inet.h>	//inet_addr

int main(int argc , char *argv[]){
	int socket_desc;
	struct sockaddr_in server;
	
	//Create socket
	socket_desc = socket(AF_INET , SOCK_STREAM , 0);
	if (socket_desc == -1) printf("Could not create socket");
			
	server.sin_addr.s_addr = inet_addr("74.125.235.20");
	server.sin_family = AF_INET;
	server.sin_port = htons( 80 );

	//Connect to remote server
	if (connect(socket_desc , (struct sockaddr *)&server , sizeof(server)) < 0){
		puts("connect error");
		return 1;
	}
	
	puts("Connected");
	return 0;
}
```

It cannot be any simpler. It creates a socket and then connects. If you run the program it should show Connected. Try connecting to a port different from port 80 and you should not be able to connect which indicates that the port is not open for connection.

So we are now connected. The next step is to send some data to the remote server.

**Note : Connections are present only in tcp sockets** : 
The concept of "connections" apply to SOCK_STREAM/TCP type of sockets. Connection means a reliable "stream" of data such that there can be multiple such streams each having communication of its own. Think of this as a pipe which is not interfered by other data.

Other sockets like UDP , ICMP , ARP dont have a concept of "connection". These are non-connection based communication. Which means you keep sending or receiving packets from anybody and everybody.

### Send data over socket

Function `send` will simply send data. It needs the socket descriptor, the data to send and its size. Here is a very simple example of sending some data to google.com IP :

```C++
#include<stdio.h>
#include<string.h>	//strlen
#include<sys/socket.h>
#include<arpa/inet.h>	//inet_addr

int main(int argc , char *argv[]){
	int socket_desc;
	struct sockaddr_in server;
	char *message;
	
	//Create socket
	socket_desc = socket(AF_INET , SOCK_STREAM , 0);
	if (socket_desc == -1) printf("Could not create socket");
		
	server.sin_addr.s_addr = inet_addr("74.125.235.20");
	server.sin_family = AF_INET;
	server.sin_port = htons(80);

	//Connect to remote server
	if (connect(socket_desc , (struct sockaddr *)&server , sizeof(server)) < 0){
		puts("connect error");
		return 1;
	}
	
	puts("Connected\n");
	
	//Send some data
	message = "GET / HTTP/1.1\r\n\r\n";
	if( send(socket_desc , message , strlen(message) , 0) < 0){
		puts("Send failed");
		return 1;
	}
	puts("Data Send\n");
	
	return 0;
}
```

In the above example , we first connect to an IP address and then send the string message `"GET / HTTP/1.1\r\n\r\n"` to it. The message is actually a http command to fetch the mainpage of a website.

When sending data to a socket you are basically writing data to that socket. This is similar to writing data to a file. Hence you can also use the write function to send data to a socket. Later in this tutorial we shall use write function to send data.

Now that we have sent some data, it's time to receive a reply from the server.

### Receive data from socket

Function `recv` is used to receive data from a socket. In the following example we shall send the same message as the last example and receive a reply from the server.

```C++
#include<stdio.h>
#include<string.h>	//strlen
#include<sys/socket.h>
#include<arpa/inet.h>	//inet_addr

int main(int argc , char *argv[]){
	int socket_desc;
	struct sockaddr_in server;
	char *message , server_reply[2000];
	
	//Create socket
	socket_desc = socket(AF_INET , SOCK_STREAM , 0);
	if (socket_desc == -1) printf("Could not create socket");
		
	server.sin_addr.s_addr = inet_addr("74.125.235.20");
	server.sin_family = AF_INET;
	server.sin_port = htons( 80 );

	//Connect to remote server
	if (connect(socket_desc , (struct sockaddr *)&server , sizeof(server)) < 0){
		puts("connect error");
		return 1;
	}
	
	puts("Connected\n");
	
	//Send some data
	message = "GET / HTTP/1.1\r\n\r\n";
	if( send(socket_desc , message , strlen(message) , 0) < 0){
		puts("Send failed");
		return 1;
	}
	puts("Data Send\n");
	
	//Receive a reply from the server
	if (recv(socket_desc, server_reply , 2000 , 0) < 0){
		puts("recv failed");
	}
	puts("Reply received\n");
	puts(server_reply);
	
	return 0;
}
```

Here is the output of the above code :

```raw
Connected

Data Send

Reply received

HTTP/1.1 302 Found
Location: http://www.google.co.in/
Cache-Control: private
Content-Type: text/html; charset=UTF-8
Set-Cookie: PREF=ID=0edd21a16f0db219:FF=0:TM=1324644706:LM=1324644706:S=z6hDC9cZfGEowv_o; expires=Sun, 22-Dec-2013 12:51:46 GMT; path=/; domain=.google.com
Date: Fri, 23 Dec 2011 12:51:46 GMT
Server: gws
Content-Length: 221
X-XSS-Protection: 1; mode=block
X-Frame-Options: SAMEORIGIN

<HTML><HEAD><meta http-equiv="content-type" content="text/html;charset=utf-8">
<TITLE>302 Moved</TITLE></HEAD><BODY>
<H1>302 Moved</H1>
The document has moved
<A HREF="http://www.google.co.in/">here</A>.
</BODY></HTML>
```

We can see what reply was send by the server. It looks something like Html, well IT IS html. Google.com replied with the content of the page we requested. Quite simple!

**Note** : When receiving data on a socket , we are basically reading the data on the socket. This is similar to reading data from a file. So we can also use the read function to read data on a socket. For example :
```C++ 
read(socket_desc, server_reply , 2000);
```

Now that we have received our reply, its time to close the socket.

### Close socket

Function close is used to close the socket. Need to include the `unistd.h` header file for this.
```C++
close(socket_desc);
```

---
## Socket Server

The other kind of socket application is called a socket server. A server is a system that uses sockets to receive incoming connections and provide them with data. Servers are the opposite of clients, that instead of connecting out to others, they wait for incoming connections. So www.google.com is a server and your web browser is a client. Or more technically www.google.com is a HTTP Server and your web browser is an HTTP client.

Socket servers operate in the following manner
1. Open a socket
2. Bind to a address(and port)
3. Listen for incoming connections
4. Accept connections
5. Read/Send data

We have already learnt how to open a socket. So the next thing would be to bind it.

### Bind socket to a port

The `bind` function can be used to bind a socket to a particular "address and port" combination. It needs a `sockaddr_in` structure similar to `connect` function.
```C++
int socket_desc;
struct sockaddr_in server;
	
//Create socket
socket_desc = socket(AF_INET , SOCK_STREAM , 0);
if (socket_desc == -1) printf("Could not create socket");
	
//Prepare the sockaddr_in structure
server.sin_family = AF_INET;
server.sin_addr.s_addr = INADDR_ANY;
server.sin_port = htons( 8888 );
	
//Bind
if( bind(socket_desc,(struct sockaddr *)&server , sizeof(server)) < 0) puts("bind failed");
puts("bind done");
```

Now that bind is done, its time to make the socket listen to connections. We bind a socket to a particular IP address and a certain port number. By doing this we ensure that all incoming data which is directed towards this port number is received by this application. 

This makes it obvious that you cannot have 2 sockets bound to the same port.

### Listen for incoming connections on the socket

After binding a socket to a port the next thing we need to do is listen for connections. For this we need to put the socket in listening mode. Function `listen` is used to put the socket in listening mode. Just add the following line after bind.

```C++
// Listen
listen(socket_desc , 3);
```

Now comes the main part of accepting new connections.

### Accept connection

Function `accept` is used for this. Here is the code
```C++
#include<stdio.h>
#include<sys/socket.h>
#include<arpa/inet.h>	//inet_addr

int main(int argc , char *argv[]){
	int socket_desc , new_socket , c;
	struct sockaddr_in server , client;
	
	//Create socket
	socket_desc = socket(AF_INET , SOCK_STREAM , 0);
	if (socket_desc == -1) printf("Could not create socket");
	
	//Prepare the sockaddr_in structure
	server.sin_family = AF_INET;
	server.sin_addr.s_addr = INADDR_ANY;
	server.sin_port = htons( 8888 );
	
	//Bind
	if( bind(socket_desc,(struct sockaddr *)&server , sizeof(server)) < 0) puts("bind failed");
	puts("bind done");
	
	//Listen
	listen(socket_desc , 3);
	
	//Accept and incoming connection
	puts("Waiting for incoming connections...");
	c = sizeof(struct sockaddr_in);
	new_socket = accept(socket_desc, (struct sockaddr *)&client, (socklen_t*)&c);
	if (new_socket<0) perror("accept failed");
	
	puts("Connection accepted");

	return 0;
}
```

**Program output** : running the program, it should show
```raw
bind done
Waiting for incoming connections...
```

So now this program is waiting for incoming connections on port 8888. Dont close this program, keep it running. Now a client can connect to it on this port. We shall use the telnet client for testing this. Open a terminal and type
`$ telnet localhost 8888`.

On the terminal you shall get
```raw
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
Connection closed by foreign host.
```

And the server output will show
```raw
bind done
Waiting for incoming connections...
Connection accepted
```

So we can see that the client connected to the server. Try the above process till you get it perfect.

### Get the ip address of the connected client

You can get the ip address of client and the port of connection by using the `sockaddr_in` structure passed to accept function. It is very simple :
```C++
char *client_ip = inet_ntoa(client.sin_addr);
int client_port = ntohs(client.sin_port);
```

We accepted an incoming connection but closed it immediately. This was not very productive. There are lots of things that can be done after an incoming connection is established. Afterall the connection was established for the purpose of communication. So lets reply to the client. 

We can simply use the `write` function to write something to the socket of the incoming connection and the client should see it. Here is an example.

```C++
#include<stdio.h>
#include<string.h>	//strlen
#include<sys/socket.h>
#include<arpa/inet.h>	//inet_addr
#include<unistd.h>	//write

int main(int argc , char *argv[]){
	int socket_desc , new_socket , c;
	struct sockaddr_in server , client;
	char *message;
	
	//Create socket
	socket_desc = socket(AF_INET , SOCK_STREAM , 0);
	if (socket_desc == -1) printf("Could not create socket");
	
	//Prepare the sockaddr_in structure
	server.sin_family = AF_INET;
	server.sin_addr.s_addr = INADDR_ANY;
	server.sin_port = htons( 8888 );
	
	//Bind
	if( bind(socket_desc,(struct sockaddr *)&server , sizeof(server)) < 0){
		puts("bind failed");
		return 1;
	}
	puts("bind done");
	
	//Listen
	listen(socket_desc , 3);
	
	//Accept and incoming connection
	puts("Waiting for incoming connections...");
	c = sizeof(struct sockaddr_in);
	new_socket = accept(socket_desc, (struct sockaddr *)&client, (socklen_t*)&c);
	if (new_socket<0){
		perror("accept failed");
		return 1;
	}
	
	puts("Connection accepted");
	
	//Reply to the client
	message = "Hello Client , I have received your connection. But I have to go now, bye\n";
	write(new_socket , message , strlen(message));
	
	return 0;
}
```

Run the above code in 1 terminal. And connect to this server using telnet from another terminal and you should see this :
```raw
$ telnet localhost 8888
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
Hello Client , I have received your connection. But I have to go now, bye
Connection closed by foreign host.
```

So the client(telnet) received a reply from server.

We can see that the connection is closed immediately after that simply because the server program ends after accepting and sending reply. A server like www.google.com is always up to accept incoming connections. 

It means that a server is supposed to be running all the time. Afterall its a server meant to serve. So we need to keep our server RUNNING non-stop. The simplest way to do this is to put the accept in a loop so that it can receive incoming connections all the time.

### Live Server

So a live server will be alive for all time. Lets code this up :
```C++
#include<stdio.h>
#include<string.h>	//strlen
#include<sys/socket.h>
#include<arpa/inet.h>	//inet_addr
#include<unistd.h>	//write

int main(int argc , char *argv[]){
	int socket_desc , new_socket , c;
	struct sockaddr_in server , client;
	char *message;
	
	//Create socket
	socket_desc = socket(AF_INET , SOCK_STREAM , 0);
	if (socket_desc == -1) printf("Could not create socket");
	
	//Prepare the sockaddr_in structure
	server.sin_family = AF_INET;
	server.sin_addr.s_addr = INADDR_ANY;
	server.sin_port = htons( 8888 );
	
	//Bind
	if( bind(socket_desc,(struct sockaddr *)&server , sizeof(server)) < 0){
		puts("bind failed");
		return 1;
	}
	puts("bind done");
	
	//Listen
	listen(socket_desc , 3);
	
	//Accept and incoming connection
	puts("Waiting for incoming connections...");
	c = sizeof(struct sockaddr_in);
	while ((new_socket = accept(socket_desc, (struct sockaddr *)&client, (socklen_t*)&c))){
		puts("Connection accepted");
		
		//Reply to the client
		message = "Hello Client , I have received your connection. But I have to go now, bye\n";
		write(new_socket , message , strlen(message));
	}
	
	if (new_socket<0){
		perror("accept failed");
		return 1;
	}
	
	return 0;
}
```

We havent done a lot there. Just the accept was put in a loop.

Now run the program in 1 terminal, and open 3 other terminals. From each of the 3 terminal do a telnet to the server port.

Each of the telnet terminal would show :
```raw
$ telnet localhost 8888
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
Hello Client , I have received your connection. But I have to go now, bye
```
And the server terminal would show
```raw
bind done
Waiting for incoming connections...
Connection accepted
Connection accepted
Connection accepted
```

So now the server is running nonstop and the telnet terminals are also connected nonstop. Now close the server program. All telnet terminals would show `Connection closed by foreign host`.  Good so far. But still there is not effective communication between the server and the client.

The server program accepts connections in a loop and just send them a reply, after that it does nothing with them. Also it is not able to handle more than 1 connection at a time. So now its time to handle the connections, and handle multiple connections together.

### Handling multiple socket connections with threads

To handle every connection we need a separate handling code to run along with the main server accepting connections. One way to achieve this is using *threads*. The main server program accepts a connection and creates a new thread to handle communication for the connection, and then the server goes back to accept more connections.

On Linux threading can be done with the `pthread` (posix threads) library. From Raspberry Pi Raspbian OS you may probably have to install *PThreads* library with
```sh
sudo apt-get install libpthread-stubs0-dev
```
	
We shall now use threads to create handlers for each connection the server accepts.
```C++
#include<stdio.h>
#include<string.h>	//strlen
#include<stdlib.h>	//strlen
#include<sys/socket.h>
#include<arpa/inet.h>	//inet_addr
#include<unistd.h>	//write

#include<pthread.h> //for threading , link with lpthread

void *connection_handler(void *);

int main(int argc , char *argv[]){
	int socket_desc , new_socket , c , *new_sock;
	struct sockaddr_in server , client;
	char *message;
	
	//Create socket
	socket_desc = socket(AF_INET , SOCK_STREAM , 0);
	if (socket_desc == -1) printf("Could not create socket");
	
	//Prepare the sockaddr_in structure
	server.sin_family = AF_INET;
	server.sin_addr.s_addr = INADDR_ANY;
	server.sin_port = htons( 8888 );
	
	//Bind
	if( bind(socket_desc,(struct sockaddr *)&server , sizeof(server)) < 0){
		puts("bind failed");
		return 1;
	}
	puts("bind done");
	
	//Listen
	listen(socket_desc , 3);
	
	//Accept and incoming connection
	puts("Waiting for incoming connections...");
	c = sizeof(struct sockaddr_in);
	while ((new_socket = accept(socket_desc, (struct sockaddr *)&client, (socklen_t*)&c))){
		puts("Connection accepted");
		
		//Reply to the client
		message = "Hello Client , I have received your connection. And now I will assign a handler for you\n";
		write(new_socket , message , strlen(message));
		
		pthread_t sniffer_thread;
		new_sock = malloc(1);
		*new_sock = new_socket;
		
		if( pthread_create( &sniffer_thread , NULL ,  connection_handler , (void*) new_sock) < 0){
			perror("could not create thread");
			return 1;
		}
		
		//Now join the thread , so that we dont terminate before the thread
		//pthread_join( sniffer_thread , NULL);
		puts("Handler assigned");
	}
	
	if (new_socket<0){
		perror("accept failed");
		return 1;
	}
	
	return 0;
}

/*
 * This will handle connection for each client
 * */
void *connection_handler(void *socket_desc)
{
	//Get the socket descriptor
	int sock = *(int*)socket_desc;
	
	char *message;
	
	//Send some messages to the client
	message = "Greetings! I am your connection handler\n";
	write(sock , message , strlen(message));
	
	message = "Its my duty to communicate with you";
	write(sock , message , strlen(message));
	
	//Free the socket pointer
	free(socket_desc);
	
	return 0;
}
```

Run the above server and open 3 terminals like before. Now the server will create a thread for each client connecting to it.

```raw
The telnet terminals would show :
$ telnet localhost 8888
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
Hello Client , I have received your connection. And now I will assign a handler for you
Hello I am your connection handler
Its my duty to communicate with you
```

This one looks good , but the communication handler is also quite dumb. After the greeting it terminates. It should stay alive and keep communicating with the client.

One way to do this is by making the connection handler wait for some message from a client as long as the client is connected. If the client disconnects , the connection handler ends.

So the connection handler can be rewritten like this :
```C++
/*
 * This will handle connection for each client
 * */
void *connection_handler(void *socket_desc)
{
	//Get the socket descriptor
	int sock = *(int*)socket_desc;
	int read_size;
	char *message , client_message[2000];
	
	//Send some messages to the client
	message = "Greetings! I am your connection handler\n";
	write(sock , message , strlen(message));
	
	message = "Now type something and i shall repeat what you type \n";
	write(sock , message , strlen(message));
	
	//Receive a message from client
	while ((read_size = recv(sock , client_message , 2000 , 0)) > 0){
		//Send the message back to client
		write(sock , client_message , strlen(client_message));
	}
	
	if(read_size == 0){
		puts("Client disconnected");
		fflush(stdout);
	}
	else if(read_size == -1){
		perror("recv failed");
	}
		
	//Free the socket pointer
	free(socket_desc);
	
	return 0;
}
```

The above connection handler takes some input from the client and replies back with the same. Simple! Here is how the telnet output might look
```raw
$ telnet localhost 8888
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
Hello Client , I have received your connection. And now I will assign a handler for you
Greetings! I am your connection handler
Now type something and i shall repeat what you type 
Hello
Hello
How are you
How are you
I am fine
I am fine
```

So now we have a server thats communicative. Thats useful now.

### Linking the pthread library

When compiling programs that use the pthread library you need to link the library. This is done like this :

```sh
$ gcc program.c -lpthread
```

---
## [Appendix 1 : Hot to get the IP address of hostname](#appendix-1)

When connecting to a remote host, it is necessary to have its IP address. Function `gethostbyname` is used for this purpose. It takes the domain name as the parameter and returns a structure of type `hostent`. This structure has the IP information. It is present in `netdb.h`. Lets have a look at this structure.
```C++
/* Description of data base entry for a single host.  */
struct hostent{
  char *h_name;			/* Official name of host.  */
  char **h_aliases;		/* Alias list.  */
  int h_addrtype;		/* Host address type.  */
  int h_length;			/* Length of address.  */
  char **h_addr_list;		/* List of addresses from name server.  */
};
```

The `h_addr_list` has the IP addresses. So now lets have some code to use them.
```C++
#include<stdio.h> //printf
#include<string.h> //strcpy
#include<sys/socket.h>
#include<netdb.h>	//hostent
#include<arpa/inet.h>

int main(int argc , char *argv[]){
	char *hostname = "www.google.com";
	char ip[100];
	struct hostent *he;
	struct in_addr **addr_list;
	int i;
		
	if ((he = gethostbyname(hostname)) == NULL){
		//gethostbyname failed
		herror("gethostbyname");
		return 1;
	}
	
	//Cast the h_addr_list to in_addr , since h_addr_list also has the ip address in long format only
	addr_list = (struct in_addr **) he->h_addr_list;
	
	for(i = 0; addr_list[i] != NULL; i++){
		//Return the first one;
		strcpy(ip , inet_ntoa(*addr_list[i]) );
	}
	
	printf("%s resolved to : %s" , hostname , ip);
	return 0;
}
```

Output of the code would look like :

```raw
www.google.com resolved to : 74.125.235.20
```

So the above code can be used to find the IP address of any domain name. Then the IP address can be used to make a connection using a socket.

Function `inet_ntoa` will convert an IP address in long format to dotted format. This is just the opposite of `inet_addr`.

So far we have see some important structures that are used. Lets revise them :

1. `sockaddr_in` - Connection information. Used by connect , send , recv etc.
2. `in_addr` - IP address in long format
3. `sockaddr`
4. `hostent` - The IP addresses of a hostname. Used by `gethostbyname`
