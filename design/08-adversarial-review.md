# Adversarial Review Agents

Adversarial review agents serve as an automated security review layer for high-blast-radius operations. They are conceptually similar to separation of duties in traditional security: the agent performing the work should not be the same one approving elevated access.

## Design Principles

- **Context isolation** — adversarial agents receive the task description and the specific permission request, but NOT the full working context of the requesting agent. This prevents a shared context poisoning attack from compromising both the worker and the reviewer.
- **Independence** — multiple adversarial agents evaluate the request independently, without seeing each other's assessments.
- **Conservatism** — adversarial agents are tuned to be skeptical. Their job is to ask "does this request make sense given the stated task?" and flag anything that seems out of scope.
- **Statistical safety** — the requesting agent may have been led astray by poisoned context or accumulated errors. Because adversarial agents evaluate the request with isolated context and independent reasoning, it is unlikely that they would be fooled by the same failure mode. If any adversarial agent flags a concern, the request escalates to a human rather than proceeding unchecked.

## What Adversarial Agents Evaluate

- Does the requested resource and permission level align with the stated task?
- Is the scope of the request as narrow as it could be?
- Is there a less risky way to accomplish what the agent is trying to do?
- Does the request represent an escalation beyond what the task manifest anticipated?

## Cost Considerations

Each adversarial review is a few API calls. For high-blast-radius operations (production deployments, IAM changes, infrastructure modifications), this cost is trivial relative to the risk being mitigated. The tiered model ensures adversarial review only fires on operations that warrant it.

---

## Open Questions

### Adversarial Agent Protocol

What exactly do adversarial review agents receive, return, and how is consensus evaluated?

To define before implementing Tier 3:
- **Input**: the task description, the specific credential request (provider, resource, permission, reason), and the task manifest. NOT the requesting agent's full conversation context.
- **Output**: an approve/deny decision with a written justification.
- **Consensus**: how many adversarial agents are consulted? Is it configurable? What constitutes agreement vs. disagreement?
- **Tuning**: how are adversarial agents prompted to be appropriately skeptical without being obstructive?
- **Failure**: what happens if an adversarial agent times out or errors? Is it treated as a denial, or is a replacement consulted?
