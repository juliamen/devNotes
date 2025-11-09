# ⚙️ Salesforce Lightning Web Components (LWC) Setup & Customization Guide

## Overview
Lightning Web Components (LWC) is a modern framework built on web standards for developing fast, reusable UI components in Salesforce.  
This guide covers environment setup, creating your first LWC, deploying it, and customizing it with JavaScript and Apex.

---

## 1. Prerequisites

Before starting, ensure you have the following tools installed:

| Tool | Version | Purpose | Link |
|------|----------|----------|------|
| **Salesforce CLI (SFDX)** | Latest | Command-line interface for Salesforce development | [Download CLI](https://developer.salesforce.com/tools/sfdxcli) |
| **VS Code** | Latest | IDE for Salesforce development | [Download VS Code](https://code.visualstudio.com/) |
| **Salesforce Extension Pack** | Latest | VS Code extensions for Apex, LWC, and metadata deployment | [VS Code Marketplace](https://marketplace.visualstudio.com/items?itemName=salesforce.salesforcedx-vscode) |
| **Git** | Optional | Version control | [Download Git](https://git-scm.com/) |

**Salesforce Org Requirements**
- Developer Edition or Sandbox org.
- **My Domain** enabled.
- Lightning Web Components enabled (default in all modern orgs).

---

## 2. Environment Setup

### Step 1 — Authenticate to Salesforce
```bash
sfdx auth:web:login -a MyDevHub
