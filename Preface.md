# Introduction

So you want to run virtual machines. Good call.

Whether you're a student building your first home lab, an IT professional who's been meaning to get your hands dirty with Hyper-V for years, or a sysadmin who inherited a bunch of VMs and needs to actually understand what's going on -- this guide is for you.

## Why Hyper-V?

Microsoft built Hyper-V directly into Windows. That means if you're running Windows 10 Pro, Windows 11 Pro, or Windows Server, you already own a production-grade hypervisor. You don't need to buy extra software, and you don't need a dedicated machine. Your existing laptop or workstation can run multiple virtual machines right now.

Hyper-V is also what powers Azure under the hood. Learning it here, in your own environment, puts you one step closer to understanding how cloud infrastructure actually works.

## How This Guide Is Structured

This guide is split into two parts -- and that split is intentional.

**Part 1 (Chapters 1-6)** covers everything you need to get a Hyper-V environment running on a single machine. By the end of Chapter 6, you'll have created VMs, configured virtual networks, managed storage, and set up your first backups. This part works on a single Windows 10/11 PC or a standalone Windows Server. No enterprise infrastructure required.

**Part 2 (Chapters 7-13)** goes deeper into the enterprise side: performance tuning, high-availability clustering, security hardening, PowerShell automation, and integration with Microsoft's cloud services. Some of this requires Windows Server and multiple physical hosts. Each chapter that assumes enterprise infrastructure will say so upfront.

**Quick reference by goal:**

| Your situation | Where to start | What you can skip |
|---|---|---|
| Home lab, brand new to Hyper-V | Chapter 1 | Ch8 (clustering needs multiple servers) |
| Windows Server admin, Hyper-V beginner | Chapter 1 | Nothing -- it all applies |
| Just need the PowerShell commands | Chapter 10 | Reference Ch3/4 for context |
| Building an HA cluster | Start at Ch8 | You may already know Ch1-4 |
| Need to understand backup options | Chapter 6 | - |

## What You'll Build

By the end of Part 1, your setup will look something like this:

- A Hyper-V host (your Windows PC or server)
- One or more virtual machines running inside it
- At least two virtual switches: one connected to your physical network, one isolated for testing
- At least one VHDX disk, correctly sized and formatted
- A working backup schedule

It's a small but complete virtualisation environment -- and it's the same fundamental architecture that scales to thousands of VMs in a datacenter.

## A Note on Windows Versions

Throughout this guide, examples use Windows Server 2022 and Windows 11. Most of what's shown works identically on Windows Server 2019 and Windows 10. Where a feature requires a specific version or edition, that's noted clearly.

Let's get started.
