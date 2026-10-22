# Documentation Vault

This repository stores the tutorials I write.

---

## At a glance

| Document | Topic | Audience | Length |
| -------- | ----- | -------- | ------ |
| [Apache CloudStack Installation Guide](cloudstack_installation.md) | Building a private cloud on one Ubuntu VM (KVM) | Junior sysadmin / cloud learner | 8 sections, about 30 steps |
| [Connecting Antigravity to a Google Drive MCP Server](despro_mcp/) | Letting an AI coding assistant read and write files in a shared Google Drive folder | Project team members (non-expert) | 8 steps |

---

## 1. Apache CloudStack Installation Guide

**File:** [`cloudstack_installation.md`](cloudstack_installation.md)

**What it is.** A complete walkthrough for installing Apache CloudStack 4.22, an open-source platform for running your own cloud (similar in idea to AWS), on a single Ubuntu virtual machine. The one VM plays three roles at once: management server, compute host (KVM), and storage server (NFS).

**What the reader ends up with.** A working CloudStack environment that is ready to create a zone and register a virtual machine image.

**What it covers.**

| Section | Plain-language summary |
| ------- | ---------------------- |
| Lab environment | Why the VM uses NAT networking instead of bridged, with a table of every IP address used |
| Management server and SSH | Secure remote login with SSH keys instead of passwords |
| CloudStack management | Installing the main program and its database (MySQL) and tuning the database for CloudStack |
| Nginx | Putting a reverse proxy between the user and the cloud servers |
| Compute server | Preparing the VM to host other VMs: network bridge, libvirt, AppArmor |
| Storage server | Setting up shared NFS storage and the system VM template |
| Image registration | Downloading a test image and serving it so CloudStack can import it |
| Appendix | Reference values for creating the zone, pod and host |

**Documentation skills shown.**

- Breaking a long, error-prone installation into ordered, numbered stages
- Explaining the *why* behind each step (for example, why NAT mode avoids a repeated captive-portal login), not only the *what*
- Separating command templates (`<placeholder>`) from the exact commands used in the lab
- Using tables for addresses and credentials so values are easy to find
- Adding a table of contents and an appendix so the guide works both as a tutorial and as a reference

**Technologies mentioned.** Ubuntu, KVM/libvirt, MySQL, Nginx, NFS, Netplan, OpenSSH, AppArmor, chrony.

---

## 2. Connecting Antigravity to a Google Drive MCP Server

**Folder:** [`despro_mcp/`](despro_mcp/)

**What it is.** A setup guide for connecting Antigravity (an AI-assisted code editor) to a Google Drive folder through the Model Context Protocol (MCP). Once connected, the AI agent can search, read and create files in one specific shared folder, and nothing outside it.

**Who it was written for.** Teammates on a shared project who needed to set this up on their own computers, without deep technical background.

**What is in the folder.**

| File | Purpose |
| ---- | ------- |
| [`how to connect antigravity to mcp server.docx`](despro_mcp/how%20to%20connect%20antigravity%20to%20mcp%20server.docx) | The user guide: install Node.js, add the dependencies, paste the MCP configuration, sign in with Google, restart, verify |
| [`index.js`](despro_mcp/index.js) | The server code used by the guide: a customised Google Drive MCP server that adds tools for creating text files and Google Docs, Sheets and Slides, and limits the agent to one target folder |

**Documentation skills shown.**

- Writing for non-experts: each step says exactly where to click or what to paste
- Giving an exact configuration block with clearly marked values the reader must replace (such as the folder path)
- Documenting a security boundary: the agent is limited to one folder and can only edit files it created
- Ending with a way to confirm that the setup worked

**Technologies mentioned.** Node.js, npm, Model Context Protocol, Google Drive API, OAuth 2.0.

---

## How these guides are written

The same conventions are used across the vault:

1. **Goal first.** Each guide starts with what will be built and what the reader needs.
2. **One action per step.** Steps are numbered and kept short; commands sit in code blocks.
3. **Template, then example.** A generic command with placeholders is followed by the real command used.
4. **Explain the reason.** Non-obvious steps include a short note on why they are needed.
5. **Values in tables.** IP addresses, ports and file names are collected in tables instead of buried in paragraphs.
6. **Tested order.** Steps are written in the order that worked in the lab, including fixes for steps that would otherwise fail.

---

## Repository structure

```
documentation_vault/
├── README.md                    <- this page
├── cloudstack_installation.md   <- CloudStack on KVM guide
└── despro_mcp/                  <- Antigravity + Google Drive MCP guide
```

New guides are added as separate files or folders, and listed in the table at the top of this page.

---

## Author

Adi Nugroho: [github.com/Adi-Ngrh](https://github.com/Adi-Ngrh)
