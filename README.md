# System Design
![Trusted_Notifications_Reliable_Safe_and_Timely_Design](https://github.com/user-attachments/assets/217d4850-42e7-4453-b6b4-159ed8246acb)

# Trusted Notifications — Reliable, Safe, and Timely

## Problem Statement
Users have been reporting issues with transactional notifications such as OTPs and banking alerts.  
Key problems observed:
- Some alerts arrive late or are missed entirely.
- Users occasionally get confused by spoofed or suspicious messages.
- Retry behaviour is inconsistent across channels.
- No clear routing logic from event → channel.

**Goal:** Build a notification system that is **reliable**, **safe**, and **timely**, ensuring every critical alert reaches customers through trusted channels with simple retries and anti-spoof protections.

Success looks like:
- Clear mapping of event types to delivery channels.
- Controlled, consistent retry logic.
- Guardrails preventing spoofing or unauthorized message formats.
- Transparent, traceable flow from event creation → delivery.

---

## System Design Overview

This design uses AWS-managed components (Lambda, SNS, SQS, DynamoDB, SES, Pinpoint, FCM/APNs) to create a **decoupled**, **fault-tolerant**, and **observable** notification pipeline.

---

## High-Level Flow

### 1. Source System  
The banking system generates a notification event (e.g., OTP, fraud alert) and sends it to the **Notification Service**.

### 2. Notification Service (AWS Lambda)  
- Validates and normalizes the incoming event.  
- Fetches user preferences and verified contact details from **DynamoDB**.  
- Determines which channels to use (SMS, Email, Push).  
- Publishes a single enriched message to an **SNS Topic**.

### 3. SNS Fan-Out  
SNS fans out the event into multiple **channel-specific SQS queues**:
- `SMS Queue`
- `Email Queue`
- `Push Queue`

Each channel processes messages independently without affecting others.

### 4. Channel Workers (Lambdas)  
Each queue triggers its own worker:
- **SMS Worker** → sends via Amazon Pinpoint  
- **Email Worker** → sends via Amazon SES  
- **Push Worker** → sends via FCM/APNs  

The workers:
- Apply trusted templates  
- Validate message authenticity  
- Attempt delivery  
- Log success or failure

### 5. Failure Handling  
If a worker repeatedly fails (e.g., provider errors, retries exceeded), the message is moved to the **Dead Letter Queue (DLQ)**.

A **Retry Logic Lambda** processes the DLQ:
- Applies controlled retries  
- Uses backoff timing  
- Escalates or marks final failure if needed  

### 6. Customer Device  
Once successfully delivered by the provider (SMS/Email/Push), the notification reaches the user’s device.

---

## Why This Design Works

### ✔ Reliability  
- Event fan-out and per-channel SQS decouple failures.  
- Consistent retry behaviour driven by SQS + DLQ + Retry Lambda.  

### ✔ Safety & Anti-Spoofing  
- Only approved templates and sender IDs.  
- Verified user contact details from DynamoDB.  
- Signed/validated message payloads.  

### ✔ Timeliness  
- Parallel channel processing.  
- Managed infrastructure ensures high throughput and low latency.  

### ✔ Observability  
- Each step is logged and measurable.  
- Clear audit trail from event → delivery.

---

## Summary
This architecture ensures transactional notifications—especially time-critical alerts like OTPs—are:
- Delivered quickly  
- Protected from spoofing  
- Retried safely when needed  
- Tracked end-to-end  

It provides a clean, maintainable foundation for trusted customer communication at scale.

