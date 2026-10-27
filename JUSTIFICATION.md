# Instance Type Justification

## Host A: Prompt Optimizer API

**Instance type chosen:** `t4g.micro`

**Workload characteristics:**

The Prompt Optimizer API handles approximately 250,000 requests per day, with traffic during business hours (9 AM–6 PM) reaching about 10 times the overnight rate. Despite these traffic fluctuations, average CPU utilization is only 8–12% during peak hours and below 2% overnight. Each process requires approximately 150 MB of memory, and the latency requirement is under 100 ms per request. Since this is an early-stage product, minimizing unnecessary infrastructure costs is an important consideration.

**Why this instance family:**

I chose the T4g burstable general-purpose family because the API has low average CPU utilization with occasional traffic spikes. As discussed in Module 6, burstable instances are suitable for workloads that do not consistently require high CPU performance.

The `t4g.micro` provides 2 vCPUs and 1 GiB of memory, making it a cost-efficient starting point for this lightweight API. It also uses an AWS Graviton processor, which can offer better price-performance than comparable x86 instances. This choice follows the principle of rightsizing cloud resources according to actual workload requirements rather than overprovisioning. The latency target and memory headroom should still be validated through load testing.

**Why not the alternatives:**

I considered `t3.micro`, which provides the same 2 vCPUs and 1 GiB of memory as `t4g.micro`. However, `t3.micro` uses x86_64 architecture and has a higher On-Demand price.

Since the provided Python application uses the standard library and supports ARM64, I chose `t4g.micro` for its better cost efficiency. A compute-optimized instance would also be unnecessary for the API's relatively low average CPU utilization.

**Cost comparison:**

Estimated Linux On-Demand compute pricing in `us-west-2`, assuming 730 hours per month:

|                               | Your choice  | Alternative considered |
| Instance type                 | `t4g.micro`  | `t3.micro` |
| Hourly rate                   | $0.0084      | $0.0104 |
| Estimated monthly rate (24/7) | $6.13        | $7.59 |

Choosing `t4g.micro` saves approximately $1.46 per month, or 19.2%, compared with `t3.micro`.

Although the absolute savings are modest, this difference supports the choice of a cost-efficient Graviton instance for an early-stage product. It also aligns with the FinOps principle of optimizing infrastructure costs by selecting resources that match actual workload requirements.

## Host B: Embedding Batch Regeneration Worker

**Instance type chosen:** `c7g.large`

**Workload characteristics:**

The worker regenerates vector embeddings for approximately 50,000 documents during a nightly batch job starting at 2 AM. Each job runs continuously for approximately 45–60 minutes, with sustained CPU utilization of 85–95%. The workload requires approximately 2 GB of working memory and does not need public internet access.

Unlike the API, this workload is CPU-intensive and maintains high utilization throughout its execution rather than experiencing short, occasional spikes.

**Why this instance family:**

I chose the C7g compute-optimized family because the worker is a CPU-bound application requiring sustained processing performance. Module 6 identifies compute-optimized instances as suitable for workloads with high CPU requirements.

The `c7g.large` provides 2 vCPUs and 4 GiB of memory, which accommodates the worker's approximately 2 GB working-memory requirement. Its AWS Graviton processor uses ARM64, which is compatible with the provided Python standard-library application.

Unlike burstable T-family instances, the C7g family provides sustained compute performance without depending on CPU credits. This makes it a predictable choice for completing nightly batch-processing jobs.

**Why not T-family specifically:**

I considered `t4g.large`, which provides 2 vCPUs and 8 GiB of memory at a slightly lower hourly price. However, T-family instances rely on CPU credits to sustain performance above their baseline. During prolonged high CPU utilization, insufficient credits may cause throttling in Standard mode or additional charges in Unlimited mode.

Although the worker runs for only 45–60 minutes each night and may accumulate enough credits between jobs, its performance depends on the available credit balance and operating mode. I chose `c7g.large` because it provides predictable sustained CPU performance without relying on CPU credits and offers sufficient memory for the workload.

**Cost comparison:**

Estimated Linux On-Demand compute pricing in `us-west-2`, assuming 730 hours per month:
 
|                               | Your choice | Alternative considered |
| Instance type                 | `c7g.large` | `t4g.large` |
| Hourly rate                   | $0.0725     | $0.0672 |
| Estimated monthly rate (24/7) | $52.93      | $49.06 |

The `c7g.large` costs approximately $3.87 more per month than `t4g.large` when both run continuously.

This relatively small difference is reasonable for a CPU-intensive workload that benefits from predictable sustained performance. The compute-optimized instance also better matches the worker's CPU-to-memory requirements.

**Realistic runtime consideration:**

The worker only needs to process documents for approximately 45–60 minutes each night. Therefore, running the EC2 instance continuously would waste resources during idle hours.

If the `c7g.large` runs 24/7, the estimated monthly compute cost is:

730 hours × $0.0725/hour = **$52.93 per month**.

If the instance runs for approximately one hour per night over 30 days:

30 hours × $0.0725/hour = **$2.18 per month**.

For 45-minute nightly runs, the estimated compute cost would be approximately $1.63 per 30-day month, excluding startup and shutdown overhead.

To achieve these savings in practice, the instance could be automatically started before the nightly batch job and stopped afterward using Amazon EventBridge Scheduler and AWS Lambda or another scheduling mechanism. This would require additional automation that is not part of the current Terraform implementation.

EBS storage and other applicable infrastructure charges must also be considered separately.

This approach follows the FinOps principles discussed in Module 6, particularly rightsizing resources and eliminating unnecessary idle compute costs.

## AI Tool Usage

I asked ChatGPT to analyze the workload characteristics of Host A and Host B, compare suitable EC2 instance types, and explain the advantages and disadvantages of each option in terms of CPU performance, memory, architecture, and cost.

For Host A, I compared `t3.micro` and `t4g.micro`. Both are burstable instances suitable for the API's low average CPU utilization and fluctuating traffic. The `t3.micro` offers broad x86 compatibility, while the `t4g.micro` uses AWS Graviton processors and has a lower On-Demand price. Since the provided application supports ARM64, I chose `t4g.micro` for better cost efficiency.

For Host B, I compared `t4g.large` and `c7g.large`. The `t4g.large` is slightly cheaper and provides more memory, but it relies on CPU credits for performance above its baseline. The `c7g.large` provides sustained compute performance without CPU-credit limitations, making it more suitable for the worker's consistently high CPU utilization.

Before finalizing my choices, I reviewed the concepts covered in Module 6, including instance families, CPU credits, Graviton architecture, and rightsizing. I also verified the instance specifications and On-Demand hourly and monthly prices using the official AWS Pricing Calculator for the `us-west-2` region. Additionally, I confirmed that both selected instance types use ARM64 and that my Terraform configuration references the matching Ubuntu ARM64 AMI.

Based on these comparisons, I selected `t4g.micro` for Host A and `c7g.large` for Host B because they best match the respective workload requirements while balancing performance and cost.

## References

Fannin, D. (2026). *Module 6: Cloud Compute and Virtualization*. Northeastern University, CSYE 6225.

Amazon Web Services. (n.d.). *Amazon EC2 On-Demand Pricing*. https://aws.amazon.com/ec2/pricing/on-demand/

Amazon Web Services. (n.d.). *AWS Pricing Calculator*. https://calculator.aws/

Amazon Web Services. (n.d.). *Amazon EC2 Instance Types*. https://aws.amazon.com/ec2/instance-types/