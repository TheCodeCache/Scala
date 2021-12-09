# Actor in Scala:

It's a conceptual model of concurrent computation.  

In the `Actor model`, an `Actor` is a fundamental unit of computation.  

The only allowed operation for an `Actor` are:  
- Create another Actor
- Send a message
- Designate how to handle the next message

![image](https://user-images.githubusercontent.com/26399543/145444869-625b27fe-286c-4f3d-be53-e20f6b3dafe8.png)

An Actor maintains its states.  and can take decision of next message based on its current state.  
they have a state, but the only way to change is by receiving a message.  

Actors are lightweight..  and they are isolated from each other and `they do not share memory`.  
it's easy to create thousands or even millions of them as `actors require fewer resources than threads`.  

![image](https://user-images.githubusercontent.com/26399543/145444711-1ea4cee2-880e-44e8-8ee3-8f9ed029d04b.png)

Every actor has its own mailbox, which is similar to a message queue.  those messages are stored in the actor's mailboxes until they are processed.  
Actors can wait for the messages to arrive in its mailbox, and they can communicate only through messages.  
messages are processed in FIFO order.  
messages are simple, immutable data structure, that can be easily send over the network.  
Actors work asynchronously, so they don't wait for the responses from another actor.  
It can handle only one message at a time conceptually.  
`Actors have addresses`, so it's possible to send a message to another actor by knowing its address.  

![image](https://user-images.githubusercontent.com/26399543/145444315-0e5dc54a-64f4-401c-b572-bd12b25c8000.png)

Actors can run locally or remotely on another machine.  

![image](https://user-images.githubusercontent.com/26399543/145445233-06a0f7ae-965b-4e2f-9bf5-01acb70bec91.png)

**Fault-Tolerence:**  

![image](https://user-images.githubusercontent.com/26399543/145445577-958fa0e4-0b4f-4395-897d-6b7c586cff73.png)


**Pros and Cons:**  
![image](https://user-images.githubusercontent.com/26399543/145445743-b10f7fe0-4f3e-4c63-8e77-33f362ed3865.png)

The best known `implementation of the Actor model` are:
1. Akka
2. Elixir

**Reference:**  
1. https://www.youtube.com/watch?v=ELwEdb_pD0k

