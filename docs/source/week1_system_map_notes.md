# Part B — ChatGPT Conceptual System Map Notes

**Purpose:** Supporting explanation and reflection for the Part B system map. This file is stored beside the map so the diagram and its reflection are easy to find together.

## Files

- [System map PNG](week1_system_map.png) — visual map for submission
- [Editable Draw.io source](week1_system_map.drawio) — source diagram

## Components and evidence status

The diagram shows a conceptual flow from user prompt to input guardrails, foundation model, output checks and chat response. Data/context handling, optional retrieval, feedback and evaluation are shown as assumed components rather than confirmed details of the proprietary ChatGPT architecture.

- **Grey — User interface:** prompt entry and visible chat response; this is observable.
- **Green — Data/context handling:** assumed data or context flow and optional retrieval.
- **Blue — Model component:** assumed conceptual model stage that generates a response.
- **Orange — Safety / output checks:** input screening and output checking are assumed implementation details; the refusal in the observed safety test is observable.
- **Purple — Feedback / evaluation:** possible feedback and quality monitoring components; not directly tested.

The map is a reasonable engineering approximation based on observable behaviour. It does not claim to describe OpenAI's actual internal architecture.

## Potential failure points

| ID | Potential failure | Likely impact |
|---|---|---|
| F1 | Stale data or context | Outdated information may lead to a wrong answer. |
| F2 | Prompt ambiguity | The system may answer a different interpretation from the user's intended meaning. |
| F3 | Hallucination | An invented claim may misinform the user. |
| F4 | Safety check misses harmful content | An unsafe response may cause security or privacy harm. |
| F5 | Feedback is ignored | Repeated problems may remain unresolved. |
| F6 | User overtrusts task completion | A user may mistake guidance for an action that was actually completed. |

These are plausible failure scenarios for engineering analysis, not claims that all these failures occurred in the tests.

## Part B Reflection

The **model-to-output validation interaction** has the greatest concentration of related failure points in this conceptual map: a hallucinated claim (F3), a missed safety issue (F4), and a user mistaking advice for a completed external action (F6). Engineering effort should therefore concentrate on checking generated responses, applying safety checks, and clearly communicating whether the system has actually completed an action.

This connects to the Topic 1 quality dimensions of **accuracy and reliability** (reduce unsupported answers), **safety** (reduce harmful outputs), and **transparency** (make the system's capabilities and action status clear). These are plausible failure scenarios shown in a conceptual map, not incidents proven to have occurred.
