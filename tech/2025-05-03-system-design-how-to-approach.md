---
title: "How to Approach a System Design Problem"
date: 2025-05-03
tags: [system-design]
categories: [Tech]
excerpt: "Guidelines to approach a system design problem/ interview"
---

# How to approach this problem, or any system design problem
We could approach a system design problem, with the following steps:
1. Understand the problem
   - i.e. what does the system do, what features it has, how it works.
2. Understand the functional requirements and non-functional requirements for this system. 
   - Functional requirements specify the specific functions or behaviors a system must perform to meet user needs. 
   - Non-functional requirements define how a system should perform, focusing on qualities attributes, like performance, security, usability, reliability, and scalability.
3. Identify and define core entities.
    - There could have a plenty primary entities we identify by this step. However, during this phase we should only focus on the most important ones that's crucially involved in the functional requirements defined earlier. In additional, we don't need to finalize all the attributes (columns) any entity should have, just identify the most important ones. Otherwise, we'll be lost in the details.
4. Design APIs or interfaces.
5. Create a high level design. This design should focus and meet all functional requirements made in Step#1.
6. Dive deeper into the design. This phase should be focused on some, if not all, non-functional requirements made in Step#1.
