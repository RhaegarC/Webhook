# Architecture Design: Queue & Worker Model for AI Dev Automation

## Overview
This document outlines the architecture for a reliable, self-hosted AI development workflow (utilizing Claude Code). It uses a Queue & Worker model to ensure reliable webhook processing, prevent race conditions, and handle concurrent requests gracefully.

## Core Components

1. **Event Source (External)**
   * GitHub/GitLab webhooks (e.g., PR creation, issue comments).
   * Initiates the workflow by sending an HTTP POST request.

2. **Cloudflare Tunnel**
   * Secure ingress point.
   * Routes external webhook traffic to the local API Container without opening inbound firewall ports.
   * Should be protected by Cloudflare Access (Zero Trust) for additional security.

3. **API Container (Producer)**
   * **Role:** Acts as the entry point and job dispatcher.
   * **Responsibilities:**
     * Validates incoming webhook signatures (e.g., HMAC).
     * Parses payload and creates a job.
     * Pushes the job to the Redis Queue.
     * Immediately returns an HTTP `202 Accepted` response to the webhook source to prevent timeouts.
   * **Security:** Does *not* execute code or run AI agents.

4. **Message Queue (Redis)**
   * **Role:** Buffers incoming tasks.
   * **Benefit:** Decouples the API from the worker. If 10 webhooks arrive at once, they are safely queued instead of overwhelming the system or failing.

5. **Dev Container (Worker/Consumer)**
   * **Role:** The execution environment for the AI agent.
   * **Responsibilities:**
     * Listens for jobs in the Redis Queue.
     * Pulls a job when available.
     * Clones the target repository into an **internal Docker volume** (isolated from the host).
     * Executes Claude Code to perform the requested task.
     * Commits and pushes changes directly to the remote Git provider (GitHub/GitLab).
     * Notifies the API container or queue upon completion.
   * **Resource Management:** Must have strict CPU/RAM limits applied via Docker to prevent host server starvation.

6. **Remote Git Repository (External)**
   * The source of truth for the code.
   * The Dev Container pulls from and pushes to this remote directly, avoiding conflicts with local host Git operations.

## Workflow Sequence

1. **Trigger:** An event occurs on the Git provider, sending a webhook.
2. **Ingress:** Cloudflare Tunnel securely routes the request to the API Container.
3. **Validation & Queuing:** API Container validates the request, pushes a job to Redis, and returns `202 Accepted`.
4. **Execution:** Dev Container picks up the job from Redis.
5. **Isolation:** Dev Container clones the repo into its internal volume.
6. **AI Action:** Claude Code runs the required task (e.g., code review, bug fix).
7. **Delivery:** Dev Container commits and pushes the changes to the Remote Git Repository.
8. **Completion:** The job is marked as complete, and the Dev Container awaits the next task in the queue.

## Key Advantages of this Design

* **Reliability:** Webhooks are acknowledged immediately. Long-running AI tasks won't cause webhook timeouts.
* **Concurrency Handling:** Multiple events can be queued and processed sequentially by the worker without race conditions or `.git/index.lock` conflicts.
* **Security:** Code execution is isolated within the Dev Container. The host OS is protected, and ingress is secured via Cloudflare.
* **State Management:** Isolating the active repository in a Docker volume prevents conflicts with the user's local host repository.

## Implementation Checklist

- [ ] Set up Cloudflare Tunnel and Access policies.
- [ ] Configure Redis container for message queuing.
- [ ] Develop API Container with webhook validation and Redis producer logic.
- [ ] Configure Dev Container with Claude Code, Redis consumer logic, and Git credentials.
- [ ] Set strict Docker resource limits (CPU/Memory) on the Dev Container.
- [ ] Ensure Dev Container uses a dedicated internal Docker volume for cloning repos.