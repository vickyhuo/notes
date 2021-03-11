## **Reliable, Scalable, and Maintainable Applications**

_Functional requirements_: what application should do

_Non-functional requirements_: security, reliability, compliance, scalability, compatibility, maintainability

### **Reliability**

Expectations:

- Application performs function user expected
- Tolerate user making mistakes
- Performance is good enough for required use case and expected load
- System prevents abuse

Systems that anticipate and cope with faults are _fault-tolerant_ or _resilient_

**A fault refers to when one component of the system deviates from the spec** , whereas failure is when the system as a whole stops providing the required service to the user.

Generally prefer tolerating faults over preventing faults

- **Hardware faults** : Until recently, redundancy of hardware components was sufficient for most applications. As data volumes increase, more applications use more machines, proportionally increasing the rate of hardware faults. Cloud platforms like AWS are designed to prioritize flexibility elasticity over single-machine reliability, and can become unavailable without warning. **There is a move towards systems that tolerate the loss of entire machines** through software fault-tolerance techniques. A system that can tolerate machine failure can be patched one node at a time, without downtime of the entire system (_rolling upgrade_)
- **Software errors** : Hardware faults are random and independent from each other. Software faults are a systematic error within the system and cause many more failures across nodes.
- **Humans errors** : Humans are unreliable. Configuration errors by operators are a leading cause of outages. To make systems more reliable:
  - Minimize opportunities for error - abstractions or interfaces that make it easy to do the &quot;right thing&quot; and discourage the &quot;wrong thing&quot;
  - Provide fully features non-production _sandbox_ environments where people can explore and experiment safely
  - Automated testing
  - Quick and easy recovery from human error: fast to roll back configuration changes, roll out new code gradually, tools to recompute data
  - Telemetry - clear and detailed monitoring on performance metrics and error rates
  - Implement good management practices and training

### **Scalability**

- The system&#39;s ability to cope with increased load. Load can be described through _load parameters_ (e.g. requests per second to server, ratio of reads to writes, hit rate on cache). We need to succinctly describe current load on system before discussing growth questions

Twitter example

2 main Twitter operations

- Post tweet - user publish new message to followers (4.6k req/sec, >12k req/sec peak)
- Home timeline - user view tweets from people they follow (300k req/sec)

Two ways of implementing these operations:

1. Posting a tweet inserts the new tweet into a global collection of tweets. When a user requests their home timeline, look up all the people they follow, find all tweets for those users, and merge them (sorted by time). Can be done with SQL JOIN.
2. Maintain cache for each user&#39;s home timeline. When a user posts a tweet, look up all the people who follow that user, and insert new tweet into each of their home timeline caches

Approach 1: system struggles to keep up with load of home timeline queries, so company switches to approach 2. Avg. rate of published tweets two orders of magnitude lower than rate of home timeline reads - better to do more work at write time than read time.

Approach 2: posting a tweet requires lots of extra work. Some users have \&gt; 30 million followers, so a single tweet may result in 30 million writes to home timelines.

Twitter now has a hybrid of both approaches. Tweets fanned out to home timelines. Users with huge followings are fetched separately and merged with the user&#39;s home timeline when it is read.

**Describing Performance**

What happens when load increases:

- How is performance affected?
- How much do you need to increase the resources?

In batch processing system like Hadoop, we care about _throughput_ (# of records we can process per second)

_Response time_ is what the client sees. _Latency_ is the duration that a request is waiting to be handled

Response time is a _distribution_ of values that you can measure. Average (arithmetic mean) response time is not a good metric for &quot;typical&quot; response time since it doesn&#39;t tell you how many users actually experienced that delay.

**Better to use percentiles.**

- _Median_ (p50)- 50% of user requests served in less than median response time, 50% take longer
- p95, p99, p999 useful for figuring out how bad your outliers are

Amazon describes response time requirements in terms of 99.9th percentile because customers with the slowest requests have the most data - and therefore the most valuable

Optimizing for the 99.99th percentile was too expensive since they are easily affected by random events outside of your control.

Percentiles are used in _service level objectives (SLOs)_ and _service level agreements (SLAs)_, contracts that define expected performance and availability of a service. **These metrics set expectations for clients and allow customers to demand a refund if SLA is not met.**

Queuing delays account for a large part of response times at high percentiles. **It is important to measure response times on the client side.** When generating load artificially, the client needs to send requests independently of response time.

Percentiles in practice: For calls in parallel, need to wait for slowest of the calls to complete. The chance of getting a slow call increases if request requires multiple back-end calls (_tail latency amplification_)

**Approaches for Coping with Load**

- Scaling up (vertical scaling): more powerful machine
- Scaling out (horizontal scaling): distributing load across multiple smaller machines
- Elastic systems: auto add computing resources upon load increase - useful if load is highly unpredictable

While distributing stateless services across multiple machines is straightforward, taking stateful data systems from a single node to distributed setup can introduce a lot of additional complexity. Until recently, it was common wisdom to keep the database on a single node until scaling cost or high availability requirements forced you to make it distributed.

### **Maintainability**

Majority of software cost is in its ongoing maintenance. 3 design principles for software systems:

- **Operability** : make it easy to keep system running
- **Simplicity** : easy for new engineers to understand by removing as much complexity as possible
- **Evolvability** : make it easy for engineers to make changes to system in the future

**Operability: making life easy for operations**

A good operations team is responsible for

- Monitoring and quickly restoring service if it goes into a bad state
- Tracking down cause of problems
- Keeping software and platforms up to date
- Keep tabs on how different systems affect each other
- Anticipate future problems
- Establishing good practices and tools for deployment
- Performing complex maintenance tasks, like platform migration
- Maintaining system security
- Defining processes that make operations predictable and environment stable
- Preserving organizational knowledge about the system

***Good operability means making routine tasks easy***.

**Simplicity: managing complexity**

When complexity makes maintenance hard, budget and schedules are often overrun. There is a greater risk of introducing bugs.

Making a system simpler means removing _accidental_ complexity (complexity that is not inherent in the problem that it solves but arises from implementation)

Remove accidental complexity through _abstraction_ that hides implementation details behind clean and simple to understand APIs and facades.

**Evolvability: making change easy**

_Agile_ working patterns provide framework for adapting to change
