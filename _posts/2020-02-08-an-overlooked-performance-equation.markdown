---
layout: post
title:  "Part 1 - Understanding Wait Time in Pull-Based Message Consumers"
image_folder: overlooked
categories: perf
---

### Purpose

This is going to be a multi-series post on developing the right mental models when optmizing for throughput when using Cloud Pub/Sub. My goal is to give you the right tools such that you control throughput tailored to your systems' requirements and needs. I'm using Cloud Pub/Sub here in order to frame the problem but **hopefully, this post is able to shed deeper fundamental principles around understanding the behavior of your messaging systems**. Before diving into all of the knobs and levers that Cloud Pub/Sub offers, I am going to talk first about the problem at a higher abstraction with the hope that you can choose the right tools and configurations which intentionally fit your use case. Push messaging at Spotify, and messaging in general, relies quite heavily on these ideas.

Before we begin this post assumes some super elementary knowledge of Cloud Pub/Sub which is a persistent message bus. Messages are published to topics and messages are consumed from subscriptions. De-coupling these two concepts allows publishers to be de-coupled from consumers. For example, I can publish "hello world" to topic X. Two subscriptions can be furnished from topic X called subscription Z and Y, respectively. Consumers who are subscribed to subscription Z can receive message "hello world" and consumers who are subscribed to subscription Y can receive another copy of "hello world." The Google Cloud documentation has awesome information, especially the *Core Concepts* section you can find [here](https://cloud.google.com/pubsub/docs/overview). I'm not going to do any deeper than this but will assume that you can refer to documentation as necessary.

Fow now, I'm not going to go much deeper in Cloud Pub/Sub until the case study, because I think the most general and useful mental models can be derived without talking specifically about the technology. For now, we'll keep it simple and refer back to these concepts later in the [Case Study]()! With that in mind, let's get started.

### Know You "Wait Time"

```
Response Time = Wait Time + Service Time
```

Quite generally, *Response Time* is the total time it takes from request to completion. Imagine you're ordering a cocktail at the bar. The bartender receives your request and proceeds to take a few more orders. After about a minute or so, he begins to mix your drink and he finishes it in about another minute. So the *Response Time* you perceive was a total of 2 minutes, 1 minute was spent making the cocktail (Service Time) and another minute was spent doing something else (Wait Time). Notice how I mentioned that this was what **you preceieved.** We'll come back to this because surprisngly, this is an important level of detail.

When running Cloud Pub/Sub at "high throughput," a common practice is to pull a message, process it, and acknowledge that you're done with it. In many client implementations, the logic to "pull" the messasge is not even something that you control. We'll dive deeper into the exact mechanics of this later but you imagine that in your application the following happens:

> "Messages can be received in your application using a long running message listener, and acknowledged one message at a time" [Source](https://cloud.google.com/pubsub/docs/pull)

So, what's your **wait time**, from the perspective of your application? It's **zero**, or at least very very close to zero. Yeah, a message has to wait in a persistent queue like Cloud Pub/Sub, but from the perspective of your application, assuming it receives work whenever it's ready for more, has a "zero wait time." This will be super handy later on but before we go there, let's dive a bit deeper and more concretly into this mental model. So strap up for a small bit of code.

### So Your Wait Time is Zero, Huh?

Concretely, imagine you implemented "long running message listener" and messages are passed to your application by implementing `receiveMessage` method like below:

{% highlight java %}
@Override
void receiveMessage(PubsubMessage message, AckReplyConsumer consumer) {
    meter.mark("arrived");

    handle(message).whenComplete((result, throwable) -> {
        consumer.ack();
        meter.mark("finished");
    });
}

{% endhighlight %}
[Source](https://googleapis.dev/java/google-cloud-clients/latest/com/google/cloud/pubsub/v1/MessageReceiver.html)


When you receive a message, you meter the rate of it's arrival. When you're finished, you acknowledge the message to indicate that you're finished processing and you meter when you finish. By acknowledging that you are finished, the client library then is able to hand your application another message. And so the process continues. There's a few details I skimmed by let's assume that there's a fixed number of concurrent messages you can process at a given time and the only way to receive more is by acknowledging when you're finished. For now, that is all the detail we need since our focus is on understanding the mechanics of controlling throughput.

If you were to look at the metered data in Grafana or what not, you might see something like this:

<img src="{{site.baseurl}}/img/{{page.image_folder}}/arrival-vs-finished.png">

Notice that the rate of seeing an "arrived" message is nearly identical to the rate of seeing a "finished" message. This intuitively makes sense - when we see a new message, we're ready to immediately process it. And when we're done, we ask for more. Nice. So...**what does this have to do with wait time?**

### Defining Your Transaction

I'm going to start using the term "transaction" to refer to the following: 

> In its broadest sense, a "transaction" is a group of actions that should be performed as if they were a single "bulk" action a group of actions that should be performed as if they were a single "bulk" action." [Source](https://softwareengineering.stackexchange.com/questions/289139/what-is-a-transaction)

Depending on the system that you are working on, and at what perspective, your definition of a transaction may vary.

The common example is the classic web server, with a traditional request/response model. A transaction here can be defined from the point of the request and the response. I really like the example of the single threaded web server Evan Jones talks about [here](https://www.evanjones.ca/prevent-server-overload.html). Imagine you have a single threaded web server that can process a single request in 10 ms. If a request arrives while the web server is processing, it has no option but to wait, probably in some sort of in-memory queue. Here, the wait time is non-trivial and can't be ignored. This is often why a transaction is better defined by the response time, since it acknowledges that there may be a portion of waiting that occurs.

In our pull-based message consuming system, the transaction is the time from which a message arrives to when it is "finished." Since the rate of arrival is equal to the rate of completion, we know that messages are processed immediately and hence there is a zero wait time. This means that the response time is equal to the service time.

### So What??

Finally, we'll end this part of the series with an equation that ties all of this together. Since we can assume our wait time to be zero, we can actually begin to extrapolate throughput of the system. Specifically, we can lean on the following equation:

```
maximum-throughput <= 1 / average-service-time
```
[Source](https://www.speforums.com/resources/throughput-transactions-per-second-tps.60/)

In our pull-based messaging consuming system, it's quite simple to extrapolate average service time since its equal to our response time. Since we know our response time is the duration it takes to complete a transaction, we just need to find how long on average it takes for a single message to arrive versus for it to finish. In the next post, we'll dive deep into a Cloud Pub/Sub case study to full understand how to tune throughput. See ya then.