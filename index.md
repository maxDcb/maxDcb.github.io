---
layout: page
title: Articles
permalink: /
description: Offensive security engineering, C++ security tooling, Windows internals, AppSec, and lab-oriented security research.
---

# MaxDcb Research

Technical research notes on offensive security engineering, low-level tooling, and security architecture. I write about how security tools are built, how runtime mechanisms behave, how protocols and modules are structured, and how complex systems can be analyzed from an attacker-informed engineering perspective.

The main areas covered here are:

- Offensive security engineering and research methodology.
- C++ security tooling, build systems, module boundaries, and test harnesses.
- Windows internals, PE loading, reflective loading, unwind metadata, and runtime analysis.
- Application security, secure code review, exploitability reasoning, and design review.
- C2 architecture, transport abstractions, message formats, module loading, and operator workflows.
- Kubernetes and OpenShift security modeling, RBAC analysis, and graph-based attack surface mapping.
- LLM agentic workflows for security research, code review, tooling automation, and lab operators.

## Boundaries

The views, research, and content shared here are my own and do not represent, reflect, or speak for my employer.

All content is published for authorized security research, education, and controlled lab environments only.

<div class="quick-links">
  <a href="{{ '/projects/' | relative_url }}">Project map</a>
  <a href="{{ '/BuildingAModernC2/' | relative_url }}">Building a Modern C2</a>
  <a href="{{ '/DreamWalkers/' | relative_url }}">DreamWalkers</a>
  <a href="{{ '/OpenShiftGrapher/' | relative_url }}">OpenShiftGrapher</a>
</div>

## Latest Articles

{% include article-list.html %}
