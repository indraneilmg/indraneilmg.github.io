---
layout: post
title: "SoC Performance Architecture"
date: 2024-11-11
categories: blog
---

<p style="font-size: 20px;">I am going to do a simple thought exercise. Let’s build a very simple <strong>Accelerator</strong>. The goal is obviously not to build the most intricate AI accelerator like what <strong>Google TPUs</strong>, <strong>AWS’ Terrarium</strong>, <strong>Meta’s MTIA</strong>, <strong>Azure Maia</strong> etc., are (and it’s not humanly possible for one person to even do it!) but one that’s simple enough to stress the fundamental concepts and building blocks of an <em>‘AI accelerator’</em>.</p>


<p style="font-size: 20px;">The idea is to incorporate concepts of <strong>System Architecture</strong>, <strong>Core micro-architecture</strong>, <strong>Design</strong>, <strong>Performance Modeling and Analysis</strong> as well as we dig deep into understanding what it takes to build an SoC.</p>

<p style="font-size: 20px;">Before I begin, a <strong>Big Disclaimer</strong>. If this is too simplistic for your liking, I am sorry! If you find this useful, thank you! The structure could be a bit unorganized too. I will be referring to multiple papers, blogs, lectures, youtube videos, etc <strong>AND</strong> - I will use the help of AI, ChatGPT and other tools wherever I can. The idea is to learn as I go through this process and if it helps others, GREAT! With that, let’s begin!</p>

<p style="font-size: 20px;">Before we look into the SoC architecture and deep-dive, let’s talk about the lifecycle of a typical <strong>SoC Architecture program</strong>, from concept & pathfinding all the way to Silicon release, there are probably a million or more steps/things that need to happen in between. But, at a high level, for my mental model i’d like to break it into 4 stages.</p>

<strong style="font-size: 22px; text-align: center; display: block;">Pathfinding ↔ Arch Definition ↔ Pre-Silicon & Execution ↔ Post-Silicon</strong>

<p style="font-size: 20px;"><strong>Pathfinding</strong> is where you determine what you want to build, what the product requirements are, SoC KPIs are, what are the key workloads you want to use to build the SoC, what architecture choices you want to make, how the high level SoC construction should look like. For the product you’re building, determine the  Power, Performance, Area targets, Latency/Bandwidth targets for the SoC. Critical is knowing and internalizing why you’re building something. What’s the purpose? What is the market this product will address? What is the Software Model, will it use an existing stack or is there a need to build a new Software Ecosystem.</p>

<p style="font-size: 20px;"><strong>Architecture Definition</strong> is solidifying the findings of the pathfinding stage and making certain choices for key parameters like 1/ SoC Performance KPIs, V/F curves 2/ SoC Construction Evaluation 3/ High level Performance and Power Analysis 4/ Key SoC Components and IPs 5/ Begin Performance Modeling of IPs and SoC, etc. Also, at this stage we’re getting our ‘Simulation Infrastructure’ ready i.e. test plans, micros, benchmarks, workloads etc.</p>

<p style="font-size: 20px;"><strong>Execution</strong> is working on details of the performance models, running micros, getting initial architecture validation and correlation going. Working on the RTL code for IPs and the SoC. Building and integrating the Power and Performance models for different IPs and integrating them for the SoC. E.g. Compute Model, Fabric Model, PCIE model, IO model etc. Go over SoC flows for various use-cases Work towards a final projection for a broad set of workloads. The execution stage is probably the most complicated one as it involves stitching multiple items in flight. Let’s scratch the surface of one of the topics i.e. Pre-Silicon correlation.</p>

<p style="font-size: 20px;"><strong>Pre-Silicon correlation</strong> is a critical part of any Architecture team. The goal is to make sure, for critical workloads of interest (for which you’re probably designing the system) we are able to correlate the cycle-accurate simulator performance (really IPC, BW/Latency) with the RTL model and in the future steps of the program, with Emulation. If you’re building a program from scratch, even before you start doing this correlation, a few items need to happen, namely </p>

<p style="font-size: 20px;">

<strong>* Simulation Infrastructure</strong><br>
	--> High level Abstract simulators<br>
	--> Cycle accurate simulators<br>
	--> IP RTL Models<br>
<br>
<strong>* Tools and Flows</strong><br>
	--> Tracing environment<br>
	--> Simulation Flows/Processes<br>
	--> Post Processing tools<br>
<br>
<strong>* Test Plans and Stimulus</strong><br>
	--> <strong>Directed Tests</strong> : Tests focusing on a particular feature or functionality e.g. Branch Prediction, Instruction Latency, Cache Hit/Miss, etc.<br>
	--> <strong>Micros</strong> : Tests are a bit more complicated than just writing a test for say a Branch Predictor. Here, we would look at an ‘Hotspot kernel’, typically look at more than just 1 hot function.<br>
	--> <strong>Real Workloads</strong> : These are ‘snippets’ from a Real world workload e.g. speccpu, spec int rate, etc. Typically, these workloads are traced, add weights, correlate (to make sure they are not Garbage-In Garbage-Out)<br>
<br>
<span style="color:#FFA500;">Working with simulators is mostly about balancing <strong>speed</strong> and <strong>accuracy</strong>. You (typically!) begin with an back-of-the-envelope excel based model, usually an <strong>analytical model</strong> (we'll cover that here in this blog) then followed by an <strong>cycle approximate model</strong>, followed by an <strong>cycle accurate model</strong> and then an <strong>full RTL simulation</strong> and if time and money permits, an <strong>Emulation Model</strong></span>.
</p>

<p style="font-size: 20px;"><strong>Post-Silicon</strong> begins with the first silicon coming in and Power On. At this stage, the focus is on getting the fundamentals right i.e. usually at power on, not all cores are active. Typically, we start with KPI validation, SoC flows, run micros, benchmarks compare BW/Latency curves pre-si and measure on 1st silicon and correlate.</p>

<p style="font-size: 20px;">SoC/CPU Architecture is almost always a <strong>‘feedback loop’</strong>. Each stage informs and influences not only the next stage but also, the previous one. Projects that don’t consider this or actively look for it end up suffering. It is critical for a good SoC Architecture Design Program to take into account the technical analysis and learnings from the (n+1)th step.</p>

<p style="font-size: 20px;">Unspoken themes that tie all this together are <strong>schedule</strong> and <strong>execution</strong>. All of this obviously needs to happen within the given project timelines.</p> 


<p style="font-size: 26px;"><u><strong>Building a very simple Accelerator</strong></u></p>

<p style="font-size: 20px;">Let’s start with a very simplistic (ofc! inefficient) model, and play around with it to explore the ideas touched upon above. Assume you’re building an SoC and all it can do is matrix multiplication (let’s call it a Processing Engine) and the PEs need to be fed for peak performance. Design an high level architecture, calculate speeds-n-feeds, peak performance. The high level architecture looks something like this below:</p>
<br>
<img src="/assets/arch.jpg" alt="Accelerator 101" style="object-fit: cover; object-position: top;">
<br>
<p style="font-size: 20px;">Right now, I am not going to worry about other typical components on an accelerator SoC like the Vector Cores, Control Unit DMA (Direct Memory Access), NoC (Network on Chip), Fabric, etc. In this blog I'll only scratch the surface and talk about what's the peak performance this architecture can deliver i.e. an rough <strong>roofline/analytical model</strong></p>

<p style="font-size: 20px;">Let’s make some assumptions</p>

<p style="font-size: 20px;">Let's assume that this architecture is designed to multiply a matrix of the following input C(MxN) = A(MxK) x B(KxN); M=32K N=8K K=16K.</p>
<p style="font-size: 20px;">Each PE is operating at 2 GHz.</p>
<p style="font-size: 20px;">Each PE can perform 8 FP8 operations per cycle (i.e. 1 Byte per element).</p>

<p style="font-size: 22px;"><strong>Step 1</strong> - Calculate Total FLOPs</p>
<p style="font-size: 20px;">
Size of Matrix A: 32K x 16K<br>
Size of Matrix B: 16K x 8K<br>
Size of Matrix C: 32K x 8K<br>
Each MAC operation consists of 2 FLOPs (1 mul, 1 add)<br>
<br>
Total number of operations i.e. MAC operations needed to compute Matrix C is<br>
32K x 16K x 8K<br>
<br>
Total FLOPs is: 2 x Total number of operations (since each MAC is 2 FLOPs)<br>
Total FLOPs = 2 x (32K x 16K x 8K)<br>
Total FLOPs = 2 x 32 x 16 x 8 x (2^30)<br>
<strong>Total FLOPs = 8.8 TFLOPs</strong><br>
</p>

<p style="font-size: 22px;"><strong>Step 2</strong> - Calculate Peak Compute Performance</p>
<p style="font-size: 20px;">
The peak theoretical performance for the system is<br>
Peak Perf = # of PEs x Operations/cycle per PE x PE_frequency<br>
We assumed 8 FP8 ops/cycle and Frequency as 2 GHz.<br>
Therefore, Peak Perf = 8 PEs x 8 ops/cycle x 2 x (10^9) cycles/second<br>
<strong>Peak Perf = 128 GFLOPs/second</strong><br>
</p>

<p style="font-size: 22px;"><strong>Step 3</strong> - Calculate the Time to run the workload</p>
<p style="font-size: 20px;">
Time to Compute = (Total FLOPs)/(Peak FLOPs/second) = (8.8 TFLOPs)/(128 GFLOPs/second) = 68.72 seconds.<br>
Therefore, with 8 PEs operating at peak capacity, the <strong>computation would take ~69 seconds</strong>.<br>
</p>

<p style="font-size: 22px;"><strong>Step 4</strong> - Calculate the total data access requirements</p>
<p style="font-size: 20px;">
We are multiplying 2 matrices (A, B) i.e. reading two matrices and storing the output matrix C.<br>
A: 32K x 16K<br>
B: 16K x 8K<br>
C: 32K x 8K<br>
<br>
The data type assumed is FP8 (i.e. 1 Byte per element)<br>
<br>
The total data accessed is<br>
Matrix A: 32K x 16K x 1 Byte/element = 32 x 16 x 1 MB = 512 MB<br>
Matrix B: 16K x 8K x 1 Byte/element = 16 x 8 x 1 MB = 128 MB<br>
Matrix C: 32K x 8K x 1 Byte/element = 32 x 8 x 1 MB = 256 MB<br>
<br>
Total Reads = Data accessed by Matrix A + Matrix B = 640 MB<br>
Total Writes = Data accessed by Matrix C = 256 MB<br>
<strong>Total Data</strong> (Total Reads + Total Writes) = <strong>896 MB</strong><br>
</p>

<p style="font-size: 22px;"><strong>Step 5</strong> - Bandwidth Requirements at each Layer (DRAM ←→ L2$ ←→ L1$ ←→ PE)</p>
<p style="font-size: 20px;">
<strong>DRAM → L2$</strong><br>
Let’s compute the Bandwidth requirements to move the data from DRAM to L2$.<br>
BW(DRAM to L2) = (total data to be moved)/(time to compute)<br>
BW(DRAM to L2) = (896 MB)/(68.72 seconds)<br>
BW(DRAM to L2) = <strong>13.04 MB/s</strong><br>
<br>
<strong>L2$ → L1$</strong><br>
Let’s do this in two ways. <strong>Method 1</strong> is where we assume no optimizations and <strong>Method 2</strong> is where we do ‘Tiling’ to improve performance i.e. to maximize locality and minimize latency.<br>
<br>
Let’s make some assumptions. The L1 cache is 32 KB per PE and is inclusive of L2. The L2 cache is 2 MB and is shared among all 8 PEs<br>
<br>
<strong>Method 1</strong><br>
We calculated the peak performance of the system to be 128 GFLOPs/second. To sustain this peak performance, data needs to be transferred from L2 to L1 at a rate that supports the computation rate of 128 GFLOPs/second. Therefore,<br>
<br>
Total BW(L2->L1) = Total FLOPs/second x Bytes/Operation<br>
Total BW(L2->L1) = 128 GFLOPs/second x 1 Byte/FLOP<br>
<strong>Total BW(L2->L1) = 128 GB/s</strong><br>
<br>
This is the best case scenario. In reality, the bandwidth between L2 and L1 depends on multiple factors including cache access patterns, locality of reference, contention, etc.<br>
<br>
<strong>Method 2</strong><br> 
<strong><u>Tiling</u></strong> is a technique where matrices are partitioned into smaller blocks i.e. tiles that fit within the cache hierarchy to maximize data reuse and reduce latency. This minimizes the number of times data needs to be transferred between L2 and L1 caches, improving the effective bandwidth.<br>
<br>
Pick a tile size for L1 cache such that the submatrices for A, B, and C fit within the 32KB L1 cache of each PE. The idea is to maximize the 32 KB while allowing for data reuse, reducing the number of times data needs to be transferred between L2 and L1. This helps in reducing the effective bandwidth requirement. Each tile loaded into the L1 cache from the L2 is reused multiple times for matrix multiplication before it’s replaced.<br>
<br>
Lets define an ‘Reuse Factor’ “R” that represents the average number of times a tile is used before new data is loaded from the L2.<br>
<br>
The new effective BW(L2→L1) now is: BW(L2→L1)/R<br>
<br>
If we assume R = 4, The effective BW(L2→L1) now is 128 GB/s / 4 = 32 GB/s<br>
<br>
Thus, this tiling strategy significantly reduces the BW requirement from L2 → L1 by improving data re-use which maximizes locality and minimizes latency.<br>
<br>
<strong>L1$ → PE</strong><br>
By similar logic as the above step, the BW required to sustain the Peak Performance is 128 GB/s.<br>
<br></p>

<p style="font-size: 22px;"><strong>Step 6</strong> - Summary</p>
<img src="/assets/arch_summary.jpg" alt="Accelerator 101" style="object-fit: cover; object-position: top;">
<br>
<p style="font-size: 20px;">
The calculations above are for a best case scenario, assuming ideal conditions. In a real world case, there are multiple limiters to achieving peak performance like impact of coherence and control mechanisms, access latencies, cache contention, misses, etc. Our goal as SoC performance architects is to improve/optimize the SoC construction defined by the architects.<br>
<br>
<strong>If you’re reading this and find mistakes or have suggestions, please feel free to point them out. Like I said, this is more of a learning/self-documenting exercise than anything else.</strong><br>
<br>
Thanks James (@jamesabel) for helping review!
<br></p>

<p style="font-size: 22px;"><u>Future topics I’ll probably discuss here</u></p>
<p style="font-size: 20px;">
* Compute and Cache Architectures<br>
* Network on Chip<br>
* Simulators and Performance Analysis<br>
* Workload Analysis and Projections<br>
</p>
