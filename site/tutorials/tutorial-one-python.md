# RabbitMQ tutorial - "Hello world!" SUPPRESS-RHS

## Introduction

<xi:include href="site/tutorials/tutorials-help.xml.inc"/>
<xi:include href="site/tutorials/tutorials-intro.xml.inc"/>

Hello World!
------------
### (using the pika 0.10.0 Python client)

Our "Hello world" won't be too complex &#8210; let's send a message, receive
it and print it on the screen. To do so we need two programs: one that
sends a message and one that receives and prints it.


Our overall design will look like:

<div class="diagram">
  <img src="/img/tutorials/python-one-overall.png" height="100" />
  <div class="diagram_source">
digraph G {
      bgcolor=transparent;
      truecolor=true;
      rankdir=LR;
      node [style="filled"];
      //
      P1 [label="P", fillcolor="#00ffff"];
      subgraph cluster_Q1 {
        label="hello";
	color=transparent;
        Q1 [label="{||||}", fillcolor="red", shape="record"];
      };
      C1 [label="C", fillcolor="#33ccff"];
      //
      P1 -> Q1 -> C1;
}
</div>
</div>

Producer sends messages to the "hello" queue. The consumer receives
messages from that queue.

> #### RabbitMQ libraries

> RabbitMQ speaks AMQP 0.9.1, which is an open, general-purpose
> protocol for messaging. There are a number of clients for RabbitMQ
> in [many different languages](/devtools.html).  In this tutorial
> series we're going to use [Pika](https://pika.readthedocs.org/en/0.10.0/#),
> which is the Python client recommended
> by the RabbitMQ team. To install it you can use the
> [`pip`](https://pip.pypa.io/en/stable/quickstart/) package management tool.

### Sending

<div class="diagram">
  <img src="/img/tutorials/sending.png" height="100" />
  <div class="diagram_source">
  digraph {
      bgcolor=transparent;
      truecolor=true;
      rankdir=LR;
      node [style="filled"];
      //
      P1 [label="P", fillcolor="#00ffff"];
      subgraph cluster_Q1 {
        label="hello";
        color=transparent;
        Q1 [label="{||||}", fillcolor="red", shape="record"];
      };
      //
      P1 -> Q1;
  }
  </div>
</div>

Our first program `send.py` will send a single message to the queue.
The first thing we need to do is to establish a connection with
RabbitMQ server.

    :::python
    #!/usr/bin/env python
    import pika

    connection = pika.BlockingConnection(pika.ConnectionParameters(
                   'localhost'))
    channel = connection.channel()

We're connected now, to a broker on the local machine - hence the
_localhost_. If we wanted to connect to a broker on a different
machine we'd simply specify its name or IP address here.

Next, before sending we need to make sure the recipient queue
exists. If we send a message to non-existing location, RabbitMQ will
just trash the message. Let's create a queue to which the message will
be delivered, let's name it _hello_:

    :::python
    channel.queue_declare(queue='hello')

At that point we're ready to send a message. Our first message will
just contain a string _Hello World!_ and we want to send it to our
_hello_ queue.

In RabbitMQ a message can never be sent directly to the queue, it always
needs to go through an _exchange_. But let's not get dragged down by the
details &#8210; you can read more about _exchanges_ in [the third part of this
tutorial](tutorial-three-python.html). All we need to know now is how to use a default exchange
identified by an empty string. This exchange is special &#8210; it
allows us to specify exactly to which queue the message should go.
The queue name needs to be specified in the `routing_key` parameter:

    :::python
    channel.basic_publish(exchange='',
                          routing_key='hello',
                          body='Hello World!')
    print(" [x] Sent 'Hello World!'")


Before exiting the program we need to make sure the network buffers
were flushed and our message was actually delivered to RabbitMQ. We
can do it by gently closing the connection.

    :::python
    connection.close()

> #### Sending doesn't work!
>
> If this is your first time using RabbitMQ and you don't see the "Sent"
> message then you may be left scratching your head wondering what could
> be wrong. Maybe the broker was started without enough free disk space
> (by default it needs at least 1Gb free) and is therefore refusing to
> accept messages. Check the broker logfile to confirm and reduce the
> limit if necessary. The <a
> href="http://www.rabbitmq.com/configure.html#config-items">configuration
> file documentation</a> will show you how to set <code>disk_free_limit</code>.


### Receiving

<div class="diagram">
  <img src="/img/tutorials/receiving.png" height="100" />
  <div class="diagram_source">
  digraph {
      bgcolor=transparent;
      truecolor=true;
      rankdir=LR;
      node [style="filled"];
      //
      subgraph cluster_Q1 {
        label="hello";
	color=transparent;
	Q1 [label="{||||}", fillcolor="red", shape="record"];
      };
      C1 [label="C", fillcolor="#33ccff"];
      //
      Q1 -> C1;
  }
  </div>
</div>


Our second program `receive.py` will receive messages from the queue and print
them on the screen.

Again, first we need to connect to RabbitMQ server. The code
responsible for connecting to Rabbit is the same as previously.

The next step, just like before, is to make sure that the queue
exists. Creating a queue using `queue_declare` is idempotent &#8210; we
can run the command as many times as we like, and only one will be
created.

    :::python
    channel.queue_declare(queue='hello')

You may ask why we declare the queue again &#8210; we have already declared it
in our previous code. We could avoid that if we were sure
that the queue already exists. For example if `send.py` program was
run before. But we're not yet sure which
program to run first. In such cases it's a good practice to repeat
declaring the queue in both programs.

> #### Listing queues
>
> You may wish to see what queues RabbitMQ has and how many
> messages are in them. You can do it (as a privileged user) using the `rabbitmqctl` tool:
>
>     :::bash
>     $ sudo rabbitmqctl list_queues
>     Listing queues ...
>     hello    0
>     ...done.
>
>(omit sudo on Windows)


Receiving messages from the queue is more complex. It works by subscribing
a `callback` function to a queue. Whenever we receive
a message, this `callback` function is called by the Pika library.
In our case this function will print on the screen the contents of
the message.

    :::python
    def callback(ch, method, properties, body):
        print(" [x] Received %r" % body)


Next, we need to tell RabbitMQ that this particular callback function should
receive messages from our _hello_ queue:

    :::python
    channel.basic_consume(callback,
                          queue='hello',
                          no_ack=True)

For that command to succeed we must be sure that a queue which we want
to subscribe to exists. Fortunately we're confident about that &#8210; we've
created a queue above &#8210; using `queue_declare`.

The `no_ack` parameter will be described [later on](tutorial-two-python.html).

And finally, we enter a never-ending loop that waits for data and runs callbacks
whenever necessary.

    :::python
    print(' [*] Waiting for messages. To exit press CTRL+C')
    channel.start_consuming()


### Putting it all together


Full code for `send.py`:

    #!/usr/bin/env python
    import pika

    connection = pika.BlockingConnection(pika.ConnectionParameters(
            host='localhost'))
    channel = connection.channel()


    channel.queue_declare(queue='hello')

    channel.basic_publish(exchange='',
                          routing_key='hello',
                          body='Hello World!')
    print(" [x] Sent 'Hello World!'")
    connection.close()

[(send.py source)](http://github.com/rabbitmq/rabbitmq-tutorials/blob/master/python/send.py)


Full `receive.py` code:

    #!/usr/bin/env python
    import pika

    connection = pika.BlockingConnection(pika.ConnectionParameters(
            host='localhost'))
    channel = connection.channel()


    channel.queue_declare(queue='hello')

    def callback(ch, method, properties, body):
        print(" [x] Received %r" % body)

    channel.basic_consume(callback,
                          queue='hello',
                          no_ack=True)

    print(' [*] Waiting for messages. To exit press CTRL+C')
    channel.start_consuming()

[(receive.py source)](http://github.com/rabbitmq/rabbitmq-tutorials/blob/master/python/receive.py)

Now we can try out our programs in a terminal. First, let's send a
message using our `send.py` program:

     :::bash
     $ python send.py
     [x] Sent 'Hello World!'

The producer program `send.py` will stop after every run. Let's receive it:

     :::bash
     $ python receive.py
     [*] Waiting for messages. To exit press CTRL+C
     [x] Received 'Hello World!'

Hurray! We were able to send our first message through RabbitMQ. As you might
have noticed, the `receive.py` program doesn't exit. It will stay ready to
receive further messages, and may be interrupted with Ctrl-C.

Try to run `send.py` again in a new terminal.

We've learned how to send and receive a message from a named
queue. It's time to move on to [part 2](tutorial-two-python.html)
and build a simple _work queue_.
