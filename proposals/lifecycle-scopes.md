# Proposal: Lifecycle-Aware Behavioral Scopes for Kubescape

## 1. Motivation & Background
Currently, runtime security profiling and enforcement often rely on a single, static set of allowed behaviors for the entire lifetime of a workload. This creates a problem: if the policy accommodates all phases (initialization, runtime, and teardown), it becomes overly permissive. That happens because startup often needs much more capabilities than runtime.

Furthermore, when an application is large and complex, a single, global Bill of Behavior (BoB) becomes exceptionally broad. Because a complex application inherently requires a diverse set of capabilities over its lifetime, a flat policy ends up allowing nearly everything, significantly reducing its security value. 

The more complicated an application is, the more logical it becomes to divide its behavior into distinct scopes. Instead of merely asking, "Can this action happen?", a robust security model must ask, "When and where is this action allowed to happen?" To achieve a strict least-privilege BoB without leaving unnecessary security gaps or generating alert fatigue, the runtime contract needs to be context-aware.

## 2. The Core Idea: Lifecycle Scopes
This proposal introduces **Lifecycle Scopes** into Kubescape. This approach aligns behavioral profiles with the natural phases of an application's lifecycle. 

While tying these scopes to existing Kubernetes primitives—such as readiness probes and termination signals—is the most robust approach, transitions can also be driven directly by the workload. For example, an application could indicate cycle transitions by creating or modifying specific files (e.g., touching a `/tmp/.app-ready` file). This allows the runtime engine to monitor these indicators and automatically "rotate" the active security contract. But that also creates a potential confused deputy problem.

### Proposed Core Phases
1. **Startup Scope:** 
   A broader set of permissions required to bootstrap the application. This includes reading configuration files, establishing database connection pools, executing JIT compilation, and setting up the runtime environment. 
   *Trigger:* Container start.
   *Transition:* Ends once the workload passes its first Kubernetes readiness/health check, or when the application signals readiness (e.g., via a designated file creation).

2. **Runtime Scope:** 
   A highly restrictive, "steady-state" profile. This is the narrow set of permissions needed to process standard requests or events. Since the application spends the vast majority of its life in this state, minimizing the allowed syscalls and behaviors here significantly reduces the attack surface.
   *Trigger:* Successful readiness probe, or application-driven readiness signal.

3. **Shutdown Scope:** 
   Permissions specifically required for graceful termination, such as flushing logs, draining connections, unregistering from service meshes, or writing final state to disk. 
   *Solving Shutdown Anomalies:* For workloads with protracted, complex teardown sequences, normal steady-state anomaly detection often triggers false positives. By transitioning to a dedicated Shutdown Scope, it is possible to cleanly separate these expected "end-of-life" behaviors from actual runtime anomalies, eliminating the false positives that occur during long shutdown phases.
   *Trigger:* Receiving a `SIGTERM`, executing a pre-stop hook, or an application-generated file signal indicating teardown has begun.

### Custom and Extended Scopes
It is worth noting that lifecycles do not have to be strictly limited to these three core phases. Depending on the architecture, other distinct life stages are entirely possible. For example, a heavy database workload might have a dedicated "Maintenance" or "Backup" phase triggered by a scheduled cron job dropping a specific state file. Scopes can be designed to match whatever distinct operational phases the application goes through.

## 3. Future Granular Scopes
Lifecycle Scopes represent an initial step toward context-aware profiles. In the future, as eBPF and runtime integrations mature, this concept could be extended to support more granular scopes, such as:
- **Thread/Process Scopes:** Enforcing security boundaries at the individual OS thread level, allowing specific threads to perform actions that are denied to the rest of the application.
- **Stacktrace/Module Scopes:** Using the calling context to validate behaviors (e.g., "Allow this outbound network call *only* if it originated from the official cloud provider SDK").

While these granular scopes currently introduce complex architectural challenges and potential performance overheads, Lifecycle Scopes represent the practical first step toward building a context-aware behavioral engine in Kubescape.
