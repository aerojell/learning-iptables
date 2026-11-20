# learning-iptables
Learning iptables using Claude

## Intro

Here are the Claude prompts I started with:

  >I'm a software developer but would like to learn more about firewall rules using hands on exercises. Can you create a lesson plan to accomplish this?

  >i'd like to use VMs and my preferred OS is ubuntu
  
  >my host OS is windows 11.  can you provide instructions on how i setup virtual box?

## Claude's Hands-On Lesson Plan

Here's the [hands-on lesson plan](firewall-reference-lab.md) from Claude.  It's a living document and will be updated as I go through it. I've documented the lessons and exercises I've completed below.

## Software
- [VirtualBox](https://www.virtualbox.org/)
- [Tabby](https://tabby.sh/) - Open source cross platform terminal client (SSH, Powershell, etc)

## Sections
  - [Virtual Box Setup](virtualbox-setup.md) - Setting up VirtualBox and VMs (server, client, gateway)
  - [Lesson 1: Mental Model and Observation](lesson_01.md) - Packets, ports, the TCP handshake, and chains/hooks
  - [Lesson 2: Basic allow and deny Rules](lesson_02.md) - Rule order, default policy, DROP vs. REJECT
  - [Lesson 3: Stateful Filtering](lesson_03.md) - Connection tracking and allowing replies
  - Lesson 4: Modern Tooling with nftable - Tables, chains, sets, and persistent rulesets
  - Lesson 5: NAT and Routing - The gateway VM, masquerading, and port forwarding
  - Lesson 6: Logging, Debugging, and Troubleshooting - Log rules, packet tracing, and a debugging drill
  - Capstone: Capstone Project - A three-tier lab with least-privilege rules, plus automated tests

