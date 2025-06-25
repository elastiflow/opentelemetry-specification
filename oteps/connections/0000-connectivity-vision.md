# Propose OpenTelemetry connectivity vision

A new OpenTelemetry signal, "connection", to natively represent network and application communication flows.

## Motivation

A critical category of telemetry data—describing network connections and communication sessions—lacks a dedicated, semantically appropriate representation within the current OpenTelemetry specification. This proposal introduces a new "connection" signal to address this gap, driven by several key factors:

### The semantic gap for network-oriented data

A fundamental challenge in representing network telemetry stems from a common misconception: that a "connection" is always a single, complete, and bidirectional entity observed identically from all points. Traditional network telemetry, such as NetFlow, IPFIX, sFlow, and firewall logs, describes source-to-destination communication over a period, often including multiple timestamps (e.g., start, end) and stateful information. However, the perspective of a "connection" varies significantly depending on the observation point:

* **Endpoint Perspective**: Applications running on hosts (endpoints) typically initiate and terminate a flow, often establishing a clear client-server relationship. From this perspective, a connection is often perceived as a single, bidirectional entity.
* **Network Device Perspective**: Routers, switches, and firewalls, on the other hand, primarily forward or inspect traffic. They commonly observe and report unidirectional flow records for each direction of traffic they see. For example, a router might send one flow record for traffic from A to B and a separate record for traffic from B to A, even if they are part of the same logical application connection. This is particularly true in networks with asymmetric routing, where the forward and reverse paths for a single logical connection may not traverse the same network devices. Some arguments suggest that the assumption of inherent biflows (bidirectional flows) can be problematic for accurate network analysis.

Without a concrete, standardized data model that accounts for these nuances, it is left to individual implementers to decide how best to represent connections, leading to inconsistencies and making it difficult to establish standard methods for identifying application endpoints versus network forwarding devices (routers/switches), and calculating network-related telemetry data like round-trip time. Attempting to represent such data using existing OpenTelemetry signals presents significant challenges:

* **Metrics**: Metrics are suitable for aggregated numerical data but are ill-suited for capturing the rich, often non-numeric, attributes of individual connections (e.g., TCP flags, firewall rule IDs, application identifiers). Furthermore, using high-cardinality attributes like IP addresses and port numbers as metric labels can overwhelm metrics backends. While metrics can be derived from connection data , they do not replace the need for the raw or semi-raw connection details.
* **Logs**: While logs are commonly used as a workaround, they are fundamentally designed for events occurring at a single point in time. Conceptually, forcing traditional network-traffic telemetry data into logs, feels like representing traces as logs — properly fit and a mistake. A flow record or connection, by contrast, has a duration and can experience multiple state changes or events throughout its lifecycle. Forcing durational data into a single-timestamp log record obscures its temporal nature and makes analysis difficult. Furthermore, when network devices report unidirectional flow records, trying to fit this into a log signal that might implicitly suggest a complete, bidirectional "event" can lead to misinterpretation.

### Hierarchical Relationship and Contextualization

Observability data often exhibits a hierarchical relationship. A log can be contextualized by a `traceId`, indicating the specific request or operation it relates to. Similarly, multiple traces can occur over a single, persistent network connection (e.g., HTTP keep-alive, a database connection, a gRPC stream). This implies a hierarchical relationship where a connection can be a parent context for multiple traces, which in turn can provide context for logs. Representing a connection as a log inverts this natural hierarchy, as it would imply a log (the "connection") containing multiple traces, rather than traces being contextualized by a `connectionId`. A native connection signal allows for the correct modeling of this hierarchy: Connection → Traces → Logs.

### Bridging observability silos (DevOps, NetOps, SecOps)

OpenTelemetry has predominantly focused on Application Performance Monitoring (APM) and DevOps use cases. However, comprehensive observability requires breaking down data silos between different operational domains, including NetOps and SecOps. Network connection data is a fundamental telemetry type often shared and utilized by all these teams. A connection signal can serve as a common data pivot point, enabling NetOps to understand network behavior, SecOps to analyze security posture and incidents through network traffic, and DevOps to correlate application performance with underlying network conditions.

### Analysis trace through a network-centric lens enhances operational value 

Operators need to understand the connectivity of their applications to effectively troubleshoot issues. For example, when diagnosing a slow microservice, traces and metrics might pinpoint a bottleneck within an application component. However, without detailed connection telemetry, it's difficult to determine if the root cause is a code issue or a network misconfiguration (e.g., high latency, packet loss, or misconfigured firewall rule between services). By correlating traces with a connection signal, operators can more precisely determine the root cause of performance issues that span application and network layers. This capability is crucial for modern distributed systems where the network is an integral part of the application stack.

### Addressing Unmet Community Needs:

The OpenTelemetry community has repeatedly expressed the need for better handling of network telemetry like NetFlow and IPFIX. Several GitHub issues highlight these ongoing requirements:

1. Requests for a [NetFlow receiver within OTel collectors](https://github.com/open-telemetry/opentelemetry-collector-contrib/issues/32732). The existing netflowreceiver converts flow data to logs, which, as discussed, is a workaround.
2. Proposals to [enrich traces with IPFIX data](https://github.com/open-telemetry/opentelemetry-collector-contrib/issues/28692), indicating a need for correlation that a native signal could simplify.
3. Discussions on deriving [metrics from flows](https://github.com/open-telemetry/opentelemetry-specification/issues/3138). The value of deriving metrics from flows is limiting, as it does not enable the full utility of flows as a data set within observability solutions which comes from raw or semi-raw flow records.
4. Requests for [generic network data receivers](https://github.com/open-telemetry/opentelemetry-collector-contrib/issues/29759). These issues demonstrate a persistent demand for a more integrated and semantically appropriate way to handle network connection data within OpenTelemetry.

## Naming Rationale: "Connection" vs. "Flow"

We've deliberately chosen the term "connection" over "flow" to prevent confusion with existing network technologies like NetFlow and to encompass a much broader spectrum of connection-oriented telemetry.

While NetFlow records are a primary candidate for representation as connection signals, this new signal also aims to cover other vital data types. Think of firewall session logs, application-level persistent connection telemetry (like gRPC streams or message queue connections, if instrumented), and even DNS interactions.

A key distinction is that a connection is inherently temporal and durational. This means that many individual "flows" (in the NetFlow sense of a unidirectional segment of traffic) could occur throughout the lifecycle of a single, overarching connection.

Although a precise connection data model is still in development, its semantic conventions will be crucial. These conventions will define the various "shapes" that data from different sources can take—whether it's a network flow, a firewall log record, or an application-level interaction. They'll also introduce specific connection.type values (e.g., network.flow, firewall.session, dns.interaction) to differentiate these sources while ensuring they share a common structural representation. This approach will also demonstrate how multiple distinct flows can relate to a single connection, potentially modeled in a way similar to how traces relate to spans in distributed tracing.


# EVERYTHING BELOW IS TEMPLATED TO BE FILLED OUT

## Explanation

Explain the proposed change as though it was already implemented and you were explaining it to a user. Depending on which layer the proposal addresses, the "user" may vary, or there may even be multiple.

We encourage you to use examples, diagrams, or whatever else makes the most sense!

### **What is a connection**
Provide a clear and concise definition of what a "connection" is in the context of this proposal. This section should define what constitutes a connection event and the key attributes that describe it (e.g., source, destination, duration, data transferred, protocol).

### **How "connection" aligns with the OpenTelemetry vision**

Explain how the "connection" signal supports the core mission of OpenTelemetry. Describe how it will enable developers to innovate faster and maintain high reliability by providing deeper insights into the interactions between components in a distributed system.

### **Making a well-rounded observability suite by adding "connection"**
Discuss how the "connection" signal complements the existing signals of traces, metrics, and logs. Explain the unique questions about a system’s behavior that can be answered with connection data, such as identifying unexpected cross-service communication or diagnosing network-level failures.

### **Current state of connection monitoring**
Briefly describe the existing landscape of connection and network monitoring tools and methodologies. Highlight the fragmentation in data formats, collection methods, and the lack of standardization, which OpenTelemetry aims to solve.

### **Making "connection" compatible with other signals**
Detail the plan for correlating the "connection" signal with traces, logs, and metrics. Explain how connections can be linked to specific requests (trace context) and to the resources that produce them (resource context), enhancing the actionability of all telemetry data.

### **Standardize "connection" data model for industry-wide sharing and reuse**
Outline the goals for a new "connection" data model. This model should be efficient, capable of representing various types of network connections, and allow for unambiguous mapping from existing formats (e.g., network flow logs, eBPF data).

### **Performance considerations**
Address the potential performance impact of collecting connection data. Emphasize that the proposed standard must be implementable in a low-overhead manner suitable for high-performance, always-on production environments.

### **Promoting cloud-native best practices with "connection"**
Explain how this signal will provide first-class support for modern, dynamic environments like Kubernetes and serverless architectures. Discuss how connection data can improve the resiliency, manageability, and observability of cloud-native applications.

### "Connection" use cases
List and describe specific, practical use cases for the "connection" signal. Examples could include generating a real-time service map, detecting network policy violations, analyzing latency between services, or identifying inefficient data transfer patterns.

## Internal details

From a technical perspective, how do you propose accomplishing the proposal? In particular, please explain:

* How the change would impact and interact with existing functionality
* Likely error modes (and how to handle them)
* Corner cases (and how to handle them)

While you do not need to prescribe a particular implementation - indeed, OTEPs should be about **behaviour**, not implementation! - it may be useful to provide at least one suggestion as to how the proposal *could* be implemented. This helps reassure reviewers that implementation is at least possible, and often helps them inspire them to think more deeply about trade-offs, alternatives, etc.\

*From a technical perspective, propose how the "connection" signal could be implemented. Discuss its impact on existing functionality, potential error modes, and corner cases, while emphasizing that this OTEP focuses on behavior, not a specific implementation.*

## Trade-offs and mitigations

What are some (known!) drawbacks? What are some ways that they might be mitigated?

Note that mitigations do not need to be complete *solutions*, and that they do not need to be accomplished directly through your proposal. A suggested mitigation may even warrant its own OTEP!

*Identify any known drawbacks or challenges associated with introducing a "connection" signal (e.g., data volume, security concerns). Suggest potential mitigations for these trade-offs.*

## Prior art and alternatives

What are some prior and/or alternative approaches? For instance, is there a corresponding feature in OpenTracing or OpenCensus? What are some ideas that you have rejected?

*Discuss existing and alternative approaches to connection monitoring. Mention any corresponding features in OpenTracing or OpenCensus, as well as any ideas that were considered and rejected during the initial proposal phase.*

## Open questions

What are some questions that you know aren't resolved yet by the OTEP? These may be questions that could be answered through further discussion, implementation experiments, or anything else that the future may bring.

*List any unresolved questions that will require further discussion or experimentation. This demonstrates foresight and invites community collaboration to find the best solutions.*

## Prototypes

Link to any prototypes or proof-of-concept implementations that you have created.
This may include code, design documents, or anything else that demonstrates the
feasibility of your proposal.

Depending on the scope of the change, prototyping in multiple programming
languages might be required.

## Future possibilities

What are some future changes that this proposal would enable?

*Describe the future opportunities that the "connection" signal could enable. This might include automated network policy generation, advanced security threat detection, or more sophisticated performance analysis.*
