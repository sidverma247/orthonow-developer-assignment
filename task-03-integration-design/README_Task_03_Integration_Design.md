# Task 03 – Integration Design

## Overview

This document outlines the proposed integration architecture for OrthoNow's consultation landing page. The objective is to ensure that every consultation enquiry is reliably captured in HubSpot CRM, an automated WhatsApp confirmation is delivered within two minutes through Karix, and the corresponding conversion is recorded in Google Ads via Google Analytics 4 (GA4) and Google Tag Manager (GTM).

---

# Question 1

## How would you architect this integration end-to-end? What connects to what, in what order, and what technology sits at each step? Be specific—name the actual tools/methods (HubSpot Forms API vs. native embed vs. Zapier vs. Make vs. direct API call—pick one and justify it).

## Solution

For this integration, I would use a **custom backend with the HubSpot CRM API** rather than the HubSpot Forms API, native HubSpot form embed, Zapier, or Make. Since the landing page uses a custom HTML form and requires custom validation, CRM logic, WhatsApp automation, and analytics tracking, a backend API provides greater control, security, scalability, and reliability than no-code automation platforms.

### End-to-End Architecture

```text
User
   │
   ▼
Landing Page (HTML/CSS/JavaScript)
   │
   │ HTTPS POST (/api/consultation)
   ▼
Backend API (Node.js/Express)
   │
   ├── Validate form data
   ├── Search HubSpot by phone number
   ├── Update existing contact or create new contact
   ├── Queue WhatsApp confirmation
   └── Return success response
   │
   ▼
Landing Page
   │
   ├── window.dataLayer.push()
   ├── Google Tag Manager (GTM)
   ├── Google Analytics 4 (GA4)
   └── Google Ads Imported Conversion
```

### Implementation Details

When a patient submits the consultation form, the browser sends a secure HTTPS POST request to a custom backend API built using Node.js and Express. The backend validates the data, searches HubSpot using the CRM Search API by **Phone**, updates the existing contact if found, or creates a new contact using the HubSpot CRM Contacts API. This search step is required because HubSpot deduplicates by **email**, not by phone.

After the CRM operation succeeds, the backend places a WhatsApp notification request into a message queue. A background worker processes the queue and sends the confirmation message through the Karix WhatsApp Business API. Once processing is successful, the landing page executes `window.dataLayer.push({ event: "consultation_form_submitted" })`. GTM forwards the event to GA4, and the GA4 conversion is imported into Google Ads for Smart Bidding.

### Technology Choices and Justification

| Technology | Purpose | Why Chosen |
|------------|---------|------------|
| Custom Backend API (Node.js/Express) | Business logic | Secure, scalable orchestration layer |
| HubSpot CRM Search API | Search contacts | Prevent duplicate contacts using phone lookup |
| HubSpot CRM Contacts API | Create/Update contacts | Full control over CRM records |
| Karix WhatsApp Business API | WhatsApp notifications | Reliable business messaging |
| Google Tag Manager | Event tracking | Centralized analytics implementation |
| Google Analytics 4 | Conversion tracking | Reporting and attribution |
| Google Ads Imported Conversion | Campaign optimization | Enables Smart Bidding |

### Why Not Other Options?

- **HubSpot Forms Embed:** Limited customization and analytics control.
- **Zapier:** Additional latency and limited retry/error handling.
- **Make:** Better suited to simpler workflows than production healthcare integrations.
- **Direct Browser-to-HubSpot Calls:** Exposes credentials and business logic.

---

# Question 2

## What is the single biggest failure point in this setup, and how would you build a fallback for it?

### Solution

The biggest failure point is the **HubSpot CRM API**. If HubSpot is unavailable, consultation enquiries could fail to be recorded.

To prevent data loss, I would introduce a **message queue** (AWS SQS or RabbitMQ). The backend validates the request and places it into the queue before responding to the user. A background worker then searches HubSpot by phone and creates or updates the contact.

If HubSpot is unavailable, the worker retries using **exponential backoff** and an **idempotency key** to prevent duplicate records. If all retries fail, the request is moved to a **Dead Letter Queue (DLQ)**. Logs are captured, and alerts are sent through CloudWatch, Grafana, or Datadog.

This design ensures temporary HubSpot outages do not result in lost leads and provides reliable recovery once the service becomes available.

---

# Question 3

## The WhatsApp message must fire within 2 minutes. What could break this SLA and how would you monitor it?

### Solution

The primary risks are backend delays, queue congestion, Karix API latency or downtime, network failures, and API rate limiting.

To meet the two-minute SLA, WhatsApp notifications are processed asynchronously through a message queue. After HubSpot processing succeeds, a worker immediately sends the message through the Karix WhatsApp Business API.

Monitoring includes:

- Queue depth
- Oldest message age
- End-to-end processing time
- Karix API response time
- Success/failure rates
- Delivery confirmations

Metrics are monitored using AWS CloudWatch, Grafana, or Datadog. Alerts are configured if queue age exceeds 60 seconds, delivery time approaches 90 seconds, or API error rates increase. Failed messages are retried using exponential backoff, and persistent failures are moved to a Dead Letter Queue (DLQ) for manual investigation.
