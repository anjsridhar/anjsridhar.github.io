---
layout: post
title:  "[Distributed ML] Distributed Training"
date:   2020-12-01 01:22:24 -0800
categories: distributed_ml
---

Training your model at scale involves different technologies at different levels of the stack. From the lowest level of the stack we first have the accelerator and framework. On top of this sits libraries that have easy to use APIs that don’t necessarily involve dealing with the complexity that comes with the machine learning frameworks out there. The level above that handles tasks that require a global view of training your model. This kind of orchestration usually involves scheduling of training tasks and cluster management. These levels don’t always have a clear boundary and functionality often criss crosses across the stack

There are primarily two popular ML frameworks that can be used to train ML models - TensorFlow and Pytorch. By ML frameworks I mean something that converts python code into op level/kernel code that can be executed on accelerators. These were not the first ML frameworks in existence and there is a lot of history here if someone wants to explore.

To survey all of the above is confusing at first. Where do we start? I’ve abstracted away the basic concepts and tried my best to provide the first step towards what you need to understand to make different choices in your ML training stack to achieve scale and performance. 

![Synchronous AllReduce](/assets/images/intro_distributed_training/synchronous-allreduce.jpg)

At first googling "Distributed Training" will often lead you to come across two popular architectures - Data Parallelism and Parameter Servers. 

At a high level data parallelism involves feeding N devices N slices of a global batch. The ML model and computation is replicated on each of these N devices. Let us assume that each worker controls one device. The model sees a lot of more data i.e you are able to use a larger global batch in a given time frame which allows the model to train faster. The steps to carry our one iteration of training are:

1. At time {t1}, the Dataset D is sharded over the three workers {W1}, {W2} and {W3}.
2. At time {t2}, the model computes the forward and backward pass individually on each of the devices using its designated data shard {D1}, {D2} or {D3}.
3. At time {t3}, the gradients {g1}, {g2} and {g3} are aggregated using the allreduce algorithm which results in all devices having the same resultant gradient {g}.
4. At time {t4}, all workers update the model parameters with the same gradient thereby remaining in sync ready for the next iteration.

The other school of thought is Parameter Server training. The data is still sliced by the number of workers N however the gradient aggregation is traditionally carried out asynchronously. The model parameters reside on the parameter servers. The workers compute the gradients and send it to the parameter server at different timestamps.This means that the model paramters requested by the model at the start of the next iteration may be updated by 0 to N gradient values depending on how many gradient updates were received and applied by the parameter server. The steps to carry out one iteration of training are:

1. The Dataset D is sharded over the three workers {W1}, {W2} and {W3}.
2. The model computes the forward and backward pass individually on each of the devices using its designated data shard {D1}, {D2} or {D3}.
3. The gradients {g1}, {g2} and {g3} are sent to the parameter server using which the model parameters are updated with each of the gradient values.
4. The workers request the updated model parameters at the start of the next iteration.

More advanced distributed training architectures such as model parallelism, pipeline parallelism and hybrid parallelism have also shown great promise. Note that the terminology of data parallelism, parameter server, model parallelism are not exclusive. For example, Parameter Server does involve sharding input data across workers (Data Parallelism) and may also involve sharding model parameters, especially large embeddings across designated parameter servers. It is more useful to think about these concepts individually (input sharding, gradient aggregation, parameter placement etc.) as opposed to in its entirety (Data Parallelism, Model Parallelism etc.) 
With this in mind, let us understand the different cut points in our ML training workflow at which we need to think about distributed specific logic.

 For ease of explaination I am going to consider a synchronous distributed training loop without parameter servers. The training code that you write for a single machine will now need to be modified to be able to work in a distributed setting. The main cutpoints are:

1. Input
2. Model
3. Training Loop 
4. Gradient aggregation
5. Losses
6. Metrics
7. Checkpointing


![Distributed Training Cutpoints](/assets/images/intro_distributed_training/distributed-training-grid.jpg)

**Input** on a single machine for training generally involves taking a list of files or raw data, adding  a bunch of transformations and finally returns data in a certain format and of a given batch. Traditionally you might represent input by describing a dataset object spec and an iterator representing how you intend to traverse the data. With distributed input there are more considerations. You need to think about how you are going to shard, shuffle and distribute this data among all the devices. The other tricky thing is to be aware of partial batches and understand if the framework deals with it. Distributed input is a complex topic by itself and requires a dedicated post.

When you instantiate a **model** on a given device, the model parameters reside on the said device. If you want to distribute your model over multiple devices and across workers, you might have to either replicate your parameters or place them on designated servers called parameter servers. Your **optimizers** may also have certain state variables that need to be updated per iteration. Some of these state variables are dependent on model parameters and may need to be colocated with them.  

The **training loop** involves defining a function that takes in some input, applies the model on it and has a predicted output. Within this training step, there are a few interesting things that happen with respect to distributed training - aggregation of data among different devices and updating model variables. For any training loop, the steps are:

1. Calculate the predicted outputs for a given input
2. Calculate the loss of the predicted vs actual outputs for a given batch of data
3. Calculate the gradients of the model variables wrt to loss
4. Aggregate gradients
5. Update model variables


Steps 1, 2, 3 and 5 are the same for single device and multi device training. Step 4. which involves **gradient aggregation** is where we need to employ something we call collective communication libraries (e.g MPI, NCCL) to communicate among devices. Employing HPC techniques to handle the increased demand for distributed training has become extremely popular. One of the first successful attempts that I read about was Baidu’s allreduce [5] implementation back in 2017. For gradient aggregation across multiple devices, we generally use the allreduce algorithm such that each of the devices participating in the communication protocol end up with the same gradient value at the end of the protocol. 

I do need to mention that Step 2 where we calculate **loss**, loss might need to be scaled depending on how we are aggregating gradients. This has been a source of great confusion for many users. If you sum gradients during aggregation you need to scale the loss by the total number of examples in the entire batch and not just the mini batch that you are computing on one single device.

**Metrics** may also involve aggregating data across all devices and affects the performance of model training. You can either aggregate the metrics such as accuracy at each step which will definitely cause a drop in performance or you can track the accuracy metric per device and aggregate it every `n` steps or at the end of the epoch. 

Certain layers such as BatchNormalization involve calculating moving mean and moving average statistics of input data during training. These **statistics** are then used to normalize input during inference. The fact that you don’t need the aggregated statistics immediately means that you can employ cross device aggregation when you checkpoint your model. If you choose to aggregate statistics of all input samples seen per iteration, you should use SyncBatchNormalization layers. These are useful if you batch size is small when using input with high memory consumption.

You’ll notice that I have been a little vague about the exact APIs to use for each of the steps above and the reason is that I want this to be applicable to any kind of distributed training that you need to do. At each of these cutpoints that require some user/framework involvement, I believe the user should understand exactly what is happening so that they can write more performant training loops.

If you are interested to learn more about the different technologies out there I suggest a few survey papers [2][3][4] which provide an excellent overview of the different distributed training methodologies out there.



#### References

[1] https://towardsdatascience.com/modern-parallel-and-distributed-python-a-quick-tutorial-on-ray-99f8d70369b8

[2] Chahal, K., Grover, M. S., & Dey, K. (n.d.). A Hitchhiker’s Guide On Distributed Training of Deep Neural Networks.

[3] Mayer, R., & Jacobsen, H.-A. (2019). Scalable Deep Learning on Distributed Infrastructures: Challenges, Techniques and Tools. In ACM Comput. Surv. 1, 1, Article (Vol. 1). Retrieved from https://doi.org/0000001.0000001

[4] Verbraeken, J., Wolting, M., Katzy, J., Klop-Penburg, J., & Rellermeyer, J. S. (n.d.). 3 A Survey on Distributed Machine Learning.

[5] https://github.com/baidu-research/baidu-allreduce
