---
title: RocketMQ
---

## 如何保证消息不丢失

消息丢失有四个方面，每个场景解决方案不同。

1. 发送消息丢失。
2. 主从同步丢失。
3. 持久化丢失。
4. 消费消息丢失。

## 发送消息丢失

1. 使用回调函数（kafka）

发消息时注册一个回调函数，生产者可以根据是否收到回调函数来确认消息是否发送成功。

1. 使用事务消息（rocketmq）

先发送一个hafl消息，hafl消息对消费者不可见，当本地事务执行完成后再发送完整消息。队列收到hafl消息后会调用回调函数，生产者会返回事务状态。事务有三个状态：成功，失败，未知。当状态是未知时，队列会继续发起回调。当状态是成功时，将消息推送到消费者。

1. 手动事务（rabbitmq）。

提供api让开发者手动提交事务。

## 主从同步丢失

1. rocketmq。

普通集群分为同步同步和异步同步，异步同步有丢失消息的风险。使用raft协议搭建dledger集群，使用两阶段提交。主从同步过程中的主节点由选举产生，当大多数节点都收到数据后，将状态改位已提交。

1. rabbitmq。

普通集群中消息是分散的，节点之间不会主动同步，只有用到的时候才会同步，有丢失消息的风险。镜像集群会在节点之间进行同步，保证不会丢失数据。

1. kafka。

允许消息少量丢失。

## 持久化不丢失

1. rocketmq：同步保存和异步保存。
2. rabbitmq：当队列配置为持久化队列。

## 消费过程不丢失

消费者维护一个队列指针，指向当前消息队列的消费位置，如果提交指针后本地事务失败，则此时消息丢失。解决方案是采用同步提交，写完事务后再提交队列指针。

## 保证幂等性

消费者本地事务成功后，会向队列提交偏移量，若在这个过程中发生网络延迟，偏移量提交前另一个消费者消费了消息，此时就会重复消费。解决方案就是给每个消息一个唯一id作为判断一句。rocketmq在消息多的情况下不能保证id唯一，因此需要自行添加业务id来保证幂等性，也可以增加分布式id。

## 顺序消息

队列只需要保证局部有序即可，不需要保证全局有序。

rocketmq中，一个toipc对应多个队列，因此需要额外措施来保证消息有序。由于队列本身具有顺序性，因此只需要将所有消息都发往同一个队列即可。