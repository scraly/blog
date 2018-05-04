---
title: "Struggle With Librdkafka"
date: 2018-05-03T07:36:32+02:00
tags: [
    "librdkafka",
    "kafka",
    "golang"
]
categories: [
    "Golang",
]
---

The beginning
-----

For an event driven microservice project in golang that was supposed to communicate with the [Kafka](https://kafka.apache.org/) message broker, among other things, I decided to use the [confluent kafka go](https://github.com/confluentinc/confluent-kafka-go) client.
To connect to a Kafka and create a producer and / or a consumer, the grip is super fast. Perfect.
Ah ... but you have to use a [librdkafka](https://github.com/edenhill/librdkafka) library. Whats is that?


> librdkafka is the Apache Kafka C/C++ library. It's a C library implementation of the Apache Kafka protocol, containing both Producer and Consumer support. It was designed with message delivery reliability and high performance in mind, current figures exceed 1 million msgs/second for the producer and 3 million msgs/second for the consumer.

So I import the lib in my golang file, go get it and add it in my vendors:

```
import (
...
"github.com/confluentinc/confluent-kafka-go/kafka"
)
```

`$ go get -u github.com/confluentinc/confluent-kafka-go/kafka`

`$ dep ensure`

OK, I'm on Ubuntu, I'll just install it with my package manager:

`$ sudo apt install librdkafka-dev`

Great, i've got the lib installed, it should works ..

`$ go test`
...
Instead to have evrything working, an error tells me that I have a librdkafka installed in version 0.8.6 but I need to have a version >0.9.4!

Ouch. The installed version is below the required minimum.


At this point there was packet search which I then installed with dpkg.
For the record, librdkafka-dev has dependencies with librdkafka1-dbg and librdkafka1.
The problem was that I found a recent version of librdkafka-dev but not librdkafka1-dbg, which it depended on and I only uninstalled librdkafka-dev (by doing an apt remove librdkafka-dev). So I had a librdkafka-dev in 0.11.0, the version of librdkafka1 remained in 0.8.6 also ...


After a good night's sleep, I got off to a good start and I simply uninstalled any existing packages starting with librdkafka*, including the dependencies, and then downloaded and installed the binaries:

You need first, to remove installed librdkafka (included librdkafka-dev, librdkafka1, librdkafka1-dbg) :

`$ sudo apt remove librdkafka*`

Then simply download the binaries and untar it in your path, like this:

```
$ cd /opt
$ wget https://launchpad.net/ubuntu/+archive/primary/+files/librdkafka_0.11.0.orig.tar.gz
$ tar zxvf librdkafka_0.11.0.orig.tar.gz
$ rm librdkafka_0.11.0.orig.tar.gz
$ cd librdkafka-0.11.0/
```

Of course, in this example, `/opt` is in my $PATH ;-).

Now, it's time to test again:

```
$ go test
...
Message to send: ...
[Producer] Delivered message to myTopic
...

```

Yeah, no error! Only my logs telling me everything is working as expected!

Dockerfile
-----

In local environment, I can build, test and zip my golang application, everything it's working, it's good ... but now I need, through my CI/CD stack, to do the same thing in docker containers.

So I had to improve my docker image containing golang (based on debian stretch):

```
FROM golang:1.9

ENV LIBRDKAFKA_VERSION 0.11.4

RUN apt-get -y update \
        && apt-get install -y --no-install-recommends upx-ucl zip \
        && apt-get clean \
        && rm -rf /var/lib/apt/lists/*

RUN curl -Lk -o /root/librdkafka-${LIBRDKAFKA_VERSION}.tar.gz https://github.com/edenhill/librdkafka/archive/v${LIBRDKAFKA_VERSION}.tar.gz && \
      tar -xzf /root/librdkafka-${LIBRDKAFKA_VERSION}.tar.gz -C /root && \
      cd /root/librdkafka-${LIBRDKAFKA_VERSION} && \
      ./configure --prefix /usr && make && make install && make clean && ./configure --clean
```

And in a AWS Lambda Go?
-----

In local machine it's working, our CI/CD tool build, test, zip and deploy correctly the AWS lambda so now we need to test it in AWS:

![AWS Lambda Go error2](/images/librdkafka.png)

Oh no... Again the librdkafka is missing!

The solution is to build the golang binary with librdkafka as a static library. So we need to include `-tags static` in our go build command:

`$ go build -tags static -ldflags "${ldflags}" -o bin/main${ext} ${repo_path}`

That's it!
