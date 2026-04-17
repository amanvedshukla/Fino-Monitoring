# Fino Payments Bank — Monitoring & Systems Learning Documentation

> Author: Aman Shukla
> Domain: Fintech Infrastructure Monitoring | P2M Pay System

---

## Important Note

This document is a work-in-progress learning record. I am currently in the early stages of understanding Fino's monitoring infrastructure. Most of the concepts documented here are things I have just started to grasp — many are still not fully clear to me. I am learning by observing live dashboards, reading system logs, and asking questions. This document reflects where I currently stand, not where I need to be.

---

## Table of Contents

1. [Overview](#overview)
2. [P2M Pay — System Architecture](#p2m-pay--system-architecture)
3. [Kiali Service Graph — P2M Transaction Flow](#kiali-service-graph--p2m-transaction-flow)
4. [Key Technical Concepts](#key-technical-concepts)
5. [Monitoring Metrics Decoded](#monitoring-metrics-decoded)
6. [Settlement Flow](#settlement-flow)
7. [Pending / Not Yet Understood](#pending--not-yet-understood)

---

## Overview

I have been monitoring the P2M (Person to Merchant) Pay system at Fino Payments Bank. P2M refers to payments made from a customer to a merchant — via QR code, UPI, or online channels.

My current focus areas:
- Trying to understand the backend architecture of P2M Pay
- Reading monitoring dashboards (Kiali, Jaeger, metrics panels)
- Decoding system logs and settlement metrics
- Understanding how money flows from transaction to CBS to merchant

---

## P2M Pay — System Architecture

### What is P2M Pay?

P2M = Person to Merchant payment. When a customer pays a merchant via QR code, UPI, or online — that is a P2M transaction.

### High-Level Request Flow

```
Customer
    |
    v
p2m-pay-nginx
(Entry point — Reverse Proxy)
    |
    v
p2m-pay-transaction-request
(API / Request handling layer)
    |
    v
transactionrequest-jvm
(Core Backend — Java/JVM — Business Logic)
    |
    |-----> p2m-pay-transaction-impsshapeup     (IMPS payment rail)
    |-----> p2m-pay-transaction-masterdata      (Config & reference data)
    |-----> p2m-pay-transaction-getduplicate    (Duplicate / fraud check)
    `-----> jaeger-collector (p2mpay-istio)     (Distributed tracing)
```

### Architecture Components

| Component | Role |
|-----------|------|
| `p2m-pay-nginx` | Reverse proxy — entry gate for all traffic |
| `p2m-pay-transaction-request` | Receives and routes transaction requests |
| `transactionrequest-jvm` | Core JVM backend — runs business logic |
| `p2m-pay-transaction-impsshapeup` | IMPS bank rail integration |
| `p2m-pay-transaction-masterdata` | Master config and reference data service |
| `p2m-pay-transaction-getduplicate` | Detects duplicate transactions |
| `jaeger-collector` | Collects distributed traces for debugging |

---

## Kiali Service Graph — P2M Transaction Flow

Kiali is a visualization tool for microservices (works with Istio service mesh). It shows how services communicate, along with traffic rates, latency, and errors.

### Graph — As Observed

```
                                                                                         [p2m-pay-transaction-impsshapeup] --0.35rps/247ms--> [p2m-pay-transaction-impsshapeup]
                                                                                        /
                                                                                       /  [jaeger-collector (p2mpay-istio)] --0.42rps/4ms---> [jaeger (p2mpay-istio)]
                                                                                      /  /
[p2m-pay-nginx] --0.35rps/249ms--> [p2m-pay-transaction-request] --0.35rps/249ms--> [transactionrequest-jvm]
                                                                                      \  \
                                                                                       \  [p2m-pay-transaction-masterdata] --0.35rps/5ms--> [p2m-pay-transaction-masterdata]
                                                                                        \
                                                                                         [p2m-pay-transaction-getduplicate] --0.35rps/9ms--> [p2m-pay-transaction-getduplicate]

All connections have mTLS active (lock icon) — encrypted service-to-service communication via Istio.
```

### What the Metrics on the Graph Mean

| Metric | Meaning |
|--------|---------|
| `0.35 rps` | Requests per second — how much traffic is flowing on that connection |
| `247ms` | Latency — how long that hop is taking to respond |
| `1.16 kbps` | Data transfer rate on that connection |
| Lock icon | mTLS active — all communication is encrypted |
| Green lines | No errors detected — system healthy at time of observation |

### Observations from the Graph

- All service connections were green at the time of observation — no active errors.
- `transactionrequest-jvm` is the central hub — every downstream service connects through it.
- `jaeger-collector` was receiving traces from the JVM service for distributed tracing.
- mTLS was active across all connections, meaning inter-service communication is secured via Istio service mesh.
- The highest latency observed was on the `impsshapeup` path (247ms), which involves actual bank rail communication.
- `masterdata` (5ms) and `getduplicate` (9ms) are fast internal lookups by comparison.

---

## Key Technical Concepts

### Infrastructure

| Concept | What I Currently Understand |
|---------|----------------------------|
| Proxy Server | A middleman between client and server. Hides identity, adds a control layer. |
| Reverse Proxy | Sits in front of backend servers and routes incoming requests inward. nginx does this for P2M. |
| HAProxy | High Availability Proxy — load balancer that distributes traffic across multiple backend servers. |
| API Gateway | Smart entry point — handles authentication, routing, rate limiting, and logging. |
| haproxy-bll | HAProxy instance specifically handling Business Logic Layer traffic. |
| api_gw_haproxy-bll | Full route identifier: API Gateway to HAProxy to Business Logic Layer. |
| Namespace | Logical grouping for services — e.g., `p2mpay-application` groups all P2M Pay services together. |
| Load Balancing | Distributing requests across servers to prevent any single server from being overloaded. |

### Monitoring and Tracing

| Tool | What I Currently Understand |
|------|-----------------------------|
| Kiali | Visual service graph showing which service talks to which, with live traffic and latency data. |
| Jaeger | Distributed tracing tool — tracks the full journey of a single request across all services. |
| Trace | The complete end-to-end journey of one request through the system. |
| Span | A single step or hop within a trace (e.g., one DB call = one span). |

### Banking and Finance

| Concept | What I Currently Understand |
|---------|----------------------------|
| CBS (Core Banking System) | The bank's main software where actual account balances are updated. Final source of truth for money. |
| Settlement | The actual transfer of money to the merchant's bank account — happens after the transaction is processed. |
| GL (General Ledger) | The accounting record of all financial transactions within the system. |
| Recon Report | Reconciliation — matching system records against bank records to find any mismatches. |
| T+1 Settlement | Money reaches the merchant the next day after the transaction date. |

---

## Monitoring Metrics Decoded

### Transaction Metrics

| Metric | What It Means |
|--------|---------------|
| `transaction_count` | Total number of transactions processed (includes success, failed, and pending) |
| `Transaction req-jvm` | Log entry indicating a transaction request is being processed by the JVM backend |
| `API traffic by endpoint` | Request count broken down per API route — e.g., `/payment`, `/status`, `/balance` |
| `Namespace: p2mpay-application` | Identifies that the log or data belongs to the P2M Pay backend namespace |

### Settlement Metrics

| Metric | What It Means |
|--------|---------------|
| `partner_settlement_count_p2mpay` | How many times settlement was processed for a merchant partner |
| `Partner_settled_at_cbs_success_count` | How many times money was successfully credited in CBS (the bank's core system) |
| `partner_cbs_response_failed_count` | How many times CBS rejected or returned a failure response for a settlement |
| `No_transaction_to_settle_GL_count` | Settlement process ran but found no pending transactions in the General Ledger |
| `Settlement_failed_to_add_in_p2mpay_logical_db` | Settlement completed at bank side but the DB entry in P2M system failed — reconciliation risk |

### HTTP Status Codes Observed in API Monitoring

| Code | Meaning | Context in P2M |
|------|---------|----------------|
| `200 OK` | Success | Payment processed correctly |
| `400 Bad Request` | Client-side error | Malformed or invalid request |
| `401 Unauthorized` | Auth failure | Missing or invalid token |
| `403 Forbidden` | Access denied | Permission issue |
| `404 Not Found` | Resource missing | Wrong endpoint called |
| `500 Internal Server Error` | Backend crash | Server-side bug or unhandled exception |
| `502 Bad Gateway` | Upstream failure | Backend or gateway is down |
| `504 Gateway Timeout` | Slow response | Server is overloaded or not responding in time |

---

## Settlement Flow

```
Customer Payment
       |
       v
  API Gateway
       |
       v
  HAProxy (BLL)
       |
       v
  Backend JVM (Business Logic)
       |
       v
  Settlement Engine
       |
  +----+----+
  |         |
  v         v
 GL        CBS (Core Banking System)
(record)  (actual bank credit)
            |
            v
     Merchant Bank Account
```

### Settlement Stages

1. Transaction — payment initiated, money deducted from customer account.
2. Processing — backend validates and routes the payment through JVM services.
3. Settlement Initiated — system prepares the transfer to merchant.
4. GL Entry — record is added to the General Ledger for accounting.
5. CBS Credit — bank's core system credits the merchant's account.
6. Reconciliation — system records and bank records are matched to confirm everything is correct.

---

## Pending / Not Yet Understood

The following topics have been identified but are not yet clear to me. I have only surface-level exposure to these so far.

- [ ] UPI System — full payment rail flow, NPCI involvement, VPA resolution process
- [ ] EBS (Electronic Banking System) — what it is and how it connects to P2M monitoring
- [ ] P2M Kiali Graph — deeper understanding of all nodes, error states, and what to look for when something goes red
- [ ] Jaeger Trace Reading — how to actually read a trace, identify slow spans, debug a real issue
- [ ] Recon Report — actual format, how mismatches are identified and resolved
- [ ] IMPS Rail — what `p2m-pay-transaction-impsshapeup` does internally

---

*Documentation prepared by: Aman Shukla*
