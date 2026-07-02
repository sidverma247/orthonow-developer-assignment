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

When a patient submits the consultation form, the browser sends a secure **HTTPS POST** request to a custom backend API built using **Node.js and Express**. The backend first validates the submitted data before interacting with external services.

The first integration is with **HubSpot CRM**. Instead of directly creating a contact, the backend uses the **HubSpot CRM Search API** to search for an existing contact using the **Phone** property. This step is necessary because **HubSpot's default deduplication is based on email addresses, not phone numbers**. Since this landing page only collects the patient's Name, Phone Number, and Clinic Preference, directly creating contacts could generate duplicates.

If a matching phone number exists, the backend updates the existing contact with the latest information, including **Clinic Preference**, **Source = "Google Ads - Consultation Landing Page"**, and **Lead Status = "New Enquiry"**. If no contact is found, a new record is created using the **HubSpot CRM Contacts API**.

After the CRM operation completes successfully, the backend places a WhatsApp notification request into a **message queue**. A background worker consumes the queue and sends the confirmation message through the **Karix WhatsApp Business API**. Processing WhatsApp notifications asynchronously keeps the landing page responsive while ensuring messages are delivered within the required two-minute SLA.

Once the backend confirms successful processing, the landing page executes a `window.dataLayer.push()` event named **`consultation_form_submitted`**. **Google Tag Manager** captures this event, forwards it to **Google Analytics 4**, and the GA4 conversion is imported into **Google Ads**. This creates a single, consistent conversion source for reporting and Smart Bidding optimization.

### Technology Choices and Justification

| Technology                               | Purpose                          | Why Chosen                                                                                        |
| ---------------------------------------- | -------------------------------- | ------------------------------------------------------------------------------------------------- |
| **Custom Backend API (Node.js/Express)** | Business logic and orchestration | Provides full control, security, scalability, validation, and centralized integration management. |
| **HubSpot CRM Search API**               | Search existing contacts         | Prevents duplicate contacts by checking the phone number before creating a new record.            |
| **HubSpot CRM Contacts API**             | Create or update contacts        | Enables complete control over CRM data without relying on embedded forms.                         |
| **Karix WhatsApp Business API**          | Send confirmation messages       | Supports automated WhatsApp notifications with reliable delivery.                                 |
| **Google Tag Manager (GTM)**             | Capture marketing events         | Centralizes analytics implementation without modifying application logic.                         |
| **Google Analytics 4 (GA4)**             | Record conversions               | Provides reporting, attribution, funnel analysis, and audience creation.                          |
| **Google Ads (Imported Conversion)**     | Campaign optimization            | Uses the GA4 conversion event to improve Smart Bidding performance.                               |

### Why Not Other Options?

* **HubSpot Forms Embed:** Not suitable because the landing page requires a fully customized form, custom validation, and custom analytics implementation.
* **Zapier:** Adds unnecessary latency, limited error handling, and less control over retries and monitoring.
* **Make:** Suitable for simple workflows but less appropriate for a production healthcare application that requires strict reliability, deduplication, security, and scalability.
* **Direct Browser-to-HubSpot API Calls:** Not recommended because API credentials would be exposed to the client, making the solution insecure and difficult to maintain.

A custom backend architecture provides the greatest flexibility, stronger security, better monitoring, easier maintenance, and room for future enhancements while meeting OrthoNow's business and technical requirements.

---

# Question 2

## What is the single biggest failure point in this setup, and how would you build a fallback for it?

## Solution

The **single biggest failure point** in this architecture is the **HubSpot CRM API**. Since HubSpot acts as the system of record for all consultation enquiries, any outage, timeout, or API failure could prevent new patient information from being stored. This could result in lost leads, duplicate form submissions, or delays in contacting potential patients, directly affecting business revenue.

To make the integration resilient, I would introduce a **message queue** (such as **AWS SQS** or **RabbitMQ**) between the backend API and the external services. When the user submits the consultation form, the backend immediately validates the request and stores a job in the queue before responding to the user. This ensures the user's submission is safely recorded even if HubSpot is temporarily unavailable.

A background worker then processes queued jobs by communicating with the **HubSpot CRM Search API** and **HubSpot CRM Contacts API**. Before creating a contact, the worker searches HubSpot using the **Phone** property. If a matching contact exists, it updates the record; otherwise, it creates a new one. This approach avoids duplicate contacts while handling the limitation that HubSpot's default deduplication is based on email rather than phone number.

If the HubSpot API is unavailable, the worker automatically retries the request using **exponential backoff** (for example, after 30 seconds, 1 minute, and 2 minutes). Each request includes an **idempotency key** or unique request identifier so that repeated retries cannot accidentally create duplicate contacts once the API becomes available again.

If all retry attempts fail, the job is moved to a **Dead Letter Queue (DLQ)** for manual investigation instead of being discarded. The system records structured logs containing the request ID, timestamp, retry count, and API response, while monitoring tools such as **AWS CloudWatch**, **Grafana**, or **Datadog** generate alerts whenever failures exceed predefined thresholds.

This architecture ensures that temporary HubSpot outages do not result in lost patient enquiries. By combining asynchronous processing, automatic retries, idempotency, dead-letter queues, and proactive monitoring, the system remains reliable, fault tolerant, and capable of recovering automatically from transient failures while protecting valuable healthcare leads.

---

# Question 3

## The WhatsApp message must fire within 2 minutes. What could break this SLA and how would you monitor it?

## Solution

The biggest risks to the **2-minute WhatsApp delivery SLA** are backend processing delays, message queue congestion, **Karix WhatsApp Business API** latency or downtime, network timeouts, and API rate limiting. For example, during periods of high traffic, a large number of consultation requests could create a queue backlog, delaying message processing. Similarly, if the Karix API experiences high response times or temporary outages, WhatsApp confirmations may not be delivered within the required timeframe.

To ensure the SLA is consistently met, I would process WhatsApp notifications **asynchronously** using a message queue such as **AWS SQS** or **RabbitMQ**. After the backend successfully creates or updates the patient's record in HubSpot, it immediately places a WhatsApp notification request into the queue. A dedicated background worker continuously monitors the queue and sends messages through the **Karix WhatsApp Business API**. This approach prevents external API delays from slowing down the user's form submission experience.

To monitor the SLA, I would track several operational metrics:

* **Queue depth** – the number of pending WhatsApp notification jobs.
* **Oldest message age** – how long the oldest queued job has been waiting.
* **End-to-end processing time** – the time between form submission and successful WhatsApp delivery.
* **Karix API response time** – average and peak response times.
* **API success and failure rates** – percentage of successful and failed requests.
* **Delivery confirmation status** – message delivery receipts received from Karix webhooks.

These metrics would be monitored using tools such as **AWS CloudWatch**, **Grafana**, or **Datadog**, with automated alerts configured for abnormal conditions. For example, alerts would be triggered if the oldest queued message exceeds **60 seconds**, if end-to-end delivery time approaches **90 seconds**, or if the Karix API error rate rises above an acceptable threshold. This provides enough time to investigate and recover before the two-minute SLA is breached.

If message delivery fails due to temporary issues such as network timeouts or rate limiting, the worker automatically retries the request using **exponential backoff**. If all retry attempts fail, the job is moved to a **Dead Letter Queue (DLQ)** for manual investigation while notifying the operations team. This monitoring and retry strategy ensures high reliability, minimizes message delays, and helps OrthoNow consistently meet its two-minute WhatsApp delivery SLA.
