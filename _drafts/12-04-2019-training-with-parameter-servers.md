---
layout: post
title:  "[Distributed ML] Training with Parameter Servers"
date:   2020-12-26 01:22:24 -0800
categories: distributed_ml
---

Li et al's [1] paper on training with Parameter Server is 5 years old and is one of the first papers I read when I started peeling back the layers of distributed ML back in 2017. I find the concepts and ideas in the paper very relevant even today. Parameter Servers have a lot of different algorithm variation that we can apply given the constraints of a system. A very basic definition of distributed training using parameter servers is 

“_Distributed data parallel training of a sharded model_“

Let's break this down a little bit more:

For training, we have N independent workers over which we are sharding our given data. Each worker is consuming a batch of data that is different from every other worker. The model parameters are sharded over servers called parameter servers. At each model iteration:

1. Each worker pulls the required model parameters from the parameter server
2. Computes the loss and gradients for the given batch
3. Sends the calculated gradients to the parameter server 
4. The parameter server updates the parameters of the model with these gradients

One of the most common questions users have is when should I use a Parameter Server and when should I stick to synchronous training without parameter servers. The answer depends on your model and its training characteristics. If you have a model with large embeddings that is of the order of million of GB with a chance that it could saturate the memory capacity of a single machine, you should use a parameter server approach. Each worker will only pull the embeddings that it references in its input data and consequently only calculates gradients for those embeddings/rows of the model parameters. The authors in [1] also mention topic modeling as an area where the word frequency learned in a given document requires the topics associated with all the words in the document being processed to be known. The other reason where I think this is useful is fault tolerance. In any setting, once you scale to 10s - 100s of workers you are going to encounter preemptions and errors. There is really no way around this. If your training depends on synchronizing across all workers, there is going to be some delay involved. If your model training does not depend on the tight bounds of gradient aggregation and you can tolerate some delays in gradient aggregation among all the workers then this should be an option to explore.

![Failure Statistics](/assets/images/training_with_parameter_servers/failure_statistics_servers.png)

The negative side of parameter servers is that they don’t converge as fast as synchronous training since each worker is not seeing a global batch of updates to a model parameter and depending on how the push/pull model of parameters is implemented, a worker could be pulling in stale gradients. This means you will probably need to run model training for longer and go through more iterations. The other issue is reproducibility because you could start with 50 workers and with failures you might train with fewer than the number over the course of model training. This means that you could converge at different epochs and there isn’t really a way you could say that you are going to see `x%` accuracy at `yth` epoch. This makes debugging and reproducing research results challenging.

I am going to take the paper by Li et al [1] as a starting point and try and understand the approaches that current researchers are taking. The goal is to see if research/industry is benefiting from the Parameter Server approach or are they looking for something different? Another question I am curious about is what part of the 1-4 steps above should you modify? What works and what doesn’t? This post won't be able to answer all these questions but hopefully in future posts the answer becomes more clear. From hereon I am going to refer to Parameter Server training as PS for the sake of brevity.

The architecture used for PS training is as follows: there are two sets of servers - parameter servers and workers. Parameter servers are managed by a server node that maintains parameter to server mappings along with information about backup servers. It also tracks the state of liveness of a given parameter server. A worker group is managed by a scheduler node. A scheduler node assigns tasks (e.g dataset shard should be used for gradient computation) to the workers in a group. In the event of failure/preemption, the scheduler node reassigns tasks accordingly. At the start of a given iteration, each worker pulls the latest model parameters from the parameter server. It then computes the loss using the assigned dataset shard. The gradients are calculated in the backward pass and pushed to the designated parameter server.

 
![Parameter Server Architecture](/assets/images/training_with_parameter_servers/ps_architecture.png)


# **Key Characteristics**

The paper mentions the following as the key characteristics of a PS approach:

1. Sharded model parameters placed on a dedicated server

2. Fault tolerance

3. Elasticity


The key implementations used to achieve this are the following:

### Sharded model parameters

Key-Value pairs are ordered and assumed to be vectors with range based semantics. This means that you can specify the parameter and the range of the vector that you want to push or pull from a server. 
Consistent Hashing is a methodology employed to distribute a set of values V over a set of workers N. It is used to designate an owner for a particular task or in this case an owner for a given model parameter. The advantage of using consistent hashing is that if any of the workers fail, the number of parameters that need to be reshuffled are smaller(on average k/N where k is the number of keys) than if you used round robin to shard parameters. You could also add virtual nodes onto the hashing ring to ensure that you don’t have hotspots. This is a useful technique employed in distributed systems in general when you are trying to horizontally scale up. It solves rehashing a large number of keys and hotspots. 

### Fault tolerance

Asynchronous update of gradients by all workers enables workers to continue processing input data even if any one worker in the pool fails (due to errors or preemptions). The gradient update does involve any barrier or synchronization of workers simplifies the implementation when compared with synchronous training.

When dealing with failing parameter servers, some amount of state has to be recreated on a new parameter server. For a worker failure the worker pool manager process will schedule a task on the new worker from the task queue. However for a parameter server, there will be lost pull/push messages. The paper in [1] mentions a few steps it takes to enable the server manager node to handle this case. The server manager process tracks healthy parameter servers and key ranges maintained by each server. When a parameter server goes down, the range of keys are remapped. The parameter server that is assigned the new key range, copies the values from the backup servers (these are the two previous servers on the hashing ring). This change is then broadcast by the server manager to all other nodes in the parameter server pool that then modify their own key-value pairs accordingly. The liveness of parameter servers are tracked by the server node using a heartbeat mechanism.

The server manager node and the resource manager seem like single point of failures that will require a complete restart on failure. For processes like these, it is often that case that we run it as a higher priority task to prevent preemption. 

##### Duplicate/Stale Messages

Vector Clocks are used to ascertain the logical timestamps of events in a given system. They help us understand the causal relationship between any two events in a system. For implementing fault tolerance, they are used to determine the staleness of messages sent by workers to the parameter server or when a parameter server receives messages regarding range split for a given key-value partition. Timestamps help determine if a server has already processed a given message and can thereby discard a delayed inbound message. The authors in [1] associate a given range with 1 timestamp so that we don’t see an explosion of timestamps (N nodes * m model parameters). 

### Elasticity

Elasticity is the process by which we increase the size of the worker or parameter server pool. The process followed is identical to that mentioned for fault tolerance above. 


# **Tuning Knobs**

The knobs used to tune the various tradeoffs are:

### Bounded delay of gradient aggregation

There are three levels of gradient synchronization that you can achieve in model training:


![Types of Delay](/assets/images/training_with_parameter_servers/types_of_delay.png)


1. Synchronous: This is similar to training in a single thread where tasks are executed sequentially. The gradients from all workers are aggregated before the parameter is updated.

2. Eventual: The gradient updates are applied asynchronously and independently by each worker. This is exactly opposite of 1.

3. Bounded delay: This is between 1 and 2 where we impose some restriction over how far workers can diverge and end up with stale gradients. In this case, a worker does not start processing input data until all the previous iterations before a time interval {tau} has been completed. The higher the value of tau, the longer it takes the model to converge.

### Reducing communication cost

In order to limit the number of gradients that are sent to the parameter servers, workers can do the following:

* Compress gradients and only send non zero gradients for updating model parameters.

* User Defined Filters can be applied to certain gradients before being sent across the wire such that only gradients that are above a certain threshold or that are significantly different than the last gradient values sent are actually used to update the model on the PS.

### Key-Value Replication

Each node stores a range of keys R and replicates this range on the previous two servers present in the consistent hashing ring. This means that any gradient that is received and results in a model update needs to propagate to the other backup servers on which the range is replicated. The two ways of doing this are:

Synchronous replication where the model parameters are updated on the master and replicas before responding to the worker. 

Asynchronous replication where the model parameters are updated on the master before responding to the worker. The replication process is started but not completed before responding to the user. This means that if the master fails, there is a chance that the replicas don’t have the latest gradients leading to model inconsistency and increasing the number of iterations required to reach convergence.

# **Results** 

The real world example of Sparse Logistic Regression provided by the authors have examples in the scale of over 150 billion examples and a feature set of 65 billion. The authors are able to demonstrate the scalability, network savings and time to convergence of PS training. The results specifically demonstrate the following:


![Idle Worker Time](/assets/images/training_with_parameter_servers/idle_worker_time.png)

* Workers are processing for a greater % of the time when compared to synchronous gradient aggregation architecture. This is because workers can begin processing the next shard of data as soon as the current shard is done without waiting for synchronization of gradients which imposes a barrier function.


* PS training requires more CPU processing because more iterations are required to reach the same objective function.


![Network Savings](/assets/images/training_with_parameter_servers/network_savings.png)


* Reducing communication cost halves the total time taken.

![KKT filter](/assets/images/training_with_parameter_servers/KKT_filter.png)


* Caching parameter keys and filtering gradient values lower than a threshold (using KKT filter) can reduce network traffic significantly. The number of gradients filtered increases with time.


![Bounded Delay](/assets/images/training_with_parameter_servers/bounded_delay.png)


* The bounded delay {tau} knob allows the authors to find a good balance between idle workers and convergence delay. Increasing bounded delay enables low idle worker times but increases time to convergence.


Two other real world examples are also implemented using the PS technique. Some things that would have been interesting to see are the memory savings on a worker when using PS and straggler mitigation techniques.


# References

[1] Li, M. (2015). Scaling Distributed Machine Learning with the Parameter Server. https://doi.org/10.1145/2640087.2644155


