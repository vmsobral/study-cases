# Designing Reliable Real-Time AI Voice Agents in Regulated Environments

## 1. Problem Context
In a financial services environment, we explored the use of AI-powered voice agents to automate debt collection workflows. The goal was to reduce operational workload while maintaining compliance, correctness, and a natural conversational experience.

Unlike chatbot interfaces, voice-based interactions introduce strict latency constraints. Long pauses or inconsistent responses directly impact user trust and call effectiveness.

Additionally, this system operated in a regulated financial context, requiring:
- Controlled messaging behavior
- Compliance adherence
- Clear conversational boundaries
- Reliable logging and traceability

This was not a research experiment — it needed to operate in production under real business constraints.

It was particularly interesting because we needed to prove that AI could replace human work that, although mechanical and with strict rules at some point, still required adaptation, empathy and compliance adequacy. All of this while keeping latency low, as this was a major concern for stakeholders, and in a challenging, under developed channel that was voice calls.

## 2. Initial Architecture
The initial implementation followed a relatively standard pipeline:

`Speech-to-Text (STT) → LLM → Text-to-Speech (TTS)`

The voice layer was handled via LiveKit, while the LLM layer processed user intent and generated responses. Early iterations relied heavily on prompt design to guide responses.

However, several issues quickly emerged:
- Hallucinations in edge cases
- Compliance risks in open-ended responses
- Context window bloat
- Increasing latency as logic complexity grew

Prompt engineering alone was insufficient to guarantee reliability.

## 3. Multi-Layer LLM Architecture
To improve reliability and safety, we introduced a layered reasoning approach:
- A primary LLM responsible for conversational reasoning
- A state-identification LLM to determine conversation step and action items completeness
- A “shield” LLM to validate compliance and response boundaries
- An orchestration layer coordinating these components

This significantly improved safety, reduced undesirable responses and prevented context injection.

However, although running in parallel, it still introduced additional latency — a critical trade-off in voice-based interactions.

This approach aligned with existing LLM architectural standards within the organization, but didn't seem fit for a use-case that needed such latency control. It also felt like “engineering around model limitations”, as LLMs were still at an earlier development stage.

I debated solutions around re-engineering the workflow, adding more iterations (like processing when the user was speaking) to reduce processing time and streaming LLM responses to the TTS to reduce silent time.

## 4. Transition to Small-Agent Architecture
As models evolved and we better understood system behavior, we redesigned the architecture around smaller, specialized agents.

Instead of one large reasoning model handling all responsibilities, we:
- Split responsibilities into specialized “small agents”
- Restricted knowledge domains per agent
- Used tool-first execution patterns
- Implemented explicit delegation between agents

This reduced reasoning surface area and improved determinism.

One of the benefits of that approach was that the external “shield” layer became unnecessary. By constraining context and limiting each agent’s scope, we reduced hallucination risk at the architectural level rather than through post-processing validation.

Also, orchestration and action items completeness check became entirely handled by tool calling and agents hand-off.

This shift significantly improved response quality and control. Code architecture became cleaner, eliminating unnecessary structures and improving maintainability.

This architectural shift marked a turning point in the development of the solution, improving reliability and reducing perceived risk. This allowed us to have the financial institution's blessing to start calling real customers at that point.

## 5. Latency vs Reliability Trade-Offs
Voice interactions introduced strict latency expectations. Every additional LLM call or tool invocation added delay.

Key contributors to latency:
- Multi-layer reasoning
- Tool-based deterministic checks
- Model inference times
- STT/TTS processing

While architectural improvements reduced hallucinations, latency remained a persistent challenge. The need for too many regulatory constraints and more determinism posed a difficult challenge.

Ultimately, latency improvements came primarily from:
- Model upgrades
- Improvements in STT and TTS systems

Not from architectural simplification alone.

This reinforced an important lesson: Some system constraints are infrastructure-dependent, not purely architectural.

## 6. Identity Verification and Sensitive Data Boundaries
A critical requirement in this system was identity verification. Before discussing any financial details, the voice agent was required to validate the user’s identity using partial sensitive information (e.g., last digits of SSN, address confirmation, or other personal data).

This introduced several architectural challenges:

- The LLM could not have unrestricted access to personal data.
- Verification logic needed to remain deterministic and auditable.
- The system needed to prevent accidental data leakage during hallucinations.
- The flow had to gracefully handle verification failures or partial matches.
- To address this, we separated identity verification from conversational reasoning.

The LLM was responsible only for guiding the interaction and collecting structured verification inputs. The actual validation was performed by deterministic backend services through tool calls.

Sensitive data never lived inside the LLM context beyond the minimum required input, and verification results were returned as structured, non-sensitive flags (e.g., verified / not verified / retry allowed). Further details, required for proceeding with the call, were just injected after verification was successful, preventing LLM from having context information earlier than necessary and mitigating risks.

This boundary ensured:

- Compliance with internal policies
- Reduced exposure risk
- Architectural containment of sensitive information
- Clear separation between probabilistic reasoning and deterministic validation

This design reinforced a key principle: AI should assist with interaction, not control access to protected data.

## 7. Observability, Evaluations and Operational Maturity
To operate reliably in production, we implemented:
- Structured logging per conversation step
- Latency tracking per pipeline component
- Token usage metrics
- Error classification and tracing
- Evaluations framework for scenario definition and testing

We treated the AI system as a distributed system with probabilistic components.

Observability proved essential for:
- Debugging edge cases
- Identifying latency bottlenecks
- Understanding model behavior drift
- Supporting post-call analysis

Evaluations were also required for:
- Scenario documentation
- Response validation (similar to TDD principles)
- Regression prevention
- Model and prompt upgrade with confidence

Without observability and proper testing/evaluations, iteration would have been guesswork. That instrumentation allowed us to analyse LLM reasoning, increase confidence in the product behavior and identify latency bottlenecks, which we handled with prompt engineering.

## 8. Workflow Orchestration with Temporal
While LLM orchestration remained application-managed, broader call workflows were managed using Temporal.

Temporal handled:
- Processing debtor lists
- Executing outbound calls
- Post-call transcript processing
- Persistence of structured results

This separation ensured:
- Idempotent processing
- Retry safety
- Controlled workflow execution


It allowed the AI layer to remain focused on reasoning while business process control remained deterministic.

## 9. Open Problems and Ongoing Challenges
Some challenges remained partially unresolved:
- Prompt versioning lacked formal governance
- Latency was partially model-dependent
- Model upgrades introduced behavioral drift
- Evaluation frameworks were still evolving

Enterprise AI systems are not static deployments — they require continuous refinement.

## 10. Reflections
Building AI systems in regulated, latency-sensitive environments requires:
- Architectural containment of model behavior
- Explicit trade-off management
- Observability-first design
- Deterministic control layers
- Clear separation between reasoning and execution

AI systems are not merely model integrations — they are distributed systems with probabilistic cores.

The most impactful improvements did not come from better prompts alone, but from better system design.

