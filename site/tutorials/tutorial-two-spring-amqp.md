# RabbitMQ tutorial - Work Queues SUPPRESS-RHS

<!--
Copyright (c) 2007-2016 Pivotal Software, Inc.

All rights reserved. This program and the accompanying materials
are made available under the terms of the under the Apache License,
Version 2.0 (the "License”); you may not use this file except in compliance
with the License. You may obtain a copy of the License at

http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
-->

## Work Queues

### (using the spring-amqp client)

<xi:include href="site/tutorials/tutorials-help.xml.inc"/>

<div class="diagram">
  <img src="/img/tutorials/python-two.png" height="110" />
  <div class="diagram_source">
    digraph {
      bgcolor=transparent;
      truecolor=true;
      rankdir=LR;
      node [style="filled"];
      //
      P1 [label="P", fillcolor="#00ffff"];
      Q1 [label="{||||}", fillcolor="red", shape="record"];
      C1 [label=&lt;C&lt;font point-size="7"&gt;1&lt;/font&gt;&gt;, fillcolor="#33ccff"];
      C2 [label=&lt;C&lt;font point-size="7"&gt;2&lt;/font&gt;&gt;, fillcolor="#33ccff"];
      //
      P1 -&gt; Q1 -&gt; C1;
      Q1 -&gt; C2;
    }
  </div>
</div>

In the [first tutorial](tutorial-one-spring-amqp.html) we
wrote programs to send and receive messages from a named queue. In this
one we'll create a _Work Queue_ that will be used to distribute
time-consuming tasks among multiple workers.

The main idea behind Work Queues (aka: _Task Queues_) is to avoid
doing a resource-intensive task immediately and having to wait for
it to complete. Instead we schedule the task to be done later. We encapsulate a
_task_ as a message and send it to a queue. A worker process running
in the background will pop the tasks and eventually execute the
job. When you run many workers the tasks will be shared between them.

This concept is especially useful in web applications where it's
impossible to handle a complex task during a short HTTP request
window.

### Preparation

In the previous part of this tutorial we sent a message containing
"Hello World!". Now we'll be sending strings that stand for complex
tasks. We don't have a real-world task, like images to be resized or
pdf files to be rendered, so let's fake it by just pretending we're
busy - by using the `Thread.sleep()` function. We'll take the number of dots
in the string as its complexity; every dot will account for one second
of "work".  For example, a fake task described by `Hello...`
will take three seconds.

Please see the setup in [first tutorial](tutorial-one-spring-amqp.html)
if you have not setup the project. We will follow the same pattern
as in the first tutorial: 1) create a package (tut2) and create
a Tut2Config, Tut2Receiver, and Tut2Sender. Start by create a new
package (tut2) where we'll place our three classes.  In the configuration
class we setup two profiles, the label for the tutorial ("tut2") and
the name of the pattern ("work-queues").  We leverage spring to expose
the queue as a bean. We setup the receiver as a profile and define two beans
to correspond to the workers in our diagram above; receiver1 and
receiver2. Finally, we define a profile for the sender and define the
sender bean.  The configuration is now done.

<pre class="sourcecode java">

import org.springframework.amqp.core.Queue;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Profile;

@Profile({"tut2", "work-queues"})
@Configuration
public class Tut2Config {

    @Bean
    public Queue hello() {
        return new Queue("hello");
    }

    @Profile("receiver")
    private static class ReceiverConfig {

        @Bean
        public Tut2Receiver receiver1() {
            return new Tut2Receiver(1);
        }

        @Bean
        public Tut2Receiver receiver2() {
            return new Tut2Receiver(2);
        }
    }

    @Profile("sender")
    @Bean
    public Tut2Sender sender() {
        return new Tut2Sender();
    }
}
</pre>

### Sender

We will modify the sender to provide a means for identifying
whether its a longer running task by appending a dot to the
message in a very contrived fashion using the same method
on the RabbitTemplate to publish the message, convertAndSend.
The documentation defines this as, "Convert a Java object to
an Amqp Message and send it to a default exchange with a
default routing key."

<pre class="sourcecode java">
import org.springframework.amqp.core.Queue;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.scheduling.annotation.Scheduled;


public class Tut2Sender {

    @Autowired
    private RabbitTemplate template;

    @Autowired
    private Queue queue;

    int dots = 0;
    int count = 0;

    @Scheduled(fixedDelay = 1000, initialDelay = 500)
    public void send() {
        StringBuilder builder = new StringBuilder("Hello");
		if (dots++ == 3) {
			dots = 1;
		}
       for (int i = 0; i &lt; dots; i++) {
            builder.append('.');
        }

        builder.append(Integer.toString(++count));
        String message = builder.toString();
        template.convertAndSend(queue.getName(), message);
        System.out.println(" [x] Sent '" + message + "'");
    }
}

</pre>

### Receiver

Our receiver, Tut2Receiver, simulates an arbitary length for
a fake task in the doWork() method where the number of dots
translates into the number of seconds the work will take. Again,
we leverage a @RabbitListener on the "hello" queue and a
@RabbitHandler to receive the message. The instance that is
consuming the message is added to our monitor to show
which instance, the message and the length of time to process
the message.

<pre class="sourcecode java">
import org.springframework.amqp.rabbit.annotation.RabbitHandler;
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.util.StopWatch;

@RabbitListener(queues = "hello")
public class Tut2Receiver {

    private final int instance;

    public Tut2Receiver(int i) {
        this.instance = i;
    }

    @RabbitHandler
    public void receive(String in) throws InterruptedException {
        StopWatch watch = new StopWatch();
        watch.start();
        System.out.println("instance " + this.instance +
            " [x] Received '" + in + "'");
        doWork(in);
        watch.stop();
        System.out.println("instance " + this.instance +
            " [x] Done in " + watch.getTotalTimeSeconds() + "s");
    }

    private void doWork(String in) throws InterruptedException {
        for (char ch : in.toCharArray()) {
            if (ch == '.') {
                Thread.sleep(1000);
            }
        }
    }
}
</pre>

### Putting it all together

Compile them using mvn package and run with the following options

<pre class="sourcecode bash">
mvn clean package

java -jar target/rabbitmq-amqp-tutorials-0.0.1-SNAPSHOT.jar --spring.profiles.active=work-queues,receiver
java -jar target/rabbitmq-amqp-tutorials-0.0.1-SNAPSHOT.jar --spring.profiles.active=work-queues,sender
</pre>

The output of the sender should look something like:

<pre class="sourcecode bash"> 
Ready ... running for 10000ms
 [x] Sent 'Hello.1'
 [x] Sent 'Hello..2'
 [x] Sent 'Hello...3'
 [x] Sent 'Hello.4'
 [x] Sent 'Hello..5'
 [x] Sent 'Hello...6'
 [x] Sent 'Hello.7'
 [x] Sent 'Hello..8'
 [x] Sent 'Hello...9'
 [x] Sent 'Hello.10'
</pre>

And the output from the workers should look something like:

<pre class="sourcecode bash">
Ready ... running for 10000ms
instance 1 [x] Received 'Hello.1'
instance 2 [x] Received 'Hello..2'
instance 1 [x] Done in 1.001s
instance 1 [x] Received 'Hello...3'
instance 2 [x] Done in 2.004s
instance 2 [x] Received 'Hello.4'
instance 2 [x] Done in 1.0s
instance 2 [x] Received 'Hello..5'
</pre>


### Message acknowledgment

Doing a task can take a few seconds. You may wonder what happens if
one of the consumers starts a long task and dies with it only partly done.
Spring-amqp by default takes a conservative approach to message
acknowledgement. If the listener throws an exception the container
calls:

<pre class="sourcecode java">
channel.basicReject(deliveryTag, requeue)
</pre>

Requeue is true by default unless you explicitly set:

<pre class="sourcecode java">
defaultRequeueRejected=false
</pre>

or the listener throws an `AmqpRejectAndDontRequeueException`. This
is typically the bahavior you want from your listener. In this mode
there is no need to worry about a forgotten acknowledgement.  After
processing the message the listener calls:
 
<pre class="sourcecode java">
channel.basicAck()
</pre>

> #### Forgotten acknowledgment
>
> It's a common mistake to miss the `basicAck` and spring-amqp
> helps to avoid this through its default configuraiton.
> The consequences are serious. Messages will be redelivered
> when your client quits (which may look like random redelivery), but
> RabbitMQ will eat more and more memory as it won't be able to release
> any unacked messages.
>
> In order to debug this kind of mistake you can use `rabbitmqctl`
> to print the `messages_unacknowledged` field:
>
> <pre class="sourcecode bash">
> sudo rabbitmqctl list_queues name messages_ready messages_unacknowledged
> </pre>
>
> On Windows, drop the sudo:
> <pre class="sourcecode bash">
> rabbitmqctl.bat list_queues name messages_ready messages_unacknowledged
> </pre>

### Message durability

With spring-amqp there are reasonable default values in the MessageProperties
that account for message durability.  In particular you can check the table 
for [common properties](http://docs.spring.io/spring-amqp/reference/htmlsingle/#_common_properties)
You'll see two relevant to our discussion here on durability:

<pre class="sourcecode java">
Property | default | description
durable  | true    | When declareExchange is true the durable flag is set to this value.
deliveryMode | PERSISTENT | PERSISTENT or NON_PERSISTENT to determine whether or not 
                            RabbitMQ should persist the messages.
</pre>

> #### Note on message persistence
>
> Marking messages as persistent doesn't fully guarantee that a message
> won't be lost. Although it tells RabbitMQ to save the message to disk,
> there is still a short time window when RabbitMQ has accepted a message and
> hasn't saved it yet. Also, RabbitMQ doesn't do `fsync(2)` for every
> message -- it may be just saved to cache and not really written to the
> disk. The persistence guarantees aren't strong, but it's more than enough
> for our simple task queue. If you need a stronger guarantee then you can use
> [publisher confirms](https://www.rabbitmq.com/confirms.html).

### Fair dispatch vs Round-robin dispatching
                     
By default, RabbitMQ will send each message to the next consumer,
in sequence. On average every consumer will get the same number of
messages. This way of distributing messages is called round-robin.
In this mode dispatching doesn't necessarily work exactly as we want.
For example in a situation with two workers, when all
odd messages are heavy and even messages are light, one worker will be
constantly busy and the other one will do hardly any work. Well,
RabbitMQ doesn't know anything about that and will still dispatch
messages evenly.

This happens because RabbitMQ just dispatches a message when the message
enters the queue. It doesn't look at the number of unacknowledged
messages for a consumer. It just blindly dispatches every n-th message
to the n-th consumer.

However, "Fair dispatch" is the default configuration for spring-amqp. The
SimpleMessageListenerContainer defines the value for
DEFAULT_PREFETCH_COUNT to be 1.  If the DEFAULT_PREFECTH_COUNT were
set to 0 the behavior would be round robin messaging as described above.  

<div class="diagram">
  <img src="/img/tutorials/prefetch-count.png" height="110" />
  <div class="diagram_source">
    digraph {
      bgcolor=transparent;
      truecolor=true;
      rankdir=LR;
      node [style="filled"];
      //
      P1 [label="P", fillcolor="#00ffff"];
      subgraph cluster_Q1 {
        label="queue_name=hello";
    color=transparent;
        Q1 [label="{||||}", fillcolor="red", shape="record"];
      };
      C1 [label=&lt;C&lt;font point-size="7"&gt;1&lt;/font&gt;&gt;, fillcolor="#33ccff"];
      C2 [label=&lt;C&lt;font point-size="7"&gt;2&lt;/font&gt;&gt;, fillcolor="#33ccff"];
      //
      P1 -&gt; Q1;
      Q1 -&gt; C1 [label="prefetch=1"] ;
      Q1 -&gt; C2 [label="prefetch=1"] ;
    }
  </div>
</div>

However, with the prefetchCount set to 1 by default,
this tells RabbitMQ not to give more than one message to a worker
at a time. Or, in other words, don't dispatcha new message to a 
worker until it has processed and acknowledged the previous one.
Instead, it will dispatch it to the next worker that is not still busy.

> #### Note about queue size
>
> If all the workers are busy, your queue can fill up. You will want to keep an
> eye on that, and maybe add more workers, or have some other strategy.


By using spring-amqp you get reasonable values configured for
message acknowledgments and fair dispatching. The default durability 
for queues and persistence for messages provided by spring-amqp 
allow let the messages to survive even if RabbitMQ is restarted.

For more information on `Channel` methods and `MessageProperties`,
you can browse the [javadocs online](http://docs.spring.io/spring-amqp/docs/current/api/index.html?org/springframework/amqp/package-summary.html)
For understanding the underlying foundation for spring-amqp you can find the
[rabbitmq-java-ciient](http://www.rabbitmq.com/releases/rabbitmq-java-client/current-javadoc/).


Now we can move on to [tutorial 3](tutorial-three-spring-amqp.html) and learn how
to deliver the same message to many consumers.