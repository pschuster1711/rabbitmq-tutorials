

Learning RabbitMQ, part 2 (Task Queue)
======================================



<center><div class="dot_bitmap">
<img src="http://github.com/rabbitmq/rabbitmq-tutorials/raw/master/_img/378bf43b3d9dcc790d2113257412c928.png" alt="Dot graph" width="376" height="125" />
</div></center>



In the [first part of this tutorial](http://github.com/rabbitmq/rabbitmq-tutorials/blob/master/python/tutorial-one.md) we've learned how
to send and receive messages from a named queue. In this part we'll create a
_Task Queue_ that will be used to distribute time-consuming work across multiple
workers.

The main idea behind Task Queues (aka: _Work Queues_) is to avoid
doing resource intensive tasks immediately. Instead we schedule a task to
be done later on. We encapsulate a _task_ as a message and save it to
the queue. A worker process running in a background will pop the tasks
and eventually execute the job. When you run many workers the work will
be shared between them.

This concept is especially useful in web applications where it's
impossible to handle a complex task during a short http request
window.


Preparations
------------

In previous part of this tutorial we were sending a message containing
"Hello World!" string. Now we'll be sending strings that
stand for complex tasks. We don't have any real hard tasks, like
images to be resized or pdf files to be rendered, so let's fake it by just
pretending we're busy - by using `time.sleep()` function. We'll take
the number of dots in the string as a complexity, every dot will
account for one second of "work".  For example, a fake task described
by `Hello...` string will take three seconds.

We need to slightly modify the _send.py_ code from previous example,
to allow sending arbitrary messages from the command line. This program
will schedule tasks to our work queue, so let's name it `new_task.py`:

<div><pre><code class='python'>import sys
message = ' '.join(sys.argv[1:]) or &quot;Hello World!&quot;
channel.basic_publish(exchange='', routing_key='test',
                      body=message)
print &quot; [x] Sent %r&quot; % (message,)</code></pre></div>



Our old _receive.py_ script also requires some changes: it needs to fake a
second of work for every dot in the message body. It will pop messages
from the queue and do the task, so let's call it `worker.py`:

<div><pre><code class='python'>import time

def callback(ch, method, header, body):
    print &quot; [x] Received %r&quot; % (body,)
	time.sleep( body.count('.') )
    print &quot; [x] Done&quot;</code></pre></div>



Round-robin dispatching
-----------------------

One of the advantages of using Task Queue is the
ability to easily parallelize work. If we have too much work for us to
handle, we can just add more workers and scale easily.

First, let's try to run two `worker.py` scripts at the same time. They
will both get messages from the queue, but how exactly? Let's
see.

You need three consoles open. First two to run `worker.py`
script. These consoles will be our two consumers - C1 and C2.

    shell1$ python worker.py
     [*] Waiting for messages. To exit press CTRL+C

    shell2$ python worker.py
     [*] Waiting for messages. To exit press CTRL+C

On the
third one we'll be publishing new tasks. Once you've started the
consumers you can produce few messages:

    shell3$ python new_task.py First message.
    shell3$ python new_task.py Second message..
    shell3$ python new_task.py Third message...
    shell3$ python new_task.py Fourth message....
    shell3$ python new_task.py Fifth message.....

Let's see what is delivered to our workers:

    shell1$ python worker.py
     [*] Waiting for messages. To exit press CTRL+C
     [x] Received 'First message.'
     [x] Received 'Third message...'
     [x] Received 'Fifth message.....'

    shell2$ python worker.py
     [*] Waiting for messages. To exit press CTRL+C
     [x] Received 'Second message..'
     [x] Received 'Fourth message....'

By default, RabbitMQ will send every n-th message to a consumer. That
way on average every consumer will get the same number of
messages. This way of distributing messages is called round-robin. Try
this out with three or more workers.

Message acknowledgments
-----------------------

Doing our tasks can take a few seconds. You may wonder what happens if
one of the consumers got hard job and has died while doing it. With
our current code once RabbitMQ delivers message to the customer it
immediately removes it from memory. In our case if you kill a worker
we will lose the message it was just processing. We'll also lose all
the messages that were dispatched to this particular worker and not
yet handled.

But we don't want to lose any task. If a workers dies, we'd like the task
to be delivered to another worker.

In order to make sure a message is never lost, RabbitMQ
supports message _acknowledgments_. It's a bit of data,
sent back from the consumer which tells Rabbit that particular message
had been received, fully processed and that Rabbit is free to delete
it.

If consumer dies without sending ack, Rabbit will understand that a
message wasn't processed fully and will redelivered it to another
consumer. That way you can be sure that no message is lost, even if
the workers occasionally die.

There aren't any message timeouts; Rabbit will redeliver the
message only when the worker connection dies. It's fine even if processing
a message takes a very very long time.


Message acknowledgments are turned on by default. Though, in previous
examples we had explicitly turned them off via `no_ack=True` flag. It's time
to remove this flag and send a proper acknowledgment from the worker,
once we're done with a task.

<div><pre><code class='python'>def callback(ch, method, header, body):
    print &quot; [x] Received %r&quot; % (body,)
	time.sleep( body.count('.') )
    print &quot; [x] Done&quot;
    ch.basic_ack(delivery_tag = method.delivery_tag)

channel.basic_consume(callback,
                      queue='test')</code></pre></div>


Using that code we may be sure that even if you kill a worker using
CTRL+C while it was processing a message, nothing  will be lost. Soon
after the worker dies all unacknowledged messages will be redelivered.

> #### Forgotten acknowledgment
>
> It's a pretty common mistake to miss `basic_ack`. It's an easy error,
> but the consequences are serious. Messages will be redelivered
> when your client quits (which may look like random redelivery),
> Rabbit will eat more and more memory as it won't be able to release
> any unacked messages.
>
> In order to debug this kind of mistake you can use `rabbitmqctl`
> to print the `messages_unacknowledged` field:
>
>     $ sudo rabbitmqctl list_queues name messages_ready messages_unacknowledged
>     Listing queues ...
>     test    0       0
>     ...done.

Message durability
------------------

We learned what to do to make sure that even if the consumer dies, the
task isn't lost. But our tasks will still be lost if RabbitMQ server
dies.

When RabbitMQ quits or crashes it will forget the queues and messages
unless you tell it not to. Two things are required to make sure that
messages aren't lost: we need to mark both a queue and messages as durable.

First, we need to make sure that Rabbit will never lose our `test`
queue. In order to do so, we need to declare it as _durable_:

    channel.queue_declare(queue='test', durable=True)

Although that command is correct by itself it won't work in our
setup. That's because we've already defined a queue called `test`
which is not durable. RabbitMQ doesn't allow you to redefine a queue
with different parameters and will return hard error to any program
that tries to do that. But there is a quick workaround - let's
declare a queue with different name, for example `task_queue`:

    channel.queue_declare(queue='task_queue', durable=True)

This `queue_declare` change needs to be applied to both the producer
and consumer code.

At that point we're sure that the `task_queue` queue won't be lost even
if RabbitMQ dies. Now we need to mark our messages as persistent - by
supplying a `delivery_mode` header with a value `2`.

    channel.basic_publish(exchange='', routing_key="task_queue",
                          body=message,
                          properties=pika.BasicProperties(
                             delivery_mode = 2, # make message persistent
                          ))

> #### Note on message persistence
>
> Marking messages as persistent doesn't really guarantee that a message
> won't be lost. Although it tells Rabbit to save message to the disk,
> there is still a short time window when Rabbit accepted a message and
> haven't saved it yet. Also, Rabbit doesn't do `fsync(2)` for every
> message - it may be just saved to caches and not really written to the
> disk. The persistence guarantees aren't strong, but it's more than enough
> for our simple task queue. If you need stronger guarantees you can wrap the
> publishing code in a _transaction_.


Fair dispatching
----------------

You might have noticed that the dispatching still doesn't work exactly
as we want to. For example in a situation with two workers, when all
odd messages are heavy and even messages are light, one worker will be
constantly busy and the other one will do hardly any work. Well,
Rabbit doesn't know anything about that and will still dispatch
messages evenly.

That happens because Rabbit dispatches message just when a message
enters the queue. It doesn't look at the number of unacknowledged messages
for a consumer. It just blindly dispatches every n-th message to a
the n-th consumer.


<center><div class="dot_bitmap">
<img src="http://github.com/rabbitmq/rabbitmq-tutorials/raw/master/_img/e7e7b4568fed280746fc83b370cd60ea.png" alt="Dot graph" width="414" height="125" />
</div></center>


In order to defeat that we may use `basic.qos` method with the
`prefetch_count=1` settings. That allows us to tell Rabbit not to give
more than one message to a worker at a time. Or, in other words, don't
dispatch a new message to a worker until it has processed and
acknowledged the previous one.

<div><pre><code class='python'>channel.basic_qos(prefetch_count=1)</code></pre></div>



Putting it all together
-----------------------

Final code of our `new_task.py` script:

<div><pre><code class='python'>#!/usr/bin/env python
import pika
import sys

connection = pika.AsyncoreConnection(pika.ConnectionParameters(
        host='127.0.0.1',
        credentials=pika.PlainCredentials('guest', 'guest')))
channel = connection.channel()

channel.queue_declare(queue='task_queue', durable=True)

message = ' '.join(sys.argv[1:]) or &quot;Hello World!&quot;
channel.basic_publish(exchange='', routing_key='task_queue',
                      body=message,
                      properties=pika.BasicProperties(
                         delivery_mode = 2, # make message persistent
                      ))
print &quot; [x] Sent %r&quot; % (message,)</code></pre></div>

[(new_task.py source)](http://github.com/rabbitmq/rabbitmq-tutorials/blob/master/python/new_task.py)


And our worker:

<div><pre><code class='python'>#!/usr/bin/env python
import pika
import time

connection = pika.AsyncoreConnection(pika.ConnectionParameters(
        host='127.0.0.1',
        credentials=pika.PlainCredentials('guest', 'guest')))
channel = connection.channel()

channel.queue_declare(queue='task_queue', durable=True)
print ' [*] Waiting for messages. To exit press CTRL+C'

def callback(ch, method, header, body):
    print &quot; [x] Received %r&quot; % (body,)
    time.sleep( body.count('.') )
    print &quot; [x] Done&quot;
    ch.basic_ack(delivery_tag = method.delivery_tag)

channel.basic_qos(prefetch_count=1)
channel.basic_consume(callback,
                      queue='task_queue')

pika.asyncore_loop()</code></pre></div>

[(worker.py source)](http://github.com/rabbitmq/rabbitmq-tutorials/blob/master/python/worker.py)


Using message acknowledgments and `prefetch_count` you may set up
quite a decent work queue. The durability options let the tasks
to survive even if Rabbit is restarted.

Now we can move on to [part 3](http://github.com/rabbitmq/rabbitmq-tutorials/blob/master/python/tutorial-three.md) and learn how to
deliver the same message to many consumers.