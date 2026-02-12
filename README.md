# CST8917 Assignment 1: Serverless Computing Critical Analysis

**Name:** Kelvin Ngabo 
**Student Number:** 041196196 
**Course Code:** CST8917  
**Assignment Title:** Serverless Computing Critical Analysis  
**Date:** February 12, 2026

---

## Part 1: Paper Summary

### Main Argument

Hellerstein et al. (2019) argue that while first-generation serverless computing specifically Functions-as-a-Service (FaaS) platforms like AWS Lambda represents a meaningful step forward for cloud programming, it simultaneously regresses on two critical dimensions: efficient data processing and distributed computing. The title's phrase "one step forward, two steps back" captures this tension precisely. The forward step is *autoscaling*: the ability for workloads to automatically drive the allocation and deallocation of resources without human intervention, billed on a pay-as-you-go basis. This is genuinely novel and valuable. The two backward steps are the platform's fundamental incompatibilities with (1) data-intensive applications, owing to what the authors call a "data-shipping architecture," and (2) distributed systems, because FaaS functions cannot directly address or communicate with one another. Given that modern computing innovation is overwhelmingly data-driven and distributed, the authors view these omissions as not merely inconvenient but as structurally disqualifying for FaaS as a general-purpose cloud programming platform.

### Key Limitations Identified

**Execution Time Constraints.** AWS Lambda (the paper's primary example) imposed a maximum function lifetime of 15 minutes at the time of writing. Because there is no guarantee that subsequent invocations will run on the same VM, functions must be written as stateless any internal state accumulated during one invocation is inaccessible to the next. The authors illustrate this problem concretely: training a neural network on Lambda required 31 sequential invocations each maxing out the 15-minute limit, totalling 465 minutes which is 21 times slower than the equivalent EC2 job.

**I/O Bottlenecks and Lack of Direct Addressability.** A single Lambda function achieved roughly 538 Mbps of network bandwidth on average already an order of magnitude slower than a modern SSD. More troublingly, as compute scales up and multiple functions are co-located on the same host, that bandwidth is shared. With 20 concurrent Lambda functions, per-function bandwidth dropped to approximately 28.7 Mbps. Worse, Lambda functions are not directly network-addressable during execution. Two functions wishing to communicate must do so through an intermediary storage service such as Amazon S3 or DynamoDB, and there is no "sticky" routing that would direct a client back to the specific function instance that handled its previous request.

**The Data-Shipping Anti-Pattern.** Because serverless functions execute in isolated VMs separate from data, and because they are short-lived and non-addressable, FaaS routinely "ships data to code" rather than the more efficient inverse. The authors measure the consequences directly: a single 100 MB batch read from S3 took 2.49 seconds within Lambda, dwarfing the 0.59 seconds spent on actual computation. The same workload on EC2 fetched data in 0.04 seconds. This anti-pattern inflates both latency and cost; the Lambda training run was 7.3 times more expensive than EC2.

**Limited Hardware Access.** FaaS offerings expose only a CPU hyperthread and a capped amount of RAM (3 GB on Lambda at the time). There is no mechanism to access GPUs, FPGAs, or other accelerators. The authors note, citing Patterson and Hennessy's Turing Lecture, that hardware specialization was accelerating making this omission increasingly consequential for workloads like deep learning that rely almost entirely on GPU throughput for performance.

**Challenges for Distributed Computing and Stateful Workloads.** Because all inter-function communication must pass through shared storage, classical distributed computing protocols leader election, consensus & membership management become prohibitively expensive. The authors implemented Garcia-Molina's bully leader election protocol over DynamoDB and found that each round took 16.7 seconds. Supporting a 1,000-node cluster in this fashion would cost at minimum $450 per hour. They also demonstrate that storage-mediated communication introduces latencies one to three orders of magnitude higher than direct point-to-point networking (290 µs via ZeroMQ versus 108 ms via S3 for a 1 KB message round-trip).

### Proposed Future Directions

The authors identify several characteristics a mature cloud programming platform must have. First, **fluid code and data placement**: infrastructure should be capable of co-locating code and data on the same side of a network boundary, dynamically, using high-level declarative languages as optimization guides. Second, **heterogeneous hardware support**: developers should be able to target specialized hardware (GPUs, FPGAs) either through high-level DSL compilation or explicit specification, with the platform making dynamic physical allocation decisions. Third, **long-running, addressable virtual agents**: the platform needs persistent, named software agents analogous to actors or services that remain reachable with network-like latency while being dynamically remapped across physical infrastructure. Fourth, **disorderly programming models** that embrace the asynchronous, loosely-consistent nature of distributed systems rather than imposing a sequential metaphor. Fifth, a **common intermediate representation** that serves as a compilation target across multiple languages and DSLs, avoiding the need to re-implement optimizations for every language stack.

---

## Part 2: Azure Durable Functions Deep Dive

### Orchestration Model

Azure Durable Functions introduces three primary function types that together address the composability problem of basic FaaS. The **client function** is any standard Azure Function (HTTP trigger, queue trigger, timer trigger, etc.) that starts and manages orchestrations via the Durable client binding. The **orchestrator function** is the workflow engine: it defines the sequence of tasks, uses deterministic code constructs (loops, conditionals), and invokes activity functions or sub-orchestrations using yield/await calls that act as checkpoints. The **activity function** is the basic unit of work which is stateless, unrestricted in what it can do, and safe to retry. This three-tier model differs fundamentally from basic FaaS because the orchestrator maintains durable workflow state across multiple activity invocations, whereas basic FaaS treats every function invocation as a completely isolated, ephemeral event with no continuity between calls. The paper criticizes FaaS for forcing developers to manually stitch together functions via queues and object stores with no higher-level coordination primitive; the orchestrator function is precisely that missing primitive.

### State Management

Durable Functions manages state using an **event sourcing** pattern built on Azure Storage (Table Storage and Queue Storage by default). Every time an orchestrator calls an activity and yields, the Durable Task Framework saves a checkpoint, the execution history, to durable storage before unloading the orchestrator from memory. When the activity completes, the orchestrator wakes up and **replays** the entire function from the start, but instead of re-executing already-completed steps, it reads their results from the history table and fast-forwards past them. This means the orchestrator is effectively stateless in memory between checkpoints, while its logical state is durably preserved in storage. This directly addresses the paper's criticism that FaaS functions must be written as if state will never be recoverable across invocations: with Durable Functions, state *is* reliably recoverable even across VM crashes, host recycling, or arbitrary pauses because it is always externalized and checkpointed. 

### Execution Timeouts

The paper identifies 15-minute execution limits as a critical FaaS shortcoming. Durable Functions address this by separating the concerns of *orchestration* and *execution*. An orchestrator function is not actually running continuously, it sleeps between activity call whilst consuming no compute power. It is only active during brief replay cycles, meaning the normal Azure Functions functionTimeout limit does not constrain how long a workflow takes end-to-end. An orchestration can span hours, days, or longer using durable timers. However, the timeout limitation is not fully eliminated: **activity functions**, which do the real work, are still subject to the standard function timeout. On the Consumption plan this is capped at 10 minutes; on the Premium plan, while configurable, activity functions can still be terminated after extended runtimes. For workloads that need more than 10 minutes of continuous compute in a single step, developers must manually split the work across multiple chained activity calls, which reintroduces complexity. 

### Communication Between Functions

In basic FaaS, the paper's Table 1 shows that the cheapest available inter-function communication, a write+read via S3, costs around 108 ms for 1 KB, versus 290 µs for direct ZeroMQ messaging. Durable Functions does not bypass this constraint entirely: orchestrators and activity functions still communicate through the underlying Azure Storage queues and tables managed by the Durable Task Framework. However, this communication is abstracted away from the developer and structured as first-class workflow transitions. Rather than writing ad hoc polling logic against a raw storage service, the developer simply awaits an activity call, and the framework handles the queuing, scheduling, and result delivery transparently. The storage-mediated communication overhead remains, but the programming model eliminates the accidental complexity of managing it by hand. For fine-grained, low-latency messaging between thousands of cooperating agents, the same fundamental bottleneck identified in the paper persists.

### Parallel Execution (Fan-Out/Fan-In)

The fan-out/fan-in pattern is one of Durable Functions' signature capabilities. An orchestrator can launch many activity functions concurrently; each dispatched without waiting for the previous to complete and then aggregate their results once all have finished. In code, this is expressed by creating a list of activity tasks (using the NoWait / non-awaited invocation pattern) and then calling WhenAll (or the equivalent in each language). The orchestrator checkpoints at the fan-in barrier, so if the host is recycled mid-execution, only incomplete activities need to be retried. This directly responds to the paper's concern that FaaS offered no real way for many cores to work together efficiently: fan-out/fan-in provides a structured, autoscaling mechanism to distribute parallel work and consolidate results without manual queue management. The pattern supports very wide fan-outs (theoretically unlimited, though practical limits around memory and storage throughput apply)

---

## Part 3: Critical Evaluation

### Limitations That Remain Unresolved

**1. Lack of specialized hardware access.** The paper identifies the absence of GPU, FPGA, and other accelerator support as a significant blocker for data-intensive and machine learning workloads. Azure Durable Functions does not resolve this limitation at all. Like all Azure Functions variants, Durable Functions runs exclusively on standard CPU-based virtual machines. An orchestrator can invoke an activity function, but that activity function still runs on a general-purpose compute instance with no mechanism to target GPU-equipped hardware. Developers who need GPU-accelerated computation within an autoscaling serverless workflow must route activity functions to external services (such as Azure Machine Learning or Azure Batch) that do support GPUs effectively the same "orchestration of proprietary autoscaling services" pattern that the paper already acknowledged as a workaround rather than a solution. This means that for deep learning training, large-scale inference, or any compute task with high arithmetic intensity, Durable Functions remains unable to close the performance gap that the paper measured: Lambda was 21 times slower than EC2 for neural network training, and Durable Functions does nothing to address the underlying hardware constraint that drove that result.

**2. Storage-mediated communication bottleneck.** The paper's most fundamental criticism of FaaS is that all inter-function communication transits through slow storage, making fine-grained distributed computing protocols impractical. Durable Functions improves the *developer experience* of working with storage-mediated coordination. It hides the queuing and checkpointing behind a clean programming model but it does not change the *physical architecture*. All communication between orchestrators and activities still flows through Azure Storage queues and tables. The paper measured storage-mediated round-trip latencies of 108 ms (S3) or 11 ms (DynamoDB), versus 290 µs for direct network messaging. Azure Table Storage latencies are comparable. This means that any Durable Functions workflow requiring tight coordination between many parallel agents, consensus protocols, iterative algorithms, real-time aggregation will still face the same order-of-magnitude latency penalty that the paper documented. The abstraction layer improves usability but does not improve the physics of the communication path.

### Verdict

Azure Durable Functions represents meaningful, substantive progress toward the future the authors envisioned but it is progress of a specific, bounded kind. It squarely addresses two of the paper's criticisms: the statelessness problem and the absence of a coordination primitive above raw FaaS. The event-sourcing state model makes long-running, stateful orchestrations genuinely feasible and the orchestrator function provides exactly the kind of "long-running, addressable virtual agent" that the paper's Section 4 called for an entity with persistent logical identity, capable of managing multi-step workflows with durability guarantees. The fan-out/fan-in pattern gives developers a principled mechanism to exploit parallelism across many activity instances which partially addresses the distributed computing gap.

However, the paper's deepest criticisms about hardware specialization and the physics of storage-mediated communication remain unaddressed. Durable Functions is, at its core, an orchestration layer on top of the same FaaS infrastructure whose architectural limitations the paper diagnosed. The Durable Task Framework's genius is making that infrastructure *tolerable* for stateful workflows, not transforming the underlying infrastructure itself. The authors envisioned fluid code-and-data co-location, heterogeneous hardware, and communication latencies competitive with direct networking. Durable Functions offers none of these. A workflow that needs to train a model, run GPU inference, or implement a fine-grained consensus protocol is no better served by Durable Functions than by raw Lambda.

The honest verdict is that Azure Durable Functions solves the *programmability* dimension of the paper's critique making orchestration and state management tractable without bespoke glue code while leaving the *systems* dimension essentially untouched. For the dominant class of enterprise workloads (multi-step business logic, human approval workflows, fan-out data processing), this is a genuine and valuable advance. For the class of workloads the authors cared most about novel data systems, distributed algorithms, hardware-accelerated computation & the fundamental bottlenecks remain. The paper called for a reinvention of cloud infrastructure; Durable Functions delivers a better framework on top of the existing one.



## References

- Hellerstein, J. M., Faleiro, J., Gonzalez, J. E., Schleier-Smith, J., Sreekanti, V., Tumanov, A., & Wu, C. (2019). *Serverless Computing: One Step Forward, Two Steps Back*. CIDR 2019. https://arxiv.org/abs/1812.03651
- Microsoft Learn. (2024). *Function types in Azure Durable Functions*. https://learn.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-types-features-overview

- Microsoft Learn. (2024). *Durable Orchestrations: Azure Functions*. https://learn.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-orchestrations

- Microsoft Learn. (2024). *Durable Functions Overview Azure*. https://learn.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-overview

- Microsoft Learn. (2024). *Timers in Durable Functions Azure*. https://learn.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-timers

- Microsoft Learn. (2024). *Bindings for Durable Functions Azure*. https://learn.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-bindings

- Microsoft Learn. (2024). *Fan-out/fan-in scenarios in Durable Functions Azure*. https://learn.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-cloud-backup

- Dennyson, R. (2024, September 21). *The Ultimate Guide to Azure Durable Functions*. Medium. https://medium.com/@robertdennyson/the-ultimate-guide-to-azure-durable-functions-a-deep-dive-into-long-running-processes-best-bacc53fcc6ba

