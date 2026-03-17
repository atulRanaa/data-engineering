# 13. Deconstructing Google: TPUv6 and Pathway

While OpenAI and the rest of the industry standardized on Nvidia GPUs, InfiniBand, and PyTorch, Google forged an entirely parallel, completely proprietary infrastructure universe. Google's 2026 AI infrastructure relies on custom Silicon (**Tensor Processing Units - TPUs**), custom optical networking, and a radically different orchestration software framework called **Pathway**.

---

## 1. The TPU: Systolic Arrays vs Von Neumann

An Nvidia GPU uses standard registers and Single Instruction, Multiple Data (SIMD) cores. It loads data, computes, and writes back.

Google's TPU is famously built on a completely different architecture known as a **Systolic Array**. 

### The Systolic Array Concept
In a 128x128 systolic matrix multiplier (the core of the TPU), the compute cores (ALUs) are wired directly to each other in a grid.
When multiplying two massive matrices to compute a Neural Network layer:
1. The memory controller injects data into the left and top edges of the grid.
2. At every clock tick, the data flows (like systemic blood pressure, hence "systolic") through the grid.
3. Core $(0,0)$ multiplies, passes the result to Core $(0,1)$, which adds its result and passes to $(0,2)$.

**The Advantage:** The TPU completely eliminates register read/write bottlenecks for Matrix multiplication. Data is read from RAM exactly *once*, pumped through the massive 16,384-core array, and outputted. TPUs run significantly hotter but mathematically destroy equivalent GPUs on pure dense matrix multiplication per watt.

---

## 2. OCS: Optical Circuit Switches

When training PaLM and Gemini models, Google realized that fixed Ethernet switches (Fat-Trees) were a bottleneck. During training, the GPUs/TPUs exhibit static communication patterns (e.g., passing gradients in a ring).

Google replaced the electronic network cores entirely with **Optical Circuit Switches (OCS) - Jupiter**.

### Mirrors Instead of Routers
Inside a Google AI data center, the core switches contain zero silicon routing chips. They are literally boxes containing microscopic, motorized **MEMS mirrors**.
- When TPU Pod A needs to send 5 Terabits of data to TPU Pod B, the central orchestrator physically tilts a microscopic mirror inside the switch.
- The laser beam from Pod A reflects off the mirror directly into the fiber optic cable connected to Pod B.

There is no packet processing. There are no IP headers. There is zero electronic latency. It is literally routing data via the speed of light bouncing off mirrors. 
This allowed Google to build 100,000+ TPU mega-clusters with perfectly balanced topologies tailored to specific Neural Network sharding parameters.

---

## 3. The Pathway Orchestrator: Multi-Modal Routing

*Source literature: "Pathways: Asynchronous Distributed Dataflow for ML" (Google Research, 2022).*

In the standard PyTorch paradigm, the Data Engineer writes a script that executes locally on `Node 0`. PyTorch uses MPI (Message Passing) to orchestrate the other thousands of GPUs. This is a Single-Program, Multiple-Data (SPMD) paradigm.

Google's **Pathway** orchestration system fundamentally reinvents this. 

Pathway treats a 10,000 TPU cluster as a single logical computer. Instead of a script launching thousands of identical workers, a centralized **Pathway Scheduler** dynamically compiles the Machine Learning graph and routes tasks across the cluster.

### Sparse Activation & Multi-Modal Routing
The goal of Pathway was to support **Sparse Expert Models (MoE)**.
In a massive multi-modal AI (like Gemini), if the model is processing a picture of a dog, it should not waste electricity activating the "Audio Processing" layers or the "French Translation" layers.

With Pathway, the data flows through the cluster geographically:
1. The image arrives at Server Rack A.
2. The Pathway graph dynamically routes the tensor to Rack D, which contains the specialized "Vision Experts".
3. The output is routed to Rack F, containing the "Reasoning Experts".

```python
# Conceptual representation of Google Pathway Orchestration
@pathway.distributed_function
def multimodal_forward_pass(input_data):
    # Pathway compiler dynamically routes evaluating this 
    # to specific specialized TPU sub-pods at runtime
    if type(input_data) == Audio:
        return audio_expert_tpus.infer(input_data)
        
    if type(input_data) == Vision:
        # Pathway physically routes the bits over the Optical network 
        # to the vision sector of the datacenter
        return vision_expert_tpus.infer(input_data)
```

Google's infrastructure philosophy is defined by vertical integration: they designed the Tensor API (JAX/TensorFlow), the compiler (XLA), the orchestration layer (Pathway), the optical network (Jupiter), and the silicon (TPU). This integration allows them to bypass the entire OSI networking model and OS stack, operating the world's most powerful AI supercomputers at peak mathematical efficiency.
