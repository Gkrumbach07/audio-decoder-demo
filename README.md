# Stateful Sessions for Intelligent Apps
### In this demo
We will look into solving an architectural problem in an end to end machine learning application.
First we will walk through the sample problem and outline the solution. Then we will
get the sample application running for you to test.

## Introduction
### Statefulness
In general, when an application is stateful, it utilizes pervious interactions.
For example this could be a shopping cart on an e-commerce website. In a stateful API,
each request builds of the last request where a single request can not be
interpreted alone. These types of APIs are not as common due to them breaking
a fundamental REST constraint: being stateless [[1](https://www.redhat.com/en/topics/api/what-is-a-rest-api)]. Therefore many of the benefits of REST APIs are no longer available such as easy scaling.

In some use cases, live analysis is desired which creates numerous issues for your
architecture such as scheduling tasks and scaling up your model services. In this
demo we will look into a use case where multiple models need to be strung together
for realtime analysis.

## Problem
Imagine you are call center supervisor and need to manage 100+ phone lines. You
are responsible for making sure the quality of the calls are positive. Right now
you are randomly joining calls and monitoring them individually but you would like
to automate this process.

# Solution
We can use two model services to accomplish this:
- An audio decoding model
- A sentiment analysis model

We can also create a simple web app to display our model's output in a readable
table.

**Note**

The audio decoding model is deployed using a stateful API. Because it is decoding audio live,
it only gets chunks of audio every 1 second apposed to getting all the audio in one big file.
Therefore, we must save the state of the API, otherwise it loses all reference of
the previously decoded speech, resulting in poor model performance.

## Issues
1. How do we string together all of our components?
2. How do we scale our audio decoder model?

![arch-1](https://user-images.githubusercontent.com/12587674/115573580-7d833400-a286-11eb-99d8-22429b2ebdac.png)

## Stringing it together
What we want is something that will send data from our audio decoder model, to our
sentiment analysis model, and then finally to our web app for visualization. We also
want to scale our models, so that they cumulatively can handle larger
loads (more phone lines).

We will use Kafka to accomplish this [[2](https://kafka.apache.org/intro)].
Kafka will stream the data from the outputs of our models to the inputs of the
other models. We will also configure Kafka to `auto-commit` messages. This means
that when we scale our model services to multiple instances, Kafka will distribute the processing load
across each instance. Kafka also prevents a single instance from reprocessing already processed data.

![arch-2](https://user-images.githubusercontent.com/12587674/115574054-e79bd900-a286-11eb-8162-83087f331ca5.png)


## References
[1] https://www.redhat.com/en/topics/api/what-is-a-rest-api
[2] https://kafka.apache.org/intro
