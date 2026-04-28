---
layout: post
title: "My vision for an interdisciplinary RSE institute: how to actually help scientists"
author: Matthias Kretz
tags: [RSE, research software, software architecture, standardization]
---

Today I attended a talk by Dr. Florian Mannseicher on [FutuRSI](https://www.futursi.de/)—an "initiative […] to 
conceptualise a German research software institution". The discussion around critical challenges and concrete needs 
facing the research software community resonated with me—particularly because I discussed many of these same 
issues—that are still unresolved—in 2020 in ["Software Engineering as Scientific Infrastructure"]({% post_url 
2020-05-11-software-engineering-as-scientific-infrastructure %}). I believe FutuRSI should be the place to solve what I 
wrote about back then. I'll elaborate in this post. Since 2020 our ability to talk about these issues has improved. 
"Research Software Engineering" (RSE) has become a widely used term. There are conferences on RSE. But the main problem 
has not been addressed—in my opinion.

## The Real Problem: Complexity Accumulation

Research software across all disciplines faces the same fundamental challenges:

1. **Missing software engineering expertise** means that software architecture and design suffer from the start. Domain 
   experts know their science but typically lack training in scalable software design.

2. **Senior software engineers are not easily accessible** for sprint participation or design sessions. Even when they 
   exist within universities or research centers, they're rarely integrated into cross-project work.

3. **Shared problems remain undiscovered** across departments, institutes, and scientific fields. This leads to massive 
   duplication of effort—and ultimately more software complexity to maintain across the ecosystem.

4. **Languages and tools are treated as "god given."** Research software communities often accept their toolchain as 
   immutable environment constraints rather than actively shaping them. A single research group cannot reasonably 
   influence language standards, compiler development, or library ecosystems. But an institution representing the 
   science community as a whole can.

The overarching goal should be clear: **reduce complexity in research software by moving problems upward in the software 
stack.** This means solid software architecture, shared solutions, and reliable standards—not more fragmented projects 
solving the same problems independently.

## What LLMs and Coding Agents Don't Solve

There's considerable excitement about AI-assisted development and coding agents. While these tools have their place, 
they don't address the core problems listed above. Large language models can generate code snippets or even complete 
and complex implementations of non-trivial problems, but they cannot replace the architectural oversight of a senior 
software engineer.

A coding agent doesn't understand the long-term maintenance implications of a design decision. It doesn't recognize when 
three different research groups are independently solving the same problem in incompatible ways. It cannot participate 
in C++ standardization meetings to advocate for features that would benefit scientific computing. Most importantly, it 
cannot provide the institutional memory and cross-project perspective that only comes from experienced human engineers 
working collaboratively over years.

The need for human expertise in software architecture and design is arguably *increasing* in the age of AI-generated 
code, because we now need even stronger oversight to ensure that generated code integrates well with existing systems 
and follows sound architectural principles.

## Moving Problems Upward: A Concrete Vision

A central RSE institute should focus on services that consolidate effort and reduce total complexity:

### Architecture Review and Design Consulting

Research software projects should be able to engage experienced software architects early in their development 
lifecycle. This isn't about imposing rigid methodologies—it's about preventing architectural mistakes that become 
expensive technical debt. Face-to-face (or video conference) discussions about code structure, dependency management, 
and API design can save years of maintenance pain.

This consulting function needs permanent positions of highly qualified software engineering experts who can serve as 
trusted advisors. These aren't temporary contractors; they're infrastructure staff, comparable to the engineers 
maintaining a particle accelerator.

### Senior Engineer Integration

An institute should maintain a pool of senior software engineers who can integrate into research projects for focused 
periods—whether for a sprint, a design session, or a code review cycle. This requires both the organizational structure 
to make such engagement possible and the recognition that this work is valuable, not a distraction from "real" research.

### Cross-Project Consolidation

A central institute has the unique position of seeing patterns across multiple research domains. When three different 
physics groups independently implement numerical linear algebra routines, or when five chemistry projects each maintain 
their own I/O libraries, someone needs to identify these overlaps and facilitate consolidation.

This consolidation work extends beyond internal projects. Consider C++26's addition of linear algebra to the standard 
library: the standard library maintainers lack both the expertise and funding to deliver an actual implementation. An 
RSE institute could fill this gap and facilitate or lead the contribution of an implementation that matches the 
requirements of the scientific users.

This isn't just about reducing duplicate code—it's about reducing the total surface area of software that needs 
maintenance, testing, and user support across the German research landscape.

### Active Standardization Participation

Research software communities depend on languages, compilers, and libraries that evolve without their input. The C++ 
standard and GCC now include `std::simd`, which will benefit countless scientific applications—but this only happened 
because I got in the lucky position where I could invest the time to contribute to WG21 (ISO C++ committee) and GCC.

A single research group cannot meaningfully participate in ISO standardization, ECMA specifications, compiler 
development, or major general-purpose library development. But an institution representing the broader science community 
can. This means having dedicated personnel who:

- Track relevant standardization efforts
- Contribute use cases and requirements from scientific computing
- Implement proposed features in compilers and libraries
- Provide feedback loops between standards bodies and end users

This is where my own work on `std::simd` demonstrates the model: by investing in C++ standardization, I've made 
vectorization accessible to scientists without requiring them to learn intrinsics or deal with vendor-specific 
extensions. The same approach can be applied to other areas where research software has unmet needs.

### Long-Term Maintenance as Infrastructure

Maintaining shared software components is work—sometimes a lot of work. Programming languages evolve, hardware changes, 
new libraries become available, users report bugs, and feature requests accumulate. Failure to act on user feedback 
means users abandon the shared solution, defeating its purpose.

An RSE institute must treat maintenance as a permanent infrastructure commitment, not an afterthought. This requires 
stable funding for permanent positions that back the software projects the wider science community relies on.

## Where This Leaves Us

The FutuRSI initiative and similar efforts are asking the right questions. The conversation has matured since then, and 
while late (from my perspective), action now is better than waiting. My 2020 document outlined many of the above 
thoughts, when the conversation around RSE was less mature. Today, the community recognition is growing.

What I envision isn't just another coordination body or networking platform. It's a service organization that does 
the hard work of consolidating effort, preventing or reducing duplication, and moving complexity upward where it can be 
managed professionally. This means making difficult decisions about which projects to consolidate, which standards to 
champion, and which architectural patterns to promote.

The alternative—continuing with fragmented, under-supported research software projects that duplicate effort and 
accumulate technical debt—is unsustainable. We either build the infrastructure to support our software properly, or we 
accept that much of it will remain fragile, short-lived, and inaccessible to the broader community.

I'm publishing this vision to contribute to the ongoing discussion. The details matter, but the direction is clear: 
research software and its foundations deserve the same level of financial and professional support that is already 
established for (critical) large-scale research infrastructure.
