http://nanomsg.org/gettingstarted/nng/index.html

[TOC]

Getting Started with 'nng'
==========================

Messaging Patterns
------------------

nng is a messaging framework that attempts to help solve common messaging problems with one of a few patterns ('protocols' in nng parlance.)

Following are examples of each pattern type in C.

Pipeline (A One-Way Pipe)
-------------------------

[image]

This pattern is useful for solving producer/consumer problems, including load-balancing. Messages flow from the push side to the pull side. If multiple peers are connected, the pattern attempts to distribute fairly.

pipeline.c
```C
#include <stdlib.h>
#include <stdio.h>
#include <string.h>
#include <unistd.h>

#include <nng/nng.h>
#include <nng/protocol/pipeline0/pull.h>
#include <nng/protocol/pipeline0/push.h>

#define NODE0 "node0"
#define NODE1 "node1"

void
fatal(const char *func, int rv)
{
        fprintf(stderr, "%s: %s\n", func, nng_strerror(rv));
        exit(1);
}

int
node0(const char *url)
{
        nng_socket sock;
        int rv;

        if ((rv = nng_pull0_open(&sock)) != 0) {
                fatal("nng_pull0_open", rv);
        }
        if ((rv = nng_listen(sock, url, NULL, 0)) != 0) {
                fatal("nng_listen", rv);
        }
        for (;;) {
                char *buf = NULL;
                size_t sz;
                if ((rv = nng_recv(sock, &buf, &sz, NNG_FLAG_ALLOC)) != 0) {
                        fatal("nng_recv", rv);
                }
                printf("NODE0: RECEIVED \"%s\"\n", buf); 
                nng_free(buf, sz);
        }
}

int
node1(const char *url, char *msg)
{
        int sz_msg = strlen(msg) + 1; // '\0' too
        nng_socket sock;
        int rv;
        int bytes;

        if ((rv = nng_push0_open(&sock)) != 0) {
                fatal("nng_push0_open", rv);
        }
        if ((rv = nng_dial(sock, url, NULL, 0)) != 0) {
                fatal("nng_dial", rv);
        }
        printf("NODE1: SENDING \"%s\"\n", msg);
        if ((rv = nng_send(sock, msg, strlen(msg)+1, 0)) != 0) {
                fatal("nng_send", rv);
        }
        sleep(1); // wait for messages to flush before shutting down
        nng_close(sock);
        return (0);
}

int
main(int argc, char **argv)
{
        if ((argc > 1) && (strcmp(NODE0, argv[1]) == 0))
                return (node0(argv[2]));

        if ((argc > 2) && (strcmp(NODE1, argv[1]) == 0))
                return (node1(argv[2], argv[3]));

        fprintf(stderr, "Usage: pipeline %s|%s <URL> <ARG> ...'\n",
                NODE0, NODE1);
        return (1);
}
```

Blithely assumes message is ASCIIZ string. Real code should check it.

Compilation
```
gcc pipeline.c -lnng -o pipeline
```

Execution
```
./pipeline node0 ipc:///tmp/pipeline.ipc & node0=$! && sleep 1
./pipeline node1 ipc:///tmp/pipeline.ipc "Hello, World!"
./pipeline node1 ipc:///tmp/pipeline.ipc "Goodbye."
kill $node0
```

Output
```
NODE1: SENDING "Hello, World!"
NODE0: RECEIVED "Hello, World!"
NODE1: SENDING "Goodbye."
NODE0: RECEIVED "Goodbye."
```


Request/Reply (I ask, you answer)
---------------------------------

[image]

Request/Reply is used for synchronous communications where each question is responded with a single answer, for example remote procedure calls (RPCs). Like Pipeline, it also can perform load-balancing. This is the only reliable messaging pattern in the suite, as it automatically will retry if a request is not matched with a response.

reqprep.c
```C
#include <stdlib.h>
#include <stdio.h>
#include <string.h>
#include <time.h>

#include <nng/nng.h>
#include <nng/protocol/reqrep0/rep.h>
#include <nng/protocol/reqrep0/req.h>

#define NODE0 "node0"
#define NODE1 "node1"
#define DATE "DATE"

void
fatal(const char *func, int rv)
{
        fprintf(stderr, "%s: %s\n", func, nng_strerror(rv));
        exit(1);
}

char *
date(void)
{
        time_t now = time(&now);
        struct tm *info = localtime(&now);
        char *text = asctime(info);
        text[strlen(text)-1] = '\0'; // remove '\n'
        return (text);
}

int
node0(const char *url)
{
        nng_socket sock;
        int rv;

        if ((rv = nng_rep0_open(&sock)) != 0) {
                fatal("nng_rep0_open", rv);
        }
          if ((rv = nng_listen(sock, url, NULL, 0)) != 0) {
                fatal("nng_listen", rv);
        }
        for (;;) {
                char *buf = NULL;
                size_t sz;
                if ((rv = nng_recv(sock, &buf, &sz, NNG_FLAG_ALLOC)) != 0) {
                        fatal("nng_recv", rv);
                }
                if ((sz == (strlen(DATE) + 1)) && (strcmp(DATE, buf) == 0)) {
                        printf("NODE0: RECEIVED DATE REQUEST\n");
                        char *d = date();
                        printf("NODE0: SENDING DATE %s\n", d);
                        if ((rv = nng_send(sock, d, strlen(d) + 1, 0)) != 0) {
                                fatal("nng_send", rv);
                        }
                }
                nng_free(buf, sz);
        }
}

int
node1(const char *url)
{
        nng_socket sock;
        int rv;
        size_t sz;
        char *buf = NULL;

        if ((rv = nng_req0_open(&sock)) != 0) {
                fatal("nng_socket", rv);
        }
        if ((rv = nng_dial(sock, url, NULL, 0)) != 0) {
                fatal("nng_dial", rv);
        }
        printf("NODE1: SENDING DATE REQUEST %s\n", DATE);
        if ((rv = nng_send(sock, DATE, strlen(DATE)+1, 0)) != 0) {
                fatal("nng_send", rv);
        }
        if ((rv = nng_recv(sock, &buf, &sz, NNG_FLAG_ALLOC)) != 0) {
                fatal("nng_recv", rv);
        }
        printf("NODE1: RECEIVED DATE %s\n", buf);  
        nng_free(buf, sz);
        nng_close(sock);
        return (0);
}

int
main(const int argc, const char **argv)
{
        if ((argc > 1) && (strcmp(NODE0, argv[1]) == 0))
                return (node0(argv[2]));

        if ((argc > 1) && (strcmp(NODE1, argv[1]) == 0))
                return (node1(argv[2]));

      fprintf(stderr, "Usage: reqrep %s|%s <URL> ...\n", NODE0, NODE1);
      return (1);
}
```

Blithely assumes message is ASCIIZ string. Real code should check it.

Compilation
```
gcc reqrep.c -lnng -o reqrep
```

Execution
```
./reqrep node0 ipc:///tmp/reqrep.ipc & node0=$! && sleep 1
./reqrep node1 ipc:///tmp/reqrep.ipc
kill $node0
```

Output
```
NODE1: SENDING DATE REQUEST DATE
NODE0: RECEIVED DATE REQUEST
NODE0: SENDING DATE Tue Jan  9 09:17:02 2018
NODE1: RECEIVED DATE Tue Jan  9 09:17:02 2018
```

Pair (Two Way Radio)
--------------------

[image]

The pair pattern is used when there a one-to-one peer relationship. Only one peer may be connected to another peer at a time, but both may speak freely.

pair.c
```C
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

#include <nng/nng.h>
#include <nng/protocol/pair0/pair.h>

#define NODE0 "node0"
#define NODE1 "node1"

void
fatal(const char *func, int rv)
{
        fprintf(stderr, "%s: %s\n", func, nng_strerror(rv));
        exit(1);
}

int
send_name(nng_socket sock, char *name)
{
        int rv;
        printf("%s: SENDING \"%s\"\n", name, name);
        if ((rv = nng_send(sock, name, strlen(name) + 1, 0)) != 0) {
                fatal("nng_send", rv);
        }
        return (rv);
}

int
recv_name(nng_socket sock, char *name)
{
        char *buf = NULL;
        int rv;
        size_t sz;
        if ((rv = nng_recv(sock, &buf, &sz, NNG_FLAG_ALLOC)) == 0) {
                printf("%s: RECEIVED \"%s\"\n", name, buf); 
                nng_free(buf, sz);
        }
        return (rv);
}

int
send_recv(nng_socket sock, char *name)
{
        int rv;
        if ((rv = nng_setopt_ms(sock, NNG_OPT_RECVTIMEO, 100)) != 0) {
                fatal("nng_setopt_ms", rv);
        }
        for (;;) {
                recv_name(sock, name);
                sleep(1);
                send_name(sock, name);
        }
}

int
node0(const char *url)
{
        nng_socket sock;
        int rv;
        if ((rv = nng_pair0_open(&sock)) != 0) {
                fatal("nng_pair0_open", rv);
        }
         if ((rv = nng_listen(sock, url, NULL, 0)) !=0) {
                fatal("nng_listen", rv);
        }
        return (send_recv(sock, NODE0));
}

int
node1(const char *url)
{
        nng_socket sock;
        int rv;
        sleep(1);
        if ((rv = nng_pair0_open(&sock)) != 0) {
                fatal("nng_pair0_open", rv);
        }
        if ((rv = nng_dial(sock, url, NULL, 0)) != 0) {
                fatal("nng_dial", rv);
        }
        return (send_recv(sock, NODE1));
}

int
main(int argc, char **argv)
{
        if ((argc > 1) && (strcmp(NODE0, argv[1]) == 0))
                return (node0(argv[2]));

        if ((argc > 1) && (strcmp(NODE1, argv[1]) == 0))
                return (node1(argv[2]));

        fprintf(stderr, "Usage: pair %s|%s <URL> <ARG> ...\n", NODE0, NODE1);
        return 1;
}
```

Blithely assumes message is ASCIIZ string. Real code should check it.

Compilation
```
gcc pair.c -lnng -o pair
```

Execution
```
./pair node0 ipc:///tmp/pair.ipc & node0=$!
./pair node1 ipc:///tmp/pair.ipc & node1=$!
sleep 3
kill $node0 $node1
```

Output
```
node0: SENDING "node0"
node1: RECEIVED "node0"
node1: SENDING "node1"
node0: SENDING "node0"
node0: RECEIVED "node1"
node1: RECEIVED "node0"
```

Pub/Sub (Topics & Broadcast)
----------------------------

[image]

This pattern is used to allow a single broadcaster to publish messages to many subscribers, which may choose to limit which messages they receive.

pubsub.c
```C
#include <stdlib.h>
#include <stdio.h>
#include <string.h>
#include <time.h>
#include <unistd.h>

#include <nng/nng.h>
#include <nng/protocol/pubsub0/pub.h>
#include <nng/protocol/pubsub0/sub.h>

#define SERVER "server"
#define CLIENT "client"

void
fatal(const char *func, int rv)
{
        fprintf(stderr, "%s: %s\n", func, nng_strerror(rv));
}

char *
date(void)
{
        time_t now = time(&now);
        struct tm *info = localtime(&now);
        char *text = asctime(info);
        text[strlen(text)-1] = '\0'; // remove '\n'
        return (text);
}

int
server(const char *url)
{
        nng_socket sock;
        int rv;

        if ((rv = nng_pub0_open(&sock)) != 0) {
                fatal("nng_pub0_open", rv);
        }
        if ((rv = nng_listen(sock, url, NULL, 0)) < 0) {
                fatal("nng_listen", rv);
        }
        for (;;) {
                char *d = date();
                printf("SERVER: PUBLISHING DATE %s\n", d);
                if ((rv = nng_send(sock, d, strlen(d) + 1, 0)) != 0) {
                        fatal("nng_send", rv);
                }
                sleep(1);
        }
}

int
client(const char *url, const char *name)
{
        nng_socket sock;
        int rv;

        if ((rv = nng_sub0_open(&sock)) != 0) {
                fatal("nng_sub0_open", rv);
        }

        // subscribe to everything (empty means all topics)
        if ((rv = nng_setopt(sock, NNG_OPT_SUB_SUBSCRIBE, "", 0)) != 0) {
                fatal("nng_setopt", rv);
        }
        if ((rv = nng_dial(sock, url, NULL, 0)) != 0) {
                fatal("nng_dial", rv);
        }
        for (;;) {
                char *buf = NULL;
                size_t sz;
                if ((rv = nng_recv(sock, &buf, &sz, NNG_FLAG_ALLOC)) != 0) {
                        fatal("nng_recv", rv);
                }
                printf("CLIENT (%s): RECEIVED %s\n", name, buf); 
                nng_free(buf, sz);
        }
}

int
main(const int argc, const char **argv)
{
        if ((argc >= 2) && (strcmp(SERVER, argv[1]) == 0))
                return (server(argv[2]));

          if ((argc >= 3) && (strcmp(CLIENT, argv[1]) == 0))
                return (client (argv[2], argv[3]));

        fprintf(stderr, "Usage: pubsub %s|%s <URL> <ARG> ...\n",
            SERVER, CLIENT);
        return 1;
}
```

Blithely assumes message is ASCIIZ string. Real code should check it.

Compilation
```
gcc pubsub.c -lnng -o pubsub
```

Execution
```
./pubsub server ipc:///tmp/pubsub.ipc & server=$! && sleep 1
./pubsub client ipc:///tmp/pubsub.ipc client0 & client0=$!
./pubsub client ipc:///tmp/pubsub.ipc client1 & client1=$!
./pubsub client ipc:///tmp/pubsub.ipc client2 & client2=$!
sleep 5
kill $server $client0 $client1 $client2
```

Output
```
SERVER: PUBLISHING DATE Tue Jan  9 08:21:31 2018
SERVER: PUBLISHING DATE Tue Jan  9 08:21:32 2018
CLIENT (client2): RECEIVED Tue Jan  9 08:21:32 2018
CLIENT (client1): RECEIVED Tue Jan  9 08:21:32 2018
CLIENT (client0): RECEIVED Tue Jan  9 08:21:32 2018
SERVER: PUBLISHING DATE Tue Jan  9 08:21:33 2018
CLIENT (client0): RECEIVED Tue Jan  9 08:21:33 2018
CLIENT (client2): RECEIVED Tue Jan  9 08:21:33 2018
CLIENT (client1): RECEIVED Tue Jan  9 08:21:33 2018
SERVER: PUBLISHING DATE Tue Jan  9 08:21:34 2018
CLIENT (client2): RECEIVED Tue Jan  9 08:21:34 2018
CLIENT (client0): RECEIVED Tue Jan  9 08:21:34 2018
CLIENT (client1): RECEIVED Tue Jan  9 08:21:34 2018
SERVER: PUBLISHING DATE Tue Jan  9 08:21:35 2018
CLIENT (client0): RECEIVED Tue Jan  9 08:21:35 2018
CLIENT (client1): RECEIVED Tue Jan  9 08:21:35 2018
CLIENT (client2): RECEIVED Tue Jan  9 08:21:35 2018
SERVER: PUBLISHING DATE Tue Jan  9 08:21:36 2018
CLIENT (client0): RECEIVED Tue Jan  9 08:21:36 2018
CLIENT (client1): RECEIVED Tue Jan  9 08:21:36 2018
CLIENT (client2): RECEIVED Tue Jan  9 08:21:36 2018
```


Survey (Everybody Votes)
------------------------

[image]

The surveyor pattern is used to send a timed survey out, responses are individually returned until the survey has expired. This pattern is useful for service discovery and voting algorithms.

survey.c
```C
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>
#include <unistd.h>

#include <nng/nng.h>
#include <nng/protocol/survey0/survey.h>
#include <nng/protocol/survey0/respond.h>

#define SERVER "server"
#define CLIENT "client"
#define DATE   "DATE"

void
fatal(const char *func, int rv)
{
        fprintf(stderr, "%s: %s\n", func, nng_strerror(rv));
        exit(1);
}

char *
date(void)
{
        time_t now = time(&now);
        struct tm *info = localtime(&now);
        char *text = asctime(info);
        text[strlen(text)-1] = '\0'; // remove '\n'
        return (text);
}

int
server(const char *url)
{
        nng_socket sock;
        int rv;

        if ((rv = nng_surveyor0_open(&sock)) != 0) {
                fatal("nng_surveyor0_open", rv);
        }
        if ((rv = nng_listen(sock, url, NULL, 0)) != 0) {
                fatal("nng_listen", rv);
        }
        for (;;) {
                printf("SERVER: SENDING DATE SURVEY REQUEST\n");
                if ((rv = nng_send(sock, DATE, strlen(DATE) + 1, 0)) != 0) {
                        fatal("nng_send", rv);
                }

                for (;;) {
                        char *buf = NULL;
                        size_t sz;
                        rv = nng_recv(sock, &buf, &sz, NNG_FLAG_ALLOC);
                        if (rv == NNG_ETIMEDOUT) {
                                break;
                        }
                        if (rv != 0) {
                                fatal("nng_recv", rv);
                        }
                        printf("SERVER: RECEIVED \"%s\" SURVEY RESPONSE\n",
                            buf); 
                        nng_free(buf, sz);
                }

                printf("SERVER: SURVEY COMPLETE\n");
        }
}

int
client(const char *url, const char *name)
{
        nng_socket sock;
        int rv;

        if ((rv = nng_respondent0_open(&sock)) != 0) {
                fatal("nng_respondent0_open", rv);
        }
        if ((rv = nng_dial(sock, url, NULL, NNG_FLAG_NONBLOCK)) != 0) {
                fatal("nng_dial", rv);
        }
        for (;;) {
                char *buf = NULL;
                size_t sz;
                if ((rv = nng_recv(sock, &buf, &sz, NNG_FLAG_ALLOC)) == 0) {
                        printf("CLIENT (%s): RECEIVED \"%s\" SURVEY REQUEST\n",
                            name, buf); 
                        nng_free(buf, sz);
                        char *d = date();
                        printf("CLIENT (%s): SENDING DATE SURVEY RESPONSE\n",
                           name);
                        if ((rv = nng_send(sock, d, strlen(d) + 1, 0)) != 0) {
                                fatal("nng_send", rv);
                        }
                }
        }
}

int
main(const int argc, const char **argv)
{
        if ((argc >= 2) && (strcmp(SERVER, argv[1]) == 0))
                return (server(argv[2]));

        if ((argc >= 3) && (strcmp(CLIENT, argv[1]) == 0))
                return (client(argv[2], argv[3]));

        fprintf(stderr, "Usage: survey %s|%s <URL> <ARG> ...\n",
            SERVER, CLIENT);
        return 1;
}
```

Blithely assumes message is ASCIIZ string. Real code should check it.

Compilation
```
gcc survey.c -lnng -o survey
```

Execution
```
./survey server ipc:///tmp/survey.ipc & server=$!
./survey client ipc:///tmp/survey.ipc client0 & client0=$!
./survey client ipc:///tmp/survey.ipc client1 & client1=$!
./survey client ipc:///tmp/survey.ipc client2 & client2=$!
sleep 3 
kill $server $client0 $client1 $client2
```

The first survey times out with no responses because the clients aren’t connected yet.

Output
```
SERVER: SENDING DATE SURVEY REQUEST
SERVER: SURVEY COMPLETE
SERVER: SENDING DATE SURVEY REQUEST
CLIENT (client2): RECEIVED "DATE" SURVEY REQUEST
CLIENT (client0): RECEIVED "DATE" SURVEY REQUEST
CLIENT (client1): RECEIVED "DATE" SURVEY REQUEST
CLIENT (client2): SENDING DATE SURVEY RESPONSE
CLIENT (client1): SENDING DATE SURVEY RESPONSE
CLIENT (client0): SENDING DATE SURVEY RESPONSE
SERVER: RECEIVED "Tue Jan  9 11:33:40 2018" SURVEY RESPONSE
SERVER: RECEIVED "Tue Jan  9 11:33:40 2018" SURVEY RESPONSE
SERVER: RECEIVED "Tue Jan  9 11:33:40 2018" SURVEY RESPONSE
SERVER: SURVEY COMPLETE
SERVER: SENDING DATE SURVEY REQUEST
CLIENT (client2): RECEIVED "DATE" SURVEY REQUEST
CLIENT (client2): SENDING DATE SURVEY RESPONSE
CLIENT (client0): RECEIVED "DATE" SURVEY REQUEST
CLIENT (client0): SENDING DATE SURVEY RESPONSE
CLIENT (client1): RECEIVED "DATE" SURVEY REQUEST
CLIENT (client1): SENDING DATE SURVEY RESPONSE
SERVER: RECEIVED "Tue Jan  9 11:33:41 2018" SURVEY RESPONSE
SERVER: RECEIVED "Tue Jan  9 11:33:41 2018" SURVEY RESPONSE
SERVER: RECEIVED "Tue Jan  9 11:33:41 2018" SURVEY RESPONSE
```

Bus (Routing)
-------------

[image]

The bus protocol is useful for routing applications, or for building fully interconnected mesh networks. In this pattern, messages are sent to every directly connected peer.

bus.c
```C
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

#include <nng/nng.h>
#include <nng/protocol/bus0/bus.h>

void
fatal(const char *func, int rv)
{
        fprintf(stderr, "%s: %s\n", func, nng_strerror(rv));
        exit(1);
}

int
node(int argc, char **argv)
{
        nng_socket sock;
        int rv;
        size_t sz;

        if ((rv = nng_bus0_open(&sock)) != 0) {
                fatal("nng_bus0_open", rv);
        }
        if ((rv = nng_listen(sock, argv[2], NULL, 0)) != 0) {
                fatal("nng_listen", rv);
        }

        sleep(1); // wait for peers to bind
        if (argc >= 3) {
                for (int x = 3; x < argc; x++) {
                        if ((rv = nng_dial(sock, argv[x], NULL, 0)) != 0) {
                                fatal("nng_dial", rv);
                        }
                }
        }

        sleep(1); // wait for connects to establish

        // SEND
        sz = strlen(argv[1]) + 1; // '\0' too
        printf("%s: SENDING '%s' ONTO BUS\n", argv[1], argv[1]);
        if ((rv = nng_send(sock, argv[1], sz, 0)) != 0) {
                fatal("nng_send", rv);
        }

        // RECV
        for (;;) {
                char *buf = NULL;
                size_t sz;
                if ((rv = nng_recv(sock, &buf, &sz, NNG_FLAG_ALLOC)) !=0) {
                        if (rv == NNG_ETIMEDOUT) {
                                fatal("nng_recv", rv);
                        }
                }
                printf("%s: RECEIVED '%s' FROM BUS\n", argv[1], buf); 
                nng_free(buf, sz);
        }
        nng_close(sock);
        return (0);
}

int
main(int argc, char **argv)
{
        if (argc >= 3) {
                return (node(argc, argv));
        }
        fprintf(stderr, "Usage: bus <NODE_NAME> <URL> <URL> ...\n");
        return 1;
}
```

Blithely assumes message is ASCIIZ string. Real code should check it.

Compilation
```
gcc bus.c -lnng -o bus
```

Execution
```
./bus node0 ipc:///tmp/node0.ipc ipc:///tmp/node1.ipc ipc:///tmp/node2.ipc & node0=$!
./bus node1 ipc:///tmp/node1.ipc ipc:///tmp/node2.ipc ipc:///tmp/node3.ipc & node1=$!
./bus node2 ipc:///tmp/node2.ipc ipc:///tmp/node3.ipc & node2=$!
./bus node3 ipc:///tmp/node3.ipc ipc:///tmp/node0.ipc & node3=$!
sleep 5
kill $node0 $node1 $node2 $node3
```

Output
```
node3: SENDING 'node3' ONTO BUS
node0: SENDING 'node0' ONTO BUS
node1: SENDING 'node1' ONTO BUS
node2: SENDING 'node2' ONTO BUS
node0: RECEIVED 'node1' FROM BUS
node1: RECEIVED 'node0' FROM BUS
node2: RECEIVED 'node0' FROM BUS
node3: RECEIVED 'node1' FROM BUS
node0: RECEIVED 'node2' FROM BUS
node1: RECEIVED 'node2' FROM BUS
node3: RECEIVED 'node2' FROM BUS
node0: RECEIVED 'node3' FROM BUS
node2: RECEIVED 'node3' FROM BUS
node1: RECEIVED 'node3' FROM BUS
node3: RECEIVED 'node0' FROM BUS
node2: RECEIVED 'node1' FROM BUS
```
