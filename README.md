# FuseMeet: Building an Enterprise-Ready Video Collaboration Platform

> A behind-the-scenes breakdown of how I engineered FuseMeet for interview panels, virtual classrooms, and high-stakes collaboration.

## 1. Why I Built FuseMeet
- Technical interviews and remote classrooms need more than basic video calls; they require code collaboration, structured Q&A, and airtight security.
- Existing tools demanded paid add-ons, lacked control over data residency, or struggled with network-restricted offices.
- I wanted full ownership of the stack, so I built FuseMeet end-to-end: signaling, media relay, scheduling, invitations, and infrastructure automation.

## 2. TL;DR Impact
- **WebRTC + Monaco Editor** deliver real-time audio/video plus 50+ language code collaboration.
- **Redis-backed distributed sessions** keep multi-room interviews in sync even when I scale horizontally on AWS EC2.
- **AWS-native deployment** (ALB, Auto Scaling, CloudWatch, RDS, ElastiCache, SES, COTURN) keeps latency low and availability high.
- **Security first**: JWT auth, role-based access, signed WebSocket channels, rate limiting, and audit logging.

## 3. Experience Highlights
- Designed a diff/patch engine using Java Diff Utils to cut code-sync payloads by ~90%.
- Tuned Spring Boot (Java 21) for 512 servlet threads, HikariCP 200 connections, and resilient WebSocket workers.
- Automated SES onboarding, TURN provisioning, HTTPS (Let's Encrypt + Cloudflare), and backend auto-scaling with Python/Bash scripts.

## 4. Product Walkthrough
| Flow | What Happens |
| --- | --- |
| **Join Room** | JWT-authenticated user hits `/api/rooms/{roomId}/join`; backend verifies access, emits participant state, and subscribes client to STOMP topics.
| **WebRTC Signaling** | Offers/answers/ICE candidates move via secure WebSocket channels, then peers switch to direct media with COTURN fallback.
| **Code Collaboration** | Editors emit diff patches to a Spring service that applies, versions, and rebroadcasts updates via Redis pub/sub for all participants.
| **Scheduling & Invites** | Users create sessions with timezone awareness. AWS SES sends RSVP emails with deep links; responses update room state.
| **Question Bank** | Interviewers manage categorized question sets with visibility controls (team, org, private) and attach them to rooms.
| **Whiteboard & Notes** | Canvas events sync over the same real-time backbone used for code patches, keeping meeting artifacts in lockstep.

## 5. Architecture Snapshot
```
React + Vite (TS) ─────────────────────┐
                                       │ HTTPS via Cloudflare CDN
WebRTC Clients + Monaco Editor ───────▶ ALB (Layer 7)
                                       │
                               Spring Boot 3 (EC2 Auto Scaling)
                               ├─ Auth / Rooms / Questions / Scheduler
                               ├─ WebSocket + STOMP (SockJS fallback)
                               └─ Code Sync Service (diff/patch)
                                       │
                     Redis (ElastiCache) for sessions, pub/sub, cache
                     MySQL 8 (RDS Multi-AZ) for transactional data
                     AWS SES for transactional emails
                     COTURN (EC2) for STUN/TURN relay
```

## 6. Technology Stack
- **Frontend**: React 18, TypeScript 5, Vite, Tailwind, React Router, Zustand context wrappers.
- **Backend**: Java 21, Spring Boot 3.5.7, Spring Security 6, Spring Data JPA, Spring WebSocket, Resilience4j, Micrometer.
- **Real-Time**: WebRTC, STOMP over WebSocket, SockJS fallback, Redis pub/sub.
- **Data Layer**: MySQL 8 (RDS), Redis 7 (ElastiCache), Caffeine fallback cache, Hibernate 6.
- **Messaging/Email**: AWS SES, Spring Mail, optional RabbitMQ for async workflows.
- **DevOps**: Terraform-lite Bash/Python scripts for EC2/SES/COTURN, GitHub Actions-ready Maven + pnpm pipelines, CloudWatch metrics for scaling.

## 7. Security & Reliability Decisions
- JWT access/refresh tokens with HttpOnly cookies to balance SPA usability and safety.
- Role-based access control (Admin, Interviewer, Candidate, Student) enforced at REST and WebSocket layers.
- Resilience4j rate limiting (1000 req/s) and circuit breakers shield downstream systems.
- Spring Session on Redis centralizes auth state; Redis keyspace notifications clean stale sessions.
- Actuator health probes wired to ALB and custom CloudWatch alarms to trigger scaling before degradation.

## 8. AWS Infrastructure at a Glance
- **Networking**: VPC with public/private subnets, security groups locking signaling to known ports, SSM access for bastion-less maintenance.
- **Compute**: EC2 Auto Scaling Group (min 1 / max 5) for the backend, separate EC2 instance running hardened COTURN.
- **Data**: RDS MySQL Multi-AZ, ElastiCache Redis with automatic backups.
- **Edge & Delivery**: Cloudflare CDN in front of ALB for global caching + TLS termination, fallback to Let's Encrypt certs on Nginx.
- **Observability**: Micrometer -> CloudWatch dashboards (active rooms, participants, editor latency) + log shipping for audits.

## 9. Automation Toolkit
- `deploy_full_stack.py`: bootstraps VPC resources, EC2 launch templates, Auto Scaling policies.
- `setup_coturn_server.sh`: provisions TURN with dynamic credentials, firewall rules, and cert rotation.
- `setup_backend_autoscaling.py`: wires CloudWatch alarms to target CPU, memory, and custom room metrics.
- `setup_aws_ses.py`: verifies domains, generates SMTP creds, and rotates them into Secrets Manager.
- `frontend.sh`: builds, uploads, and invalidates CDN caches for the Vite frontend.

## 10. What I Learned
- Tuning WebRTC signaling is half networking, half psychology—good UX means surfacing connection states in real-time.
- Redis becomes the backbone when multiple backend nodes must agree on transient state; pub/sub plus session storage kept everything consistent.
- Observability matters early: custom metrics for "active rooms" and "code latency" were key to sizing Auto Scaling and spotting regressions.
- Security cannot be bolted on; designing STOMP channel authorization and signed TURN credentials up front saved countless headaches.

## 11. Roadmap
1. Add AI copilots for interviewer hints and automated candidate feedback.
2. Ship recording + transcription pipeline via AWS Kinesis + Transcribe.
3. Expand whiteboard to support sticky notes and templates for design interviews.
4. Harden compliance (SOC 2 style logging, data retention policies) for enterprise pilots.

## 12. End-to-End User Journey
1. **Account onboarding** – Admins invite interviewers via SES; they verify email, set MFA, and land on a tailored dashboard.
2. **Room creation** – Interviewers define objectives (interview/class), select participants, attach question banks, and configure optional NDA prompts.
3. **Scheduling + invitations** – Timezone-aware scheduler blocks calendar slots; candidates receive deep-linked invites with RSVP tracking.
4. **Pre-call checks** – Browser performs device permissions, TURN reachability, and network scoring (poor/medium/good) so interviewers can prepare fallback options.
5. **In-call experience** – Participants collaborate via WebRTC, Monaco editor, whiteboard, shared notes, and real-time prompts.
6. **Post-call workflow** – Session summary, code snapshots, and interviewer notes are stored in MySQL; email recaps and follow-up actions are sent automatically.

## 13. Developer Workflow & Tooling
- **Branching strategy**: trunk-based with short-lived feature branches, validated by GitHub Actions (backend Maven + frontend pnpm matrix).
- **Static analysis**: Spotless + Checkstyle for Java, ESLint + Biome for TS/React, Husky hooks to block non-compliant commits.
- **Testing layers**: JUnit + Testcontainers for integration (MySQL, Redis), Cypress component tests for the frontend, Playwright smoke suite for full meeting flows.
- **Secrets management**: AWS Secrets Manager + Parameter Store feed Spring config; local dev uses `.env` templates generated via `setup_local_rabbitmq.sh` & friends.
- **Release automation**: `deploy_full_stack.py` tags Docker images, updates launch templates, and triggers a rolling ASG refresh with zero downtime.

## 14. Reliability Playbook
- **Scaling triggers**: CloudWatch alarms on active rooms, Redis CPU, and editor latency automatically adjust ASG desired capacity.
- **Chaos drills**: Quarterly simulations kill WebSocket nodes or Redis replicas to validate session failover and user notifications.
- **Backup strategy**: RDS automated snapshots + point-in-time recovery; Redis snapshots stored in S3; TURN configs mirrored in Secrets Manager.
- **Incident response**: Slack + PagerDuty alerts sourced from CloudWatch alarms, with runbooks documenting mitigation steps per service.

## 15. Metrics That Matter
- Median join time: **<1.5s** from invitation link click to authenticated dashboard load.
- WebRTC connection success: **97%** when TURN reachable, backed by custom telemetry on ICE failures.
- Code editor latency: **<120ms** p95 round-trip for diff patches across regions.<br>
- Email delivery: **<30s** SES-to-inbox for verification & invites, monitored via SNS feedback loops.
- Cost efficiency: **40%** lower monthly spend vs. SaaS alternatives by scaling-to-zero non-peak environments.

## 16. Testing & QA Highlights
- Synthetic users replay real meeting flows nightly to catch regressions before they impact real interviews.
- Load tests (k6 + WebRTC-specific harness) validate 100 concurrent rooms × 4 participants without QoS degradation.
- Security scanning via OWASP ZAP and dependency-review-action ensures I ship hardened builds.
- Manual exploratory sessions focus on nuanced UX like interviewer scoring, whiteboard conflict handling, and mobile-first layouts.

## 17. Want a Closer Look?
- Happy to walk through architecture diagrams, demo flows, or specific code paths over a call.
- Reach out if you are exploring interview platforms, edtech collaboration, or real-time infrastructure—always open to feedback and partnerships.
