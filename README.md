# Stateful Sessions for Intelligent Apps
### In this demo
We will look into solving an architectural problem in an end to end machine learning application.
First we will walk through the sample problem and outline the solution. Then we will
get the sample application running for you to try out. Also feel free to check out the talk [[1](https://www.youtube.com/watch?v=F_n90IBrSaM)] I did
at DevConf [[2](https://devconfcz2021.sched.com/event/gmNH)] where I went into more
 detail on the setup and troubleshooting of use cases like this.

## Introduction
This end to end demo tackles the problem of dealing with state in a realtime machine learning
application.
### Statefulness
In general, when an application is stateful, it utilizes previous interactions.
For example this could be a shopping cart on an e-commerce website. In a stateful API,
each request builds off the last request where a single request can not be
interpreted alone. These types of APIs go against OpenShift/Kube's default model of
having "stateless microservices". Therefore many of the benefits of OpenShift's
scale-out compute resources are no longer available.

In some use cases, real time analysis is desired which creates numerous issues for your
architecture such as scheduling tasks and scaling up your model services. In this
demo we will look into a use case where multiple models need to be strung together
for real time analysis. We will also tackle the scaling issue for stateful APIs.

## Problem
Imagine you are a call center supervisor and need to manage 100+ phone lines. You
are responsible for making sure the quality of the calls are positive. Right now
you are randomly joining calls and monitoring them individually but you would like
to automate this process.

## Solution
We will build our app using OpenShift and Open Data Hub. We can deploy two model
services to solve our problem:
- An audio decoding model - to convert to live phone calls to text.
- A sentiment analysis model - to extract relevant information from the text.

We will also deploy a web app to display our model's output in a readable
format.

**Note**

The audio decoding model is deployed using a stateful API. Because it is decoding audio live,
it only gets chunks of audio every 1 second opposed to getting all the audio in one big file.
Therefore, we must save the state / previous audio chunk in the API, otherwise the model loses all reference of
the previously decoded speech, resulting in poor model performance.

### Issues
1. How do we string together all of our components (models, web app, phone lines)?
2. How do we scale our audio decoder model (support many phone lines)?

![arch-1](https://user-images.githubusercontent.com/12587674/115573580-7d833400-a286-11eb-99d8-22429b2ebdac.png)
Figure 1. architecture layout with missing connection between the elements of the app.

#### Stringing it together
What we want is something that will send data from our audio decoder model, to our
sentiment analysis model, and then finally to our web app for visualization. We also
want to scale our models, so that they cumulatively can handle larger
loads (more phone lines).

We will use Kafka to accomplish this [[3](https://kafka.apache.org/intro)].
Kafka will stream the data from the outputs of our models to the inputs of the
other models. We will also configure Kafka to auto commit messages. This means
that when we scale our model services to multiple instances, Kafka will distribute the processing load
across each instance while preventing reprocessing of already processed data. This is done
by keeping all of our consumers (sentiment analysis model service instances) on the same group id.
Consumers with the same group id will not reprocess already processed data.

![arch-2](https://user-images.githubusercontent.com/12587674/115574054-e79bd900-a286-11eb-8162-83087f331ca5.png)
Figure 2. architecture layout with Kafka introduced and connected

#### Scaling the audio decoder
The last issue in our architecture is how we will scale our stateful API (audio decoder).
Because the API needs to store the state of previous calls, we can not use the traditional
scaling features of a REST API. Those strategies might use a round-robin load balancing
technique which does not guarantee that clients will hit the same endpoint on every
call to the API. To fix this, we can utilize OpenShift sticky sessions [[4](https://docs.openshift.com/container-platform/4.7/networking/routes/route-configuration.html#nw-using-cookies-keep-route-statefulness_route-configuration)]. This feature tells the
ingress controller to attach a cookie to a api response, where a cookie is linked
to an endpoint. Meaning a client will always go to the same endpoint as long as it
stores the cookie and uses it in the next API request.

Now we can safely scale the API as if it was a REST API.

![arch-3](https://user-images.githubusercontent.com/12587674/116575164-037d2b80-a8d4-11eb-9e3b-1b249c4fe67d.png)
Figure 3. ingress controller using cookies to maintain statefulness.

## Conclusion
This technique for preserving state in an API is not the only solution nor is it
a complete solution. However this technique provides a quick and relatively lightweight
way of making an API scalable. Because this type of architecture follows a
pipe and filter approach to data flow, it is easy to update single components such as
the model service without altering connections between components. A data scientists can directly play around with the ML notebook while it
is functioning inside the architecture as a whole. That alone is a powerful development tool.
You will have a chance to see that in action below. 

## Project Materials
### Launch the sentiment analysis model service
In this demo there is a default sentiment analysis model service that is always
running. You will be contributing to the computations in your Jupyter notebook
in parallel with the default model service.
1. Visit https://odh.operate-first.cloud\
2. Launch JupyterHub
3. Login with moc-sso and then login your google account
4. In this Spawn screen, select `audio-decoder-demo:latest` (no need to change other settings)
5. Once your server starts, go into the directory named `audio-decoder-demo-yyyy-mm-dd-hh-mm`
5. Run the notebook named `sentiment.ipynb` and follow instructions

### See the results
You can access the web app here: http://call-center-manage-client-fde-audio-decoder-demo.apps.zero.massopen.cloud/

You can access the OpenShift namespace here: https://console-openshift-console.apps.zero.massopen.cloud/topology/ns/fde-audio-decoder-demo

In the namespace under the [monitoring dashboard](https://console-openshift-console.apps.zero.massopen.cloud/dev-monitoring/ns/fde-audio-decoder-demo/?workloadName=audio-decoder&workloadType=deployment), notice that the audio-decoder api
is properly disturbing the load still.

### How to Contribute / Provide feedback
- Github Repositories:
  - https://github.com/Gkrumbach07/audio-decoder-demo (notebook)
  - https://github.com/Gkrumbach07/docker-py-kaldi-asr (api & simulator)
  - https://github.com/Gkrumbach07/call_center_manage (web app)
- You can open up a PR on the Git Repository highlighting the feature or issue, and we will address it.
- You can also reach out to gkrumbac@redhat.com for any questions.

## References
[1] https://www.youtube.com/watch?v=F_n90IBrSaM

[2] https://devconfcz2021.sched.com/event/gmNH

[3] https://kafka.apache.org/intro

[4] https://docs.openshift.com/container-platform/4.7/networking/routes/route-configuration.html#nw-using-cookies-keep-route-statefulness_route-configuration
