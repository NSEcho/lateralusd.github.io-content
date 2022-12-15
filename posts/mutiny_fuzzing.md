---
title : "mutiny fuzzer"
description : "Fuzzing the network with mutiny fuzzer"
date : 2021-10-19T09:29:40+02:00
---

## Introduction

[mutiny](https://github.com/Cisco-Talos/mutiny-fuzzer) is a network fuzzer which uses radamsa to generate mutations. It is really simple to use, either with the captured pcap file or with the [decept](https://github.com/Cisco-Talos/Decept) proxy.

In this post we will be looking at the captured pcap file and how to utilize mutiny with it.

Basic steps for fuzzing with mutiny:
* Setup(prepare pcap file or utilize decept)
	* In case of pcap, prepare the fuzzing file
* Modify fuzzer file
* Start the mutiny fuzzer
* Wait and hope

## Vulnerable binary
We are gonna use simple go program with provides us with the HTTP server. Program expects the user to provide it with the id in request, which will then be used to access the certain element from the array.

### Basic communication

```bash
$ go run main.go
2021/10/19 09:46:21 Connection from [::1]:61819
2021/10/19 09:46:21 Got request for id 15
2021/10/19 09:46:21 got: element15
```

```bash
$ curl 'http://localhost:8090/?id=15'
element15
```

### Setup
I will use the tcpdump in order to generate the pcap file.

```bash
$ sudo tcpdump -i lo0 -v -p -w vuln.pcap port 8090
```

In another tab in terminal, let's generate some traffic by making the request to the 'http://localhost:8089/?id=15'. After we are done you should see the message inside the tcpdump `Got 12` or some other number, if you see `Got 0` you did something wrong and make sure to fix your tcpdump capturing.

After we captured it, we can just stop the tcpdump and move to the next stage.


### Preparing the fuzzing file

Copy the `vuln.pcap` file inside the mutiny directory.

Call the `mutiny_prep` script with the generated pcap file and answer the default questions, later on you can focus on each one individually but for now defaults are ok.

```bash
$ cp /tmp/vuln.pcap ~/mutiny_fuzzer/
$ python mutiny_prep vuln.pcap
[ ... REDACTED ... ]

Wrote .fuzzer file: vuln-0.fuzzer


Do you want to generate a .fuzzer for another message number? (y/n)
Default n:

All files have been written.
```

We can see that the file `vuln-0.fuzzer` has been created which will we modify in the next stage.

## Modify .fuzzer file

Once you open up the .fuzzer file in your text editor, at the top you will see some options, you can modify them later if you want. We will focus on the actual message in the fuzzer file.

My fuzzer file looks like this:

```bash
$ cat vuln-0.fuzzer
# Directory containing any custom exception/message/monitor processors
# This should be either an absolute path or relative to the .fuzzer file
# If set to "default", Mutiny will use any processors in the same
# folder as the .fuzzer file
processor_dir default
# Number of times to retry a test case causing a crash
failureThreshold 3
# How long to wait between retrying test cases causing a crash
failureTimeout 5
# How long for recv() to block when waiting on data from server
receiveTimeout 1.0
# Whether to perform an unfuzzed test run before fuzzing
shouldPerformTestRun 1
# Protocol (udp or tcp)
proto tcp
# Port number to connect to
port 8090
# Port number to connect from
sourcePort -1
# Source IP to connect from
sourceIP 0.0.0.0

# The actual messages in the conversation
# Each contains a message to be sent to or from the server, printably-formatted
outbound fuzz 'GET /?id=15 HTTP/1.1\r\nHost: localhost:8090\r\nUser-Agent: curl/7.64.1\r\nAccept: */*\r\n\r\n'
inbound 'HTTP/1.1 200 OK\r\nDate: Tue, 19 Oct 2021 07:51:00 GMT\r\nContent-Length: 9\r\nContent-Type: text/plain; charset=utf-8\r\n\r\nelement15'
```

Our goal is to fuzz the actual number that gets sent to the server, but in order to do so we need to take a look at the actual commands inside the fuzzer file.

### Fuzzer commands

* outbound - message is outbound, from the client to the server
* inbound - what is expected from the server
* sub - when you want to break certain message in multiple lines
* fuzz - which part to fuzz

So in order to fuzz this actual message, we will use this:

```
outbound 'GET /?id='
sub fuzz '15'
sub ' HTTP/1.1\r\nHost: localhost:8090\r\nUser-Agent: curl/7.64.1\r\nAccept: */*\r\n\r\n'
inbound 'HTTP/1.1 200 OK\r\nDate: Tue, 19 Oct 2021 07:51:00 GMT\r\nContent-Length: 9\r\nContent-Type: text/plain; charset=utf-8\r\n\r\nelement15'
```

Modify the fuzzer file like I did above and it is now time to start the actual fuzzing.

## Start the mutiny fuzzer

Use the format: `python mutiny.py <fuzzerFile> <serverAddr>`

In our case, it is gonna be:

```bash
$ python mutiny.py --logAll vuln-0.fuzzer 127.0.0.1
Performing test run without fuzzing...
	Sent 84 byte packet
	Received 125 bytes
Logging run number -1

** Sleeping for 0.000 seconds **


Fuzzing with seed 0
	Sent 84 byte packet
	Received 193 bytes
Logging run number 0

** Sleeping for 0.000 seconds **


Fuzzing with seed 1
	Sent 121 byte packet
Logging run number 1
Server has closed the connection
Run aborted: Server closed connection: Server has closed the connection

** Sleeping for 0.000 seconds **


Fuzzing with seed 2
Logging run number 2
[Errno 61] Connection refused
Received LogLastAndHaltException, logging last run and halting
Logging run number 1
```

Immediately after we have run it, we can see below that we got `Connection refused`, in network fuzzing situations that is actually a good sign, meaning that you crashed the service.

If we take a look at the vulnerable binary running, we can see what is going on.

```
[ ... REDACTED ... ]
2021/10/19 10:09:34 Connection from 127.0.0.1:62181
2021/10/19 10:09:34 Got request for id 340282366920938463463374607431768211457
panic: runtime error: index out of range [9223372036854775807] with length 1000

goroutine 1 [running]:
main.main()
	/Users/daemon1/tools/wrong_srv/main.go:31 +0x1b5
```

We can see that we caused the out of bounds access for the array. But we won't always have the output of the fuzzed binary, we will only have information is it alive or not so how could we know what is the actual payload that caused this to happen. The answer lies in the logs file of mutiny.

If we take a look at the mutiny output we can see that the last seed used was 2, so  inside `vuln-0_logs` we should find the value of this seed.

```bash
$ cat vuln-0_logs/2021-10-19,100934/2
Log from run with seed 2
Error message: LogAll
Failed to connect on this run.

Packet 0: outbound fuzz 'GET /?id=15 HTTP/1.1\r\nHost: localhost:8090\r\nUser-Agent: curl/7.64.1\r\nAccept: */*\r\n\r\n'
Fuzzed Packet 0: fuzz outbound 'GET /?id=340282366920938463463374607431768211457 HTTP/1.1\r\nHost: localhost:8090\r\nUser-Agent: curl/7.64.1\r\nAccept: */*\r\n\r\n'


Packet 1: inbound 'HTTP/1.1 200 OK\r\nDate: Tue, 19 Oct 2021 07:51:00 GMT\r\nContent-Length: 9\r\nContent-Type: text/plain; charset=utf-8\r\n\r\nelement15'
```

We can see that inside the file, we can see the line __Fuzzed Packet 0:__ wit the line __GET /?id=340282366920938463463374607431768211457__. So, in order to replicate this, we can use the same id.

Let's start the server again, and issue the request to this id.

```bash
$ # server
$ go run main.go
2021/10/19 10:12:44 Connection from [::1]:62229
2021/10/19 10:12:44 Got request for id 340282366920938463463374607431768211457
panic: runtime error: index out of range [9223372036854775807] with length 1000

goroutine 1 [running]:
main.main()
	/Users/daemon1/tools/wrong_srv/main.go:31 +0x1b5
```

```bash
$ # request
$ curl 'http://localhost:8090/?id=340282366920938463463374607431768211457'
curl: (52) Empty reply from server
```

We can see that we crashed the server successfully and that should be it. Using this simple fuzzing example, you can know fuzz all kinds of service. Decept is also worth to look at, as it can immediately generate the fuzzer file.
