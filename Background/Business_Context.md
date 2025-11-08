# SwitchUp Business Context

This document outlines the core business of SwitchUp and the main challenge.

## 1. About SwitchUp

```mermaid
graph TD
    SwitchUp["SwitchUp"] --> ContractMonitoring["Contract Monitoring & Optimisation"]
    ContractMonitoring --> Energy["Energy<br>Electricity, Gas"]
    ContractMonitoring --> Telco["Telco<br>Broadband, Mobile"]
    ContractMonitoring --> Insurance["Insurance<br>Car, Home, Liability"]
    ContractMonitoring --> Other["Other<br>Subscription Contracts"]

    classDef switchup fill:#e1f5fe,stroke:#01579b,stroke-width:2px
    classDef monitoring fill:#f3e5f5,stroke:#8e24aa
    classDef service fill:#bbdefb,stroke:#0d47a1

    class SwitchUp switchup
    class ContractMonitoring monitoring
    class Energy,Telco,Insurance,Other service
```

[SwitchUp](https://www.switchup.de) acts as a "subscription contract guard", continuously monitoring and optimizing consumer contracts. The service covers a range of recurring, subscription-like contracts, including:

* **Energy:** Electricity, gas, heating electricity.
* **Telco:** Broadband, mobile.
* **Insurance:** Car, home, liability insurance.
* **Other:** Various recurring contract types.

The core value proposition involves identifying contract events (like price increases from provider emails), comparing the current contract with both market offers and retention offers, recommending the best course of action (switch or stay), and automating the optimization process, often by switching the user to a better offer.

## 2. Main Challenge: Operational Scalability

```mermaid
graph TD
    Challenge["Operational Scalability Challenge"] --> Manual["Manual Processes"]
    Manual --> Workflow["Manual Workflow Execution<br>(Failed Orders, etc.)"]
    Manual --> Data["Manual Data Extraction<br>(Price Increases, etc.)"]
    Manual --> Service["Manual Customer Service<br>Repetitive Inquiries"]

    classDef challenge fill:#ffebee,stroke:#b71c1c,stroke-width:2px
    classDef manual fill:#f5f5f5,stroke:#424242
    classDef issue fill:#ffebee,stroke:#b71c1c

    class Challenge challenge
    class Manual manual
    class Offers,Workflow,Data,Service issue
```

SwitchUp's operations are highly process-driven, revolving around contract events and their corresponding workflows. However, many activities within these workflows still require manual intervention. Key challenges include:

* **Manual Workflow Execution:** Handling the various steps required for specific contract events (e.g., handling a failed order, processing a cancellation, managing price increase notifications and subsequent comparisons including retention options).
* **Manual Data Extraction:** Processing incoming provider communications (emails, documents) to identify events and extract relevant data (e.g., new prices for adjustments, contract dates, *retention offer* details).
* **Manual Customer Service:** Responding to user inquiries, many of which are repetitive, including questions about contract state changes due to adjustments (like price increases).

This reliance on manual effort limits operational scalability, hindering the ability to handle a significantly larger volume of contracts and processes efficiently. The primary goal is to automate the vast majority of these processes.
