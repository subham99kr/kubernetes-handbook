# Debugging Framework

> **Difficulty:** ⭐⭐⭐⭐ Advanced
>
> **Prerequisites**
>
> - Kubernetes Fundamentals
> - Core Objects
> - Networking
> - Storage
>
> **Related Chapters**
>
> - CrashLoopBackOff
> - Pending Pods
> - DNS Issues
> - Storage Issues
> - Root Cause Analysis
>
> **Purpose**
>
> This chapter teaches a systematic approach to debugging Kubernetes incidents. Instead of focusing on a specific error, you'll learn how to isolate the failing component, collect evidence, verify hypotheses, and identify the root cause.

---

# Learning Objectives

After completing this chapter, you'll be able to:

- Think like an SRE during incidents.
- Avoid random troubleshooting.
- Isolate the failing Kubernetes component.
- Collect useful evidence.
- Build and verify hypotheses.
- Navigate to the correct troubleshooting playbook.
- Perform structured Root Cause Analysis (RCA).

---

# Why This Chapter Matters

Most Kubernetes books teach you **how Kubernetes works**.

Very few teach you **how to debug Kubernetes when it stops working.**

Imagine you're on call at **2:00 AM**.

Your phone rings.

> 🚨 **Production Alert**
>
> Users are reporting that the website is down.

That is all the information you have.

Immediately, dozens of questions come to mind.

- Is the application crashing?
- Is Kubernetes unhealthy?
- Is the Service broken?
- Is DNS failing?
- Is the Ingress misconfigured?
- Is the database unavailable?
- Did someone deploy a bad release?
- Is the node out of memory?

At this moment, **you do not know the answer to any of these questions.**

The only thing you know is:

> Something is broken.

---

# The Biggest Mistake Beginners Make

Most engineers immediately start changing things.

Examples:

```bash
kubectl delete pod ...
```

```bash
kubectl rollout restart deployment ...
```

```bash
kubectl apply -f deployment.yaml
```

Sometimes these actions appear to fix the problem.

But in reality, they often destroy the evidence needed to understand what actually happened.

A restarted Pod may erase important logs.

A new Deployment may hide the original configuration issue.

Deleting resources can make root cause analysis much harder.

A professional troubleshooter always **investigates first and changes later.**

---

# The Goal of Debugging

Many people believe debugging means fixing an issue.

That is only partly true.

The real objective is:

> **Identify the root cause with evidence, then apply the correct fix.**

Fixing symptoms without understanding the cause usually results in recurring incidents.

For example:

```text
CrashLoopBackOff
```

is **not** the problem.

It is only a symptom.

The real cause might be:

- Missing Secret
- Invalid ConfigMap
- Database unavailable
- Out of memory
- Application bug
- Wrong startup command

Your job is to move from:

```text
Symptom
        ↓
Evidence
        ↓
Root Cause
        ↓
Fix
```

---

# Think Like an SRE

A good Site Reliability Engineer follows a disciplined process.

They do **not** panic.

They do **not** guess.

They do **not** restart everything.

Instead, they investigate methodically.

Keep these principles in mind during every incident.

## 1. Observe Before Acting

Never assume the first error message is the root cause.

Collect information before making changes.

---

## 2. Evidence Beats Assumptions

Logs.

Events.

Metrics.

Resource status.

These are facts.

Guesses are not.

---

## 3. Every Symptom Has a Cause

Examples:

```text
CrashLoopBackOff
```

is a symptom.

```text
OOMKilled
```

is a symptom.

```text
503 Service Unavailable
```

is a symptom.

Your task is to discover **why** they occurred.

---

## 4. Change as Little as Possible

Every change destroys evidence.

Restarting a Pod may remove useful logs.

Deleting a Deployment may hide the original configuration.

Always collect evidence first.

---

## 5. Verify Every Conclusion

Never say:

> "I think DNS is broken."

Instead prove it.

Run commands.

Collect evidence.

Verify your hypothesis.

Professional debugging is based on verification—not intuition.

---

# The Universal Debugging Algorithm

Every Kubernetes incident can be investigated using the same process.

```text
Incident

↓

Observe

↓

Isolate the Failing Component

↓

Collect Evidence

↓

Build Hypotheses

↓

Verify Each Hypothesis

↓

Identify Root Cause

↓

Apply Fix

↓

Validate Recovery

↓

Document & Prevent
```

This algorithm forms the foundation of every troubleshooting playbook in this handbook.

Whether you're debugging:

- CrashLoopBackOff
- Pending Pods
- DNS failures
- Storage problems
- API Server outages

…the investigation always follows the same workflow.

---

# One Question Changes Everything

When an incident occurs, beginners ask:

> "Which command should I run?"

Experienced SREs ask:

> **Which component is actually failing?**

That single question determines the entire investigation.

Before collecting logs or restarting Pods, you must first identify **where the failure exists**.

In the next section, we'll divide Kubernetes into **Failure Domains** and learn how to isolate the problem before diving into specific troubleshooting playbooks.
---

# Kubernetes Failure Domains

A Kubernetes cluster is made up of many independent components.

When something goes wrong, **the entire cluster is rarely broken**.

Usually, only one or two components are responsible for the incident.

Your first task is **not to fix the problem**.

Your first task is to **identify which component is failing.**

This process is called **Failure Isolation**.

---

# What is a Failure Domain?

A **Failure Domain** is a logical part of Kubernetes that can fail independently.

For example:

- Pods can fail while networking works perfectly.
- DNS can fail while Pods continue running.
- Storage can fail while the application is healthy.
- The Scheduler can fail while existing applications continue serving traffic.

Instead of debugging the entire cluster, narrow the investigation to the affected failure domain.

---

# The Kubernetes Stack

Think of every request as travelling through several layers.

```text
                    User
                      │
                      ▼
                 External DNS
                      │
                      ▼
             Load Balancer (Optional)
                      │
                      ▼
          Ingress Controller / Gateway
                      │
                      ▼
                  Ingress Rules
                      │
                      ▼
                   Service
                      │
                      ▼
                EndpointSlice
                      │
                      ▼
                     Pod
                      │
                      ▼
                 Container
                      │
                      ▼
                Application
                      │
                      ▼
             Database / Storage
```

A failure in **any one** of these layers can make the application appear "down."

Your goal is to identify **where the request stops**.

---

# The Golden Rule

Never troubleshoot everything.

Instead ask:

> **Where does the request stop?**

Suppose a user cannot access the website.

The request might fail at:

```text
DNS

OR

Ingress

OR

Service

OR

Pod

OR

Application

OR

Database
```

These are completely different problems.

Treating them as the same incident wastes time.

---

# Major Failure Domains

For practical troubleshooting, we can group Kubernetes into six major domains.

```text
1. Control Plane

2. Worker Node

3. Networking

4. Workloads

5. Storage

6. Application
```

Every production issue usually belongs to one of these domains.

Let's briefly understand each one.

---

# 1. Control Plane

Responsible for managing the cluster.

Components include:

- API Server
- Scheduler
- Controller Manager
- etcd

Typical symptoms:

- kubectl not responding
- Pods not scheduling
- Controllers not creating resources
- Cluster-wide failures

Typical playbooks:

- API Server Failure
- Scheduler Failure
- etcd Failure

---

# 2. Worker Node

Responsible for running workloads.

Components include:

- kubelet
- Container Runtime
- Operating System

Typical symptoms:

- Node NotReady
- Pods Evicted
- Disk Pressure
- Memory Pressure
- Runtime failures

Typical playbooks:

- Node NotReady
- Evicted
- DiskPressure

---

# 3. Networking

Responsible for communication.

Components include:

- CNI
- kube-proxy
- CoreDNS
- Services
- Ingress

Typical symptoms:

- Connection timeout
- Service unreachable
- DNS lookup failure
- Cross-Pod communication failure

Typical playbooks:

- DNS Issues
- Service Not Reachable
- Network Policy
- CNI Failure

---

# 4. Workloads

Responsible for running your applications.

Includes:

- Pods
- Deployments
- StatefulSets
- Jobs

Typical symptoms:

- Pending
- CrashLoopBackOff
- ImagePullBackOff
- ContainerCreating
- Terminating

Typical playbooks:

- Pending Pods
- CrashLoopBackOff
- ImagePullBackOff
- OOMKilled

---

# 5. Storage

Responsible for persistent data.

Includes:

- Persistent Volumes
- PVCs
- StorageClass
- CSI Drivers

Typical symptoms:

- PVC Pending
- Volume Mount Failure
- Missing Data
- Read-only Filesystem

Typical playbooks:

- PVC Pending
- Volume Mount Failure
- CSI Issues

---

# 6. Application

Responsible for the business logic.

Kubernetes may be healthy while the application itself is failing.

Typical symptoms:

- HTTP 500
- Database connection errors
- Authentication failures
- High response time
- Memory leaks

Typical playbooks:

- Application Logs
- Configuration Errors
- Database Connectivity

---

# Visualizing Failure Isolation

Instead of thinking:

```text
Application is Down
```

Think:

```text
Application is Down

↓

Which Layer Failed?

↓

Networking?

↓

Storage?

↓

Pods?

↓

Application?

↓

Control Plane?
```

Notice how the investigation becomes much smaller.

Instead of searching the entire cluster, you're searching only one failure domain.

---

# A Real Example

Imagine users report:

> "The website is not loading."

At this point, there are dozens of possible causes.

A beginner may immediately restart the Deployment.

An experienced SRE starts by asking:

```text
Is the application running?

↓

Can I reach the Service?

↓

Are Endpoints available?

↓

Are Pods healthy?

↓

Can the application answer requests?

↓

Can it reach the database?
```

Each answer removes entire categories of possible failures.

This is called **progressive elimination**.

---

# Failure Isolation is Like Medical Diagnosis

Imagine a patient says:

> "I have chest pain."

A doctor doesn't immediately prescribe medicine.

They ask questions.

They perform tests.

They eliminate possibilities one by one.

Eventually they identify the actual cause.

Kubernetes troubleshooting works exactly the same way.

Never guess.

Always eliminate possibilities using evidence.

---

# What's Next?

Now that we've divided Kubernetes into failure domains, the next step is learning a **universal investigation workflow** that tells us exactly how to isolate the failing component and choose the correct troubleshooting playbook.

---

# Universal Investigation Workflow

Every production incident—whether it's a Pod crash, a DNS issue, or a storage problem—can be investigated using the same structured workflow.

Experienced SREs don't memorize hundreds of solutions.

They follow a repeatable process.

The goal is to reduce a large, complex problem into a small, well-defined one.

---

# The Investigation Algorithm

Whenever an incident occurs, follow this sequence:

```text
Incident Report
        │
        ▼
Understand the Symptom
        │
        ▼
Identify the Affected Component
        │
        ▼
Collect Evidence
        │
        ▼
Build Possible Hypotheses
        │
        ▼
Verify Each Hypothesis
        │
        ▼
Identify Root Cause
        │
        ▼
Apply the Fix
        │
        ▼
Validate Recovery
        │
        ▼
Document the RCA
```

Never skip a step.

Skipping directly to a fix often creates new problems or hides the original cause.

---

# Step 1 — Understand the Symptom

Every investigation starts with one question:

> **What exactly is failing?**

Not:

> "What do I think is wrong?"

Instead, describe the problem using observable facts.

Examples:

Good:

```text
Users receive HTTP 503.

Pods are in Pending state.

DNS lookup fails.

kubectl get pods shows CrashLoopBackOff.

PVC remains Pending.
```

Bad:

```text
Kubernetes is broken.

The cluster is unstable.

Networking is probably failing.
```

Facts first.

Assumptions later.

---

# Step 2 — Identify the Affected Component

Once the symptom is clear, determine which Kubernetes component owns that symptom.

Examples:

| Symptom | Likely Component |
|---------|------------------|
| Pending | Scheduler |
| ImagePullBackOff | Container Runtime / Registry |
| CrashLoopBackOff | Application / Container |
| Service unavailable | Service / EndpointSlice |
| DNS failure | CoreDNS |
| PVC Pending | Storage |
| Node NotReady | kubelet / Node |

Notice something important:

One symptom usually points to **one primary component**.

This dramatically reduces the scope of the investigation.

---

# Step 3 — Collect Evidence

Only after identifying the affected component should you begin collecting evidence.

Evidence may include:

- Events
- Logs
- Resource status
- Metrics
- Node conditions
- YAML configuration

At this stage, **do not modify anything**.

Your objective is to understand the current state of the system.

---

# Step 4 — Build Hypotheses

Once enough evidence has been collected, list the possible explanations.

For example:

```
Pod is Pending
```

Possible causes:

- No available CPU
- No available Memory
- PVC Pending
- Node Selector mismatch
- Taints
- Unschedulable node

Do not assume the first explanation is correct.

List all reasonable possibilities.

---

# Step 5 — Verify Each Hypothesis

Now eliminate possibilities one by one.

Example:

Hypothesis:

```
The Pod cannot be scheduled because there isn't enough CPU.
```

Verification:

```bash
kubectl describe pod <pod-name>
```

If the Events section reports:

```
0/3 nodes available:
Insufficient CPU
```

The hypothesis is confirmed.

If not, reject it and move to the next possibility.

Never treat a hypothesis as a fact until you've verified it.

---

# Step 6 — Identify the Root Cause

Once a hypothesis is verified, identify the underlying reason.

Example:

Symptom:

```
CrashLoopBackOff
```

Root Cause:

```
Missing database password Secret.
```

Notice the difference.

The symptom is **CrashLoopBackOff**.

The root cause is **missing configuration**.

Fixing the symptom without fixing the cause only delays the next failure.

---

# Step 7 — Apply the Fix

Only after identifying the root cause should changes be made.

Examples:

- Correct a ConfigMap.
- Create the missing Secret.
- Increase available resources.
- Fix the container image.
- Update the Service selector.

The fix should target the **root cause**, not merely the symptom.

---

# Step 8 — Validate Recovery

Never assume the issue is resolved because the error disappeared.

Verify that:

- Pods are healthy.
- Services are reachable.
- Users can access the application.
- Metrics return to normal.
- Alerts stop firing.

A successful fix restores the system—not just the resource.

---

# Step 9 — Document the Incident

Every incident is an opportunity to improve.

Record:

- What happened
- Root cause
- Evidence collected
- Fix applied
- How similar incidents can be prevented

This becomes part of your operational knowledge and future playbooks.

---

# Remember

The workflow never changes.

Whether you're debugging:

- CrashLoopBackOff
- DNS Issues
- API Server failures
- PVC problems
- Node failures

…the investigation always follows the same sequence.

Only the evidence and verification steps differ.

---

# Master Debugging Decision Tree

One of the biggest challenges during an incident is deciding **where to begin**.

Many engineers immediately jump into logs, restart Pods, or redeploy applications.

Instead, use this decision tree.

It guides you to the correct failure domain before you begin troubleshooting.

Think of it as a GPS for Kubernetes incidents.

---

# Step 1 — What is the Problem?

Every investigation starts with a simple question.

```text
Something is Broken
        │
        ▼
What is failing?
```

Choose the closest symptom.

```text
Application unavailable

Pods won't start

Application is slow

Cannot access cluster

Storage problem

Node problem

Network problem
```

Each symptom leads to a different investigation.

---

# Scenario 1 — Application is Unavailable

```
Application Down
        │
        ▼
Can users reach the application?
        │
   ┌────┴────┐
   │         │
  No        Yes
   │         │
   ▼         ▼
Ingress   Application Issue
```

If users cannot even reach the application,

investigate:

- DNS
- Load Balancer
- Ingress
- Service

If users reach the application but receive errors,

investigate:

- Application
- Database
- Configuration

---

# Scenario 2 — Pod Will Not Start

```
Pod Not Running
        │
        ▼
kubectl get pods
        │
        ▼
Current Status?
```

Then follow the appropriate branch.

```
Pending
        │
        ▼
Pending.md
```

```
ContainerCreating
        │
        ▼
ContainerCreating.md
```

```
ImagePullBackOff
        │
        ▼
ImagePullBackOff.md
```

```
CrashLoopBackOff
        │
        ▼
CrashLoopBackOff.md
```

```
OOMKilled
        │
        ▼
OOMKilled.md
```

```
Evicted
        │
        ▼
Evicted.md
```

```
Terminating
        │
        ▼
Terminating.md
```

Notice something important.

You haven't investigated anything yet.

You've simply identified the correct playbook.

---

# Scenario 3 — Service Doesn't Work

```
Service Unreachable
        │
        ▼
Are Pods Running?
```

If:

```
No
```

↓

This is **not** a Service problem.

Investigate Pods.

If:

```
Yes
```

↓

Continue.

```
Do Endpoints Exist?
```

If:

```
No
```

↓

Service Selector

↓

EndpointSlice

↓

Labels

If:

```
Yes
```

↓

Continue.

```
Can Pods talk to each other?
```

If:

```
No
```

↓

Networking

↓

NetworkPolicy

↓

CNI

↓

CoreDNS

If:

```
Yes
```

↓

Application issue.

---

# Scenario 4 — Storage Problems

```
PVC Pending
        │
        ▼
StorageClass Exists?
        │
        ▼
Persistent Volume Available?
        │
        ▼
CSI Driver Healthy?
        │
        ▼
Storage Playbooks
```

---

# Scenario 5 — Node Problems

```
Node NotReady
        │
        ▼
kubectl describe node
        │
        ▼
Check Conditions
```

Possible causes:

- MemoryPressure
- DiskPressure
- PIDPressure
- kubelet
- Container Runtime

Go to:

```
Node_NotReady.md
```

---

# Scenario 6 — Cluster Problems

```
kubectl not responding
        │
        ▼
API Server Reachable?
```

If:

```
No
```

↓

Control Plane.

Possible causes:

- API Server
- etcd
- Certificates

If:

```
Yes
```

↓

Continue investigating workloads.

---

# The Golden Rule

Always investigate **top-down**.

```
User

↓

DNS

↓

Ingress

↓

Service

↓

EndpointSlice

↓

Pod

↓

Container

↓

Application

↓

Database
```

The first layer that fails is usually where your investigation should focus.

---

# Never Skip Layers

Bad approach:

```
Website Down

↓

Restart Deployment
```

Good approach:

```
Website Down

↓

Ingress

↓

Service

↓

Endpoints

↓

Pods

↓

Logs

↓

Root Cause
```

---

# Mapping Symptoms to Playbooks

| Symptom | Playbook |
|----------|----------|
| Pending | Pending.md |
| CrashLoopBackOff | CrashLoopBackOff.md |
| ImagePullBackOff | ImagePullBackOff.md |
| OOMKilled | OOMKilled.md |
| Evicted | Evicted.md |
| ContainerCreating | ContainerCreating.md |
| Service Unreachable | Service_Not_Reachable.md |
| DNS Failure | DNS.md |
| PVC Pending | PVC_Pending.md |
| Node NotReady | Node_NotReady.md |
| API Server Down | API_Server.md |

This table is your **navigation map**.

You don't memorize solutions.

You identify the symptom and jump directly to the relevant playbook.

---

# Key Takeaways

- Every Kubernetes incident starts with identifying the symptom.
- Never troubleshoot the entire cluster at once.
- Follow the decision tree to isolate the failing component.
- Use the corresponding playbook for detailed diagnosis.
- Move from **Symptom → Component → Evidence → Root Cause → Fix**.

---

# Evidence Collection

Once you've isolated the failure domain, the next step is to collect evidence.

This is the most important phase of debugging.

Without evidence, you're guessing.

With evidence, you're investigating.

Remember:

> **Evidence first. Changes later.**

---

# Why Evidence Matters

Imagine these two engineers.

### Engineer A

```text
Website Down

↓

Restart Deployment

↓

Website Works
```

Problem solved?

No.

Nobody knows:

- Why it failed
- Whether it will happen again
- What actually caused it

---

### Engineer B

```text
Website Down

↓

Collect Evidence

↓

Identify Root Cause

↓

Apply Fix

↓

Validate

↓

Document
```

Now the team knows:

- What happened
- Why it happened
- How to prevent it

This is the SRE approach.

---

# What Counts as Evidence?

In Kubernetes, evidence comes from many sources.

Examples include:

- Resource Status
- Events
- Logs
- YAML Configuration
- Metrics
- Node Conditions
- Network Connectivity

Each provides a different piece of the puzzle.

---

# Evidence Collection Order

Always collect evidence in roughly this order.

```text
Resource Status
        │
        ▼
Events
        │
        ▼
Describe Output
        │
        ▼
Container Logs
        │
        ▼
Metrics
        │
        ▼
Configuration
        │
        ▼
Network Tests
```

This order works for most production incidents.

---

# 1. Resource Status

Start by asking:

> **What is the current state?**

Useful commands:

```bash
kubectl get pods

kubectl get deployments

kubectl get services

kubectl get nodes

kubectl get pvc
```

Look for:

- STATUS
- READY
- RESTARTS
- AGE

Example:

```text
NAME          READY   STATUS             RESTARTS

frontend      0/1     CrashLoopBackOff   12
```

The status often tells you which playbook to open.

---

# 2. Events

Events tell you **what Kubernetes tried to do**.

Example:

```bash
kubectl describe pod frontend
```

Scroll to:

```
Events:
```

Typical messages:

```text
FailedScheduling

FailedMount

Back-off restarting container

Pulled image

Created container
```

Events are one of the fastest ways to identify scheduling and startup problems.

---

# 3. Describe Output

The `describe` command provides detailed information about a resource.

Useful commands:

```bash
kubectl describe pod

kubectl describe node

kubectl describe pvc

kubectl describe deployment
```

Things to check:

- Events
- Conditions
- Mounted Volumes
- Environment Variables
- Node Assignment
- Image
- Resource Requests

---

# 4. Container Logs

If the container starts, logs are often the most valuable source of information.

```bash
kubectl logs <pod-name>
```

For restarted containers:

```bash
kubectl logs --previous <pod-name>
```

Look for:

- Stack traces
- Exceptions
- Missing configuration
- Authentication failures
- Database connection errors

Remember:

Kubernetes may be healthy.

The application itself may be failing.

---

# 5. Metrics

Metrics answer questions like:

- Is CPU exhausted?
- Is memory exhausted?
- Is the node overloaded?

Useful commands:

```bash
kubectl top pod

kubectl top node
```

Look for:

- High CPU
- High Memory
- Uneven resource usage

Metrics often explain issues like:

- OOMKilled
- Slow applications
- Scheduling failures

---

# 6. Configuration

Sometimes the problem isn't runtime.

It's configuration.

Check:

- ConfigMaps
- Secrets
- Deployment YAML
- Service selectors
- Ingress rules

Ask yourself:

- Is the image correct?
- Is the tag correct?
- Are environment variables present?
- Are labels correct?
- Does the Service selector match the Pods?

Configuration mistakes are among the most common production issues.

---

# 7. Network Tests

If everything appears healthy but communication still fails, investigate networking.

Examples:

- DNS lookup
- Service connectivity
- Pod-to-Pod communication
- External connectivity

Network testing usually comes **after** verifying that the workloads themselves are healthy.

---

# Evidence Checklist

During almost every investigation, collect answers to these questions:

```text
✓ What is the current status?

✓ What events occurred?

✓ What do the logs say?

✓ What changed recently?

✓ Is resource usage normal?

✓ Is configuration correct?

✓ Can components communicate?
```

If you cannot answer these questions, you probably don't have enough evidence yet.

---

# Common Mistakes

❌ Restarting Pods before collecting logs.

Logs from the failed container may be lost.

---

❌ Ignoring Events.

Many scheduling and startup failures are clearly explained in the Events section.

---

❌ Looking only at logs.

Logs tell only part of the story.

Always combine them with Events, Metrics, and Resource Status.

---

# Golden Rule

Never ask:

> "How do I fix it?"

First ask:

> **"What evidence do I have?"**

The quality of your evidence determines the quality of your diagnosis.

---

# Building & Verifying Hypotheses

At this point you have:

- Identified the failing component.
- Collected evidence.
- Narrowed down the failure domain.

Now comes the most important step.

> **Finding the actual root cause.**

This is done by building and verifying hypotheses.

---

# What is a Hypothesis?

A hypothesis is a possible explanation for the observed behavior.

Notice the word **possible**.

It is **not** a fact.

For example:

```
Pod is Pending.
```

Possible hypotheses:

- Not enough CPU
- Not enough Memory
- PVC Pending
- Node Selector mismatch
- Taints prevent scheduling
- Node is cordoned

Each explanation is reasonable.

Only one (or sometimes more than one) will be correct.

---

# Never Fall in Love With Your First Idea

One of the biggest mistakes beginners make is this:

```
Error appears

↓

"I know what's wrong."

↓

Start fixing
```

Instead:

```
Evidence

↓

Possible Causes

↓

Verification

↓

Conclusion
```

Good troubleshooting is the process of **eliminating incorrect explanations** until only the correct one remains.

---

# Example 1 — Pending Pod

Evidence:

```
STATUS

Pending
```

Possible hypotheses:

```
Insufficient CPU

Insufficient Memory

PVC Pending

Node Selector

Taints

Unschedulable Node
```

Now verify each one.

Example:

```bash
kubectl describe pod frontend
```

Events:

```
0/4 nodes available:

Insufficient CPU
```

Now you have evidence.

The hypothesis is confirmed.

---

# Example 2 — CrashLoopBackOff

Evidence:

```
CrashLoopBackOff
```

Possible explanations:

```
Application Crash

Missing Secret

Invalid ConfigMap

Database Connection Failure

Wrong Startup Command

OOMKilled
```

Check logs.

```
Database connection refused
```

Now check:

Can the application reach the database?

If yes

↓

Application bug.

If no

↓

Database or networking problem.

Notice how one answer leads to another question.

---

# Example 3 — Service Unreachable

Evidence:

```
Service exists.
```

Possible explanations:

```
Wrong selector

No Endpoints

Pods not Ready

NetworkPolicy

DNS

Application
```

Check:

```bash
kubectl get endpoints
```

Output:

```
No Endpoints
```

Now the investigation becomes much smaller.

You don't need to investigate DNS.

You don't need to investigate Ingress.

You only need to understand why the Service has no endpoints.

---

# Eliminate, Don't Guess

Professional troubleshooting is a process of elimination.

Example:

```
Website Down

↓

Pods Running?

YES

↓

Service Exists?

YES

↓

Endpoints?

YES

↓

DNS Works?

YES

↓

Application Responds?

NO

↓

Application Bug
```

Every answer removes an entire category of possible failures.

---

# One Piece of Evidence is Never Enough

Never conclude:

```
Pod Restarting

↓

Memory Problem
```

Instead ask:

```
Do metrics support this?

↓

Was the Pod OOMKilled?

↓

Are logs consistent?

↓

Are Events consistent?
```

Multiple pieces of evidence should point to the same conclusion.

---

# Correlation is Not Causation

Imagine:

```
Pod restarted

↓

CPU usage increased
```

Did high CPU cause the restart?

Maybe.

Maybe not.

Perhaps:

```
Application entered an infinite loop

↓

CPU increased

↓

Pod restarted
```

Or:

```
OOMKilled

↓

Pod restarted

↓

CPU normal
```

Always determine **cause**, not just timing.

---

# Keep Asking "Why?"

This is one of the simplest and most effective debugging techniques.

Example:

```
Website Down

↓

Why?

↓

Pods CrashLooping

↓

Why?

↓

Database unavailable

↓

Why?

↓

Wrong Secret

↓

Why?

↓

Incorrect deployment configuration
```

By repeatedly asking **Why?**, you move from symptoms to the actual root cause.

---

# Verify Before You Fix

Never fix something that hasn't been verified.

Example:

```
I think DNS is broken.
```

Verify:

```bash
nslookup service-name
```

If DNS works,

don't waste time changing CoreDNS.

Move on.

---

# When Do You Have the Root Cause?

You have found the root cause when all of the following are true:

- It explains every observed symptom.
- It is supported by evidence.
- It can be reproduced or verified.
- Fixing it resolves the incident.

If any of these conditions are missing,

keep investigating.

---

# The Scientific Method

Good troubleshooting follows the same process as scientific research.

```
Observation

↓

Question

↓

Hypothesis

↓

Experiment

↓

Result

↓

Conclusion
```

This is why experienced SREs solve incidents faster.

They don't have better intuition.

They have a better process.

---

# Key Takeaways

- Never assume your first hypothesis is correct.
- Build multiple possible explanations.
- Verify each hypothesis with evidence.
- Eliminate possibilities one by one.
- Keep asking "Why?" until you reach the true root cause.
- A fix is only correct if it addresses the verified root cause.

---

# Root Cause Analysis (RCA)

Finding the root cause is not the end of an incident.

Professional SRE teams always ask one final question:

> **How do we prevent this from happening again?**

This process is called **Root Cause Analysis (RCA).**

An RCA is a structured investigation that explains:

- What happened?
- Why did it happen?
- How was it detected?
- How was it fixed?
- How can it be prevented?

The goal of an RCA is **learning**, not assigning blame.

---

# The 5 Whys Technique

One of the simplest RCA methods is called the **5 Whys**.

Keep asking **"Why?"** until you reach the underlying cause.

Example:

```
Website Down

↓

Why?

↓

Pods CrashLooping

↓

Why?

↓

Application couldn't connect to Database

↓

Why?

↓

Database password missing

↓

Why?

↓

Secret wasn't created

↓

Why?

↓

Deployment pipeline skipped Secret creation
```

Notice the difference.

The root cause was **not CrashLoopBackOff**.

The root cause was a deployment process issue.

---

# Production Walkthrough

Let's investigate a complete incident.

---

## Incident Report

```
09:12 AM

Users report:

"The website is down."
```

No other information is available.

Where do we begin?

---

## Step 1 — Observe

Current symptom:

```
Website inaccessible.
```

Don't make assumptions.

Collect facts.

---

## Step 2 — Isolate

Follow the investigation workflow.

```
Website

↓

Ingress

↓

Service

↓

Endpoints

↓

Pods
```

Check Pods.

```bash
kubectl get pods
```

Output:

```
frontend

CrashLoopBackOff
```

The investigation is now focused on the Pod.

Ignore DNS.

Ignore Ingress.

Ignore Services.

---

## Step 3 — Collect Evidence

Describe the Pod.

```bash
kubectl describe pod frontend
```

Events:

```
Back-off restarting failed container
```

Now inspect logs.

```bash
kubectl logs frontend
```

Output:

```
Database authentication failed.
```

Excellent.

We now have useful evidence.

---

## Step 4 — Build Hypotheses

Possible causes:

```
Wrong Secret

Database Down

Wrong Username

Wrong Password

Network Issue
```

Notice we still haven't chosen one.

---

## Step 5 — Verify

Check Secret.

```bash
kubectl get secret
```

Database Secret exists.

Check Deployment.

```
DB_PASSWORD

↓

secretKeyRef

↓

db-secret
```

Looks correct.

Check Secret value.

Password is outdated.

Now compare it with the actual database password.

They don't match.

Hypothesis confirmed.

---

## Step 6 — Root Cause

The application failed because it used an outdated database password stored in Kubernetes Secret.

CrashLoopBackOff was only a symptom.

---

## Step 7 — Apply the Fix

Update the Secret.

Restart the Deployment.

```bash
kubectl rollout restart deployment frontend
```

Pods become healthy.

Application becomes available.

---

## Step 8 — Validate

Verify everything.

```bash
kubectl get pods

kubectl get svc

kubectl logs

curl application
```

Users confirm the website is working.

Incident resolved.

---

## Step 9 — Prevent

Now improve the system.

Possible improvements:

- Rotate credentials safely.
- Automate Secret synchronization.
- Add monitoring for failed database authentication.
- Document the deployment process.
- Improve CI/CD validation.

A good incident should make the system stronger.

---

# Incident Timeline

```
09:12

Website Down

↓

09:13

Pods Checked

↓

09:14

CrashLoopBackOff Found

↓

09:15

Logs Collected

↓

09:16

Secret Verified

↓

09:18

Password Corrected

↓

09:19

Deployment Restarted

↓

09:20

Application Healthy
```

Notice how every action was based on evidence.

There was no guessing.

---

# RCA Template

Every incident can be documented using the following template.

| Section | Description |
|----------|-------------|
| Incident | What happened? |
| Impact | Who was affected? |
| Detection | How was it discovered? |
| Timeline | Important events |
| Root Cause | Verified cause |
| Resolution | What fixed it? |
| Prevention | How to stop recurrence |

This format is widely used in production environments.

---

# The Golden Rules of Troubleshooting

Throughout this chapter, we've followed a few simple principles.

Remember them during every incident.

✅ Understand the symptom before acting.

✅ Isolate the failing component.

✅ Collect evidence before making changes.

✅ Build multiple hypotheses.

✅ Verify every hypothesis.

✅ Fix the root cause, not the symptom.

✅ Validate the solution.

✅ Document the incident.

Following these rules consistently will solve most Kubernetes incidents faster and with far greater confidence.

---

# What's Next?

The remaining troubleshooting chapters are **playbooks**.

Each playbook focuses on a specific incident such as:

- CrashLoopBackOff
- Pending Pods
- ImagePullBackOff
- DNS Issues
- PVC Pending
- Node NotReady

Every playbook follows the exact framework you learned in this chapter:

```
Symptom

↓

Failure Domain

↓

Evidence

↓

Hypotheses

↓

Verification

↓

Root Cause

↓

Fix

↓

Prevention
```

You now have a repeatable debugging process that can be applied to almost every Kubernetes incident.

---

# Interview Questions

## Beginner

### 1. What is the first thing you do during a Kubernetes incident?

**Answer:**

Do **not** restart Pods or redeploy applications immediately.

Instead:

- Understand the symptom.
- Identify the failing component.
- Collect evidence.
- Build hypotheses.
- Verify the root cause.
- Apply the appropriate fix.

---

### 2. Why shouldn't you restart Pods immediately?

Restarting Pods can:

- Destroy valuable logs.
- Hide the original problem.
- Delay root cause analysis.
- Make intermittent issues harder to diagnose.

Always collect evidence first.

---

### 3. What is the difference between a symptom and a root cause?

Example:

Symptom:

```
CrashLoopBackOff
```

Root Cause:

```
Application cannot connect to the database because of an incorrect Secret.
```

A symptom tells you **what happened**.

A root cause explains **why it happened**.

---

### 4. Why are Events important?

Events record what Kubernetes attempted to do.

Examples:

- FailedScheduling
- FailedMount
- Pulled Image
- Created Container

Events often explain scheduling and startup failures.

---

## Intermediate

### 5. Explain your Kubernetes debugging workflow.

A good answer:

```
Understand the symptom

↓

Identify the failure domain

↓

Collect evidence

↓

Build hypotheses

↓

Verify hypotheses

↓

Identify root cause

↓

Apply fix

↓

Validate

↓

Document RCA
```

---

### 6. How would you troubleshoot a Service that isn't reachable?

A systematic approach:

1. Verify Pods are running.
2. Check Pod readiness.
3. Verify Service selector.
4. Check EndpointSlice.
5. Test DNS resolution.
6. Verify NetworkPolicy.
7. Check application logs.

---

### 7. How do you identify the root cause?

The root cause should:

- Explain every symptom.
- Be supported by evidence.
- Be verifiable.
- Resolve the issue when fixed.

---

## Advanced

### 8. Why do experienced SREs solve incidents faster?

Not because they know more commands.

Because they:

- Follow a structured process.
- Avoid assumptions.
- Collect evidence first.
- Eliminate possibilities systematically.
- Verify before changing anything.

---

### 9. How do you know an incident is actually resolved?

Not when the error disappears.

An incident is resolved when:

- Users can access the application.
- Monitoring returns to normal.
- Alerts stop firing.
- The fix is verified.
- The root cause is documented.

---

# Debugging Cheat Sheet

## Universal Workflow

```text
Incident

↓

Understand Symptom

↓

Identify Failure Domain

↓

Collect Evidence

↓

Build Hypotheses

↓

Verify

↓

Root Cause

↓

Fix

↓

Validate

↓

Document RCA
```

---

## Kubernetes Investigation Flow

```text
User

↓

DNS

↓

Ingress

↓

Service

↓

EndpointSlice

↓

Pod

↓

Container

↓

Application

↓

Database / Storage
```

Always investigate from the top down.

---

## Common Symptoms

| Symptom | Likely Playbook |
|----------|-----------------|
| Pending | Pending Pods |
| CrashLoopBackOff | CrashLoopBackOff |
| ImagePullBackOff | ImagePullBackOff |
| ContainerCreating | ContainerCreating |
| OOMKilled | OOMKilled |
| Evicted | Evicted |
| Service Unreachable | Service |
| DNS Failure | DNS |
| PVC Pending | Storage |
| Node NotReady | Node |

---

## Evidence Collection Order

```text
Status

↓

Events

↓

Describe

↓

Logs

↓

Metrics

↓

Configuration

↓

Network Tests
```

---

## Golden Rules

✅ Never panic.

✅ Never guess.

✅ Never restart before collecting evidence.

✅ Never fix a symptom.

✅ Always verify the root cause.

✅ Validate every fix.

✅ Document every incident.

---

# Final Thoughts

Kubernetes troubleshooting is not about memorizing hundreds of commands.

It is about following a disciplined process.

Every incident—whether it involves Pods, Networking, Storage, or the Control Plane—can be broken down into the same sequence:

1. Observe the problem.
2. Isolate the failing component.
3. Collect evidence.
4. Build hypotheses.
5. Verify the cause.
6. Apply the correct fix.
7. Validate recovery.
8. Prevent recurrence.

If you consistently follow this framework, you'll solve incidents faster, make fewer unnecessary changes, and build confidence in debugging complex distributed systems.

---

# Next Step

You are now ready to start the troubleshooting playbooks.

Begin with the incident that matches your symptom:

- CrashLoopBackOff
- Pending Pods
- ImagePullBackOff
- ContainerCreating
- OOMKilled
- Service Not Reachable
- DNS Issues
- PVC Pending
- Node NotReady

Each playbook follows the same framework introduced in this chapter.

