# Implement Snooping Cache Coherent

Challenge: both requests from processor and bus require tag lookup

**This is an example of contention**

<div>           <!--块级封装-->    <center>    <!--将图片和文字居中-->    <img src="media/tag_contention.png"         alt="无法显示图片"         style="zoom:75%"/>   </center></div>

### Possible solution

Since write is less frequent than read, we can duplicate the tags and state, when reading there will not be contention.

<div>           <!--块级封装-->    <center>    <!--将图片和文字居中-->    <img src="media/duplicate_tag.png"         alt="无法显示图片"         style="zoom:75%"/>   </center></div>



## Report snoop results

Each processor would have three additional wires.

When a processor has that cache line then it activate *Shared*

When a processor has that cache line and it's been modified then it activate *Dirty*

When a processor has not done snooping yet *Snoop-pending* keeps on

<div>           <!--块级封装-->    <center>    <!--将图片和文字居中-->    <img src="media/additional_wires.png"         alt="无法显示图片"         style="zoom:75%"/>   </center></div>

## Problem with write-back cache

When using write-back cache, one popular optimization when evicting dirty line and then load new line to replace its place can be put the dirty line into a buffer and read the requesting line right away. **Make the write async.**

This requires extra complixity when implementing snooping, we have to snoop the lines in write-back buffer too.

<div>           <!--块级封装-->    <center>    <!--将图片和文字居中-->    <img src="media/write_back_buffer.png"         alt="无法显示图片"         style="zoom:75%"/>   </center></div>



## Race Condition

### Deadlock

<div>           <!--块级封装-->    <center>    <!--将图片和文字居中-->    <img src="media/snoop_bus_deadlock.png"         alt="无法显示图片"         style="zoom:75%"/>   </center></div>

Cache must be able to respond to snoop requests from other processors while it's waiting to acquire bus. It has to answer or acknowledge snoop requests, to help others finish using bus in order to acquire the bus itself. Or this would cause deadlock.



### Circular dependency

<div>           <!--块级封装-->    <center>    <!--将图片和文字居中-->    <img src="media/queue_deadlock.png"         alt="无法显示图片"         style="zoom:75%"/>   </center></div>

* Outgoing read request (initiated by processor)
* Incoming read request (due to another cache)
* Both requests generate reponses that require space in the other queue

##### Possible Solution:

* Make the size of the queue big enough to fit the max number of requests needed
* Separate requests/reponse queue
  * System classify all transactions as requests and responses
  * Resources can be completed without generating further transactions
  * While stalled attempting to send a request, cache must be able to service reponses
  * Responses will make progress(the generate no new work so there's no circular dependence), enventuallty feeding up resources for requests

<div>           <!--块级封装-->    <center>    <!--将图片和文字居中-->    <img src="media/separate_queues.png"         alt="无法显示图片"         style="zoom:75%"/>   </center></div>



### Livelock

<div>           <!--块级封装-->    <center>    <!--将图片和文字居中-->    <img src="media/snoop_livelock.png"         alt="无法显示图片"         style="zoom:75%"/>   </center></div>

To avoid livelock, a write that obtains exclusive owenership must be allowed to complete before exclusive owenership is relinquished.



## Memory Consistency



<div>           <!--块级封装-->    <center>    <!--将图片和文字居中-->    <img src="media/memory_switch.png"         alt="无法显示图片"         style="zoom:75%"/>   </center></div>