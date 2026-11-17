# learning-iptables
Learning iptables using Claude

## Intro

Here are the Claude prompts I started with:

  >I'm a software developer but would like to learn more about firewall rules using hands on exercises. Can you create a lesson plan to accomplish this?

  >i'd like to use VMs and my preferred OS is ubuntu
  
  >my host OS is windows 11.  can you provide instructions on how i setup virtual box?

## Software
- [VirtualBox](https://www.virtualbox.org/)
- [Tabby](https://tabby.sh/) - Open source cross platform terminal client (SSH, Powershell, etc)

## Sections
  - [Virtual Box Setup](virtualbox-setup.md) - Setting up Virtual Box and VMs (server, client, gateway)
  - [Lesson 1: Mental model and observation](lesson_01.md) - rule order, default policy, DROP vs. REJEC
  - [Lesson 2: Basic allow and deny rules](lesson_02.md) - rule order, default policy, DROP vs. REJECT
  - [Lesson 3: Stateful filtering](lesson_03.md) - rule order, default policy, DROP vs. REJEC 
  - Lesson 4: Modern tooling with nftable - tables, chains, sets, and persistent rulesets
  - Lesson 5: NAT and routing - the gateway VM, masquerading, and port forwarding
  - Lesson 6: Logging, debugging, and troubleshooting - log rules, packet tracing, and a debugging drill
  - Capstone: Capstone project - a three-tier lab with least-privilege rules, plus automated tests

