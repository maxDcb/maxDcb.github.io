---
layout: page
title: Projects
permalink: /projects/
description: Map of the main public projects documented on the blog.
---

# Projects

This page maps the public projects behind the articles and shows how they connect.

<div class="project-grid">
  <section class="project-card">
    <h2><a href="https://github.com/maxDcb/C2TeamServer">C2TeamServer</a></h2>
    <p>TeamServer and Python client for the Exploration C2 framework. This is the central repository for releases, packaging, operator workflows, and the GUI.</p>
  </section>

  <section class="project-card">
    <h2><a href="https://github.com/maxDcb/C2Core">C2Core</a></h2>
    <p>Shared C++ contracts, message structures, listener abstractions, and module helpers used across the C2 stack.</p>
  </section>

  <section class="project-card">
    <h2><a href="https://github.com/maxDcb/C2Implant">C2Implant</a></h2>
    <p>Windows C++ beacon implementation, modules, and transport implementations for Exploration C2.</p>
  </section>

  <section class="project-card">
    <h2><a href="https://github.com/maxDcb/C2LinuxImplant">C2LinuxImplant</a></h2>
    <p>Linux C++ beacon implementation and Linux module packaging for the same framework.</p>
  </section>

  <section class="project-card">
    <h2><a href="https://github.com/maxDcb/DreamWalkers">DreamWalkers</a></h2>
    <p>Reflective shellcode loader with stack unwinding metadata registration, stack spoofing research, and CLR payload support.</p>
  </section>

  <section class="project-card">
    <h2><a href="https://github.com/maxDcb/OpenShiftGrapher">OpenShiftGrapher</a></h2>
    <p>OpenShift and Kubernetes enumeration mapped into Neo4j to make security relationships and risky paths easier to inspect.</p>
  </section>
</div>

## Related Libraries

- [libDnsCommunication](https://github.com/maxDcb/libDnsCommunication): DNS communication support.
- [libSocketHandler](https://github.com/maxDcb/libSocketHandler): TCP communication helpers.
- [libPipeHandler](https://github.com/maxDcb/libPipeHandler): SMB and named pipe communication helpers.
- [libSocks5](https://github.com/maxDcb/libSocks5): SOCKS5 support used for pivoting workflows.
- [PeDropper](https://github.com/maxDcb/PeDropper), [PeInjectorSyscall](https://github.com/maxDcb/PeInjectorSyscall), and [EarlyCascade](https://github.com/maxDcb/EarlyCascade): loader and injection experiments around Windows payload delivery.
