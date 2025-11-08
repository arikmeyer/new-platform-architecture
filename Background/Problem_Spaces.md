# SwitchUp Problem Spaces

*This document outlines the fundamental challenges SwitchUp addresses, categorized by the core domains identified in the [*System Context*](./System_Context.md).*

## 1. Overall Challenge

### 1.1 Operational Scalability (Core Challenge)

* **Problem Definition:** Current operational processes heavily rely on manual intervention for handling diverse contract events, data extraction, provider interactions, and customer support. This approach does not scale effectively nor reliably to manage millions of contracts, creating bottlenecks, limiting growth, and increasing the risk of errors.
* **Intended Outcome:** A highly automated operational approach is established that dynamically adapts to growth in volume and complexity, requiring human intervention primarily for strategic direction rather than routine execution or exception handling.
* **Frequency/Scale:** Continuous, fundamental constraint; defines the boundary conditions for the business's achievable scale and scope.
* **Intermediate Approach (Admin as Operator):** Humans architect and operate configurable tools, manually bridging gaps and handling the ever-increasing volume of exceptions and adaptations required by external reality. Scale remains coupled to human effort.
* **Target Approach (Admin as Supervisor):** AI agents, governed by human-guided high-level objectives and ethical constraints, autonomously manage the operational lifecycle. They learn, adapt, predict, and self-heal the system in response to internal and external stimuli, requiring human intervention for reviews, strategic goal adjustments, and directional shifts.
* **Prerequisites:** Modular, observable system architecture; Clearly defined operational objectives and constraints; Advanced AI orchestration and reinforcement learning frameworks; Robust monitoring and feedback loops.

## 2. Offer Domain

### 2.1 Offer Modelling (Universal Service Ontology)

* **Problem Definition:** The lack of a unified, flexible data model forces inconsistent and often incomplete representations of complex offers across different markets (Energy, Telco, Insurance). The current legacy structure or manual ad-hoc methods cannot adequately capture diverse pricing structures, feature sets, eligibility rules, and geographic availability, hindering accurate comparison and automated processing.
* **Intended Outcome:** A unified, extensible provider and market agnostic *Offer Data Model* ontology serves as the canonical language for representing all service agreements, enabling consistent interpretation and processing across disparate markets.
* **Frequency/Scale:** Foundational; requires robust initial definition and continuous evolution to accommodate novel offer structures or market dynamics; impacts the representation of thousands of products.
* **Intermediate Approach (Offer Admin as Operator):** Human specialists translate domain knowledge into predefined model components (*Blueprints*, *Policies*, *Entities*) using specialized configuration tools, requiring deep expertise for each new domain.
* **Target Approach (Offer Admin as Supervisor):** Generative AI models analyze diverse examples (documents, data feeds, web pages) and propose or automatically generate the necessary schema extensions (*Blueprints*) and configurations to represent new or complex offer structures with human guidance. Humans validate and refine the AI's proposed models.
* **Prerequisites:** Core extensible data modeling framework; Large dataset of diverse offer examples for AI training/fine-tuning; AI model governance and validation framework; Tools for visualizing and validating complex offer models.

### 2.2 Offer Normalisation (Semantic Translation & Harmonization)

* **Problem Definition:** Significant manual effort and high error rates are currently involved in transforming inconsistent, incomplete, and diversely formatted offer data received from numerous external sources into a standardized internal structure. This includes standardizing units, date formats, terminology, and location identifiers needed for reliable processing.
* **Intended Outcome:** All ingested external offer data, regardless of origin or format, is reliably interpreted and transformed into the canonical internal model with high semantic fidelity.
* **Frequency/Scale:** Continuous, high-volume transformation challenge across dozens of external sources.
* **Intermediate Approach (Offer Admin as Operator):** Admins maintain brittle, source-specific transformation rules (mapping, parsing, unit conversion) that require frequent manual adjustment as external sources evolve.
* **Target Approach (Offer Admin as Supervisor):** Adaptive AI systems perform dynamic semantic interpretation and normalization of diverse input formats with minimal explicit rules, learning translation patterns from examples and corrections.
* **Prerequisites:** Canonical internal *Offer Data Model*; Robust data validation framework; Advanced AI models for data interpretation and normalization; Efficient feedback mechanisms for AI learning.

### 2.3 Offer Matching (Canonical Identity Disambiguation)

* **Problem Definition:** Incoming offer data from external sources frequently lacks stable unique identifiers or uses inconsistent naming, making it difficult to automatically link these records to the correct canonical offer definition within SwitchUp's system. This results in significant manual effort for review and resolution, risking the creation of duplicate definitions or incorrect linkages.
* **Intended Outcome:** Every external offer instance is automatically and accurately linked to its unique canonical internal definition, establishing a clean foundation for aggregation and comparison.
* **Frequency/Scale:** High-frequency operation (thousands daily) against a large, dynamic internal catalog, where matching errors have significant downstream consequences.
* **Intermediate Approach (Offer Admin as Operator):** Systems rely on configurable heuristics (identifier weighting, name similarity, static attribute comparison). Humans manually investigate and resolve a substantial fraction of low-confidence or conflicting matches.
* **Target Approach (Admin as Supervisor):** AI agents employ multi-modal reasoning (analyzing identifiers, names, detailed characteristics embeddings, source context, historical linkage data) to achieve near-perfect automated identity resolution, escalating only genuinely novel or conflicting cases requiring human judgment.
* **Prerequisites:** Comprehensive internal offer catalog with rich metadata; Access to source and channel context; AI models specialized in entity resolution and disambiguation; Continuous learning feedback loop.

## 3. Optimisation Domain

### 3.1 Optimal Offer Selection (Core Savings Calculation)

* **Problem Definition:** Current recommendation logic often relies on simplistic cost ranking or basic feature filtering, failing to identify the truly optimal offer for a user when considering the complex interplay of numerous features, time-varying costs (including bonuses, guarantees), personalized eligibility, usage patterns, and individual preferences.
* **Intended Outcome:** Every recommendation request receives a mathematically rigorous optimal offer selection that represents the maximum long-term value proposition for the user, considering both financial and non-financial factors according to their stated or inferred goals.
* **Frequency/Scale:** Requires rapid, complex optimization calculations executed millions of times annually, evaluating intricate trade-offs across numerous potential offer scenarios with high accuracy.
* **Intermediate Approach (Optimisation Admin as Operator):** Humans define explicit scoring functions or rule sets based on simplified models of user utility (e.g., weighted sums of cost and key features) to rank a pre-filtered set of eligible offers. Cost calculations use predefined formulas maintained manually.
* **Target Approach (Admin as Supervisor):** Advanced calculation engines (potentially incorporating machine learning models) perform comprehensive long-term value calculations for all available offers, automatically synthesizing financial projections with learned utility models based on user preferences and behavior patterns. Humans refine scoring models and validate algorithm performance.
* **Prerequisites:** Normalized offer data with complete pricing structures; Robust cost calculation engine capable of handling complex temporal pricing; User preference modeling framework; High-performance computation infrastructure; Real-time feedback integration for model refinement.

### 3.2 Switch Decision Assessment (Go/No-Go Evaluation)

* **Problem Definition:** Determining whether switching from a current contract to a new offer is genuinely beneficial requires complex calculations that current systems often oversimplify, risking recommendations that don't meet minimum value thresholds or align with business rules.
* **Intended Outcome:** Every proposed switch receives a rapid, definitive evaluation against configurable business rules and savings thresholds, ensuring only genuinely beneficial switches are recommended.
* **Frequency/Scale:** High-frequency decision support across millions of evaluation requests, requiring consistent application of complex calculation logic and dynamic business rules.
* **Intermediate Approach (Optimisation Admin as Operator):** Static business rules and simplified cost comparison logic manually configured and updated as market conditions change. Humans define and maintain threshold values.
* **Target Approach (Admin as Supervisor):** Stateless evaluation functions perform consistent, auditable go/no-go assessments using sophisticated value calculations and dynamically configurable business rules passed from orchestration layer. Humans focus on defining strategic business rules and monitoring aggregate decision quality.
* **Prerequisites:** Comprehensive contract and offer value calculation capability; Flexible business rules engine; Clear definition of success criteria; Audit trail for decision tracking.

## 4. Case Domain

### 4.1 Event Monitoring (Contract State Signal Detection)

* **Problem Definition:** Detecting critical contract lifecycle events relies heavily on manual review of emails and documents or brittle keyword-based rules, leading to significant delays, missed events, and inconsistencies in identifying state changes across millions of contracts.
* **Intended Outcome:** The true state and all significant lifecycle events for every contract *are* inferred with high probability in near real-time, enabling proactive and accurate process initiation.
* **Frequency/Scale:** Continuous, high-throughput analysis required across millions of contracts and associated communication streams.
* **Intermediate Approach (Case Admin as Operator):** Systems employ pattern matching and basic classification models focused on detecting explicit keywords or templates associated with a limited set of predefined, high-frequency events. Significant manual review of unclassified items.
* **Target Approach (Admin as Supervisor):** Advanced AI models (LLMs with temporal reasoning, causal inference capabilities) analyze communication streams holistically, inferring subtle state changes, predicting likely future events, and identifying anomalous event patterns requiring human investigation or new process definition.
* **Prerequisites:** Unified communication stream ingestion; AI models capable of deep semantic understanding and temporal state tracking; Framework for defining and detecting complex event patterns.

### 4.2 Workflow Orchestration (Goal-Oriented Process Synthesis)

* **Problem Definition:** Managing the complex, multi-step processes triggered by contract events is currently rigid and manually intensive. Existing workflow tools often lack the flexibility to handle exceptions gracefully or adapt execution based on real-time conditions or learned best practices, requiring frequent human intervention and process redesign.
* **Intended Outcome:** Business processes related to contract events *achieve* their defined goals through dynamically synthesized and optimized execution plans, adapting resiliently to unforeseen circumstances with minimal need for pre-scripted exception paths.
* **Frequency/Scale:** Managing millions of concurrent, potentially unique process executions, each evolving over extended periods.
* **Intermediate Approach (Case Admin as Operator):** Humans explicitly model workflows as static sequences using visual editors and predefined components. Exception handling is pre-defined and requires manual intervention for unexpected issues. Optimization is manual.
* **Target Approach (Admin as Supervisor):** AI planning agents decompose high-level goals into actionable task sequences, dynamically selecting and orchestrating resources (APIs, other AI agents, human tasks). They monitor execution, predict problems, dynamically replan to handle exceptions, and optimize strategies based on performance feedback.
* **Prerequisites:** Goal definition language; Library of semantically described, API-callable task components; AI planning and execution framework; Real-time state monitoring and feedback loop.

### 4.3 Task Management (Intelligent Work Allocation & Automation)

* **Problem Definition:** The sheer volume and diversity of operational tasks (manual data validation, customer follow-ups, exception handling) overwhelm current manual assignment and prioritization methods, leading to delays, inconsistent quality, and inefficient use of human agent time.
* **Intended Outcome:** Every task is routed to and executed by the optimal actor (human or AI) dynamically, ensuring high efficiency and quality across the hybrid human-AI team.
* **Frequency/Scale:** Continuously managing and allocating potentially millions of concurrent tasks with varying complexities, deadlines, and skill requirements across potentially thousands of human and AI colleagues.
* **Intermediate Approach (Case Admin as Operator):** Static rules assign predefined task types to specific internal/external users or basic automation handlers. Admins manage queue prioritization.
* **Target Approach (Admin as Supervisor):** An AI orchestration system dynamically assesses task requirements (complexity, skills, context), predicts optimal assignment (specific AI, specific human group, general queue), automates feasible tasks directly, monitors progress, and reallocates dynamically based on load, performance, and quality feedback.
* **Prerequisites:** Centralized task representation and tracking; AI models for task assessment and automation potential assessment; Real-time actor availability and capability mapping (human and AI); Advanced scheduling and orchestration engine.

## 5. Provider Domain

### 5.1 Provider Configuration (External System Knowledge Synthesis)

* **Problem Definition:** Maintaining an accurate internal understanding of the operational rules, technical interfaces, constraints, and geographic service boundaries for thousands of service and network providers is very difficult due to the information being scattered, undocumented, non-standardized, and constantly changing.
* **Intended Outcome:** A dynamic, self-updating internal knowledge graph serves as a high-fidelity digital twin of the provider ecosystem, enabling precise system interactions and accurate context resolution.
* **Frequency/Scale:** Requires continuous updates to reflect frequent changes across thousands of provider systems and multiple operational regions.
* **Intermediate Approach (Provider Admin as Operator):** Humans manually gather intelligence from disparate sources (documentation, interactions, error logs) and manually configure provider attributes, rules within internal systems. Updates are typically reactive.
* **Target Approach (Admin as Supervisor):** AI agents actively monitor a wide array of structured and unstructured sources (provider developer portals, websites, regulatory databases, operational logs, external geo-data), inferring changes in provider behavior, interfaces, or service areas, and autonomously proposing validated updates to the internal knowledge graph.
* **Prerequisites:** Internal knowledge graph framework; AI agents capable of web monitoring, information extraction, and change detection; Data pipelines for capturing operational interaction data; Validation framework for proposed knowledge updates.

### 5.2 API/Bot Automation (Dynamic Interface Negotiation)

* **Problem Definition:** Automating interactions with essential provider systems is severely hampered by the prevalence of unstable web portals designed for humans and the lack of stable, documented APIs, resulting in brittle automation scripts that require constant, costly manual maintenance.
* **Intended Outcome:** Critical interactions with provider systems are reliably automated, demonstrating self-adaptation capabilities to handle frequent interface changes with minimal manual intervention.
* **Frequency/Scale:** Requires managing hundreds of brittle integration points subject to frequent, unannounced changes.
* **Intermediate Approach (Bot/API Admin as Operator):** Developers maintain specific API clients and user RPA script building-blocks that are updated semi-manually when the break, requiring significant manual effort for debugging and rewriting based on observed failures.
* **Target Approach (Admin as Supervisor):** Adaptive AI agents (using techniques like visual UI understanding or learning interaction patterns) autonomously navigate and utilize external interfaces. They proactively detect interface changes, attempt automated script repair or adaptation, and provide detailed failure analysis when human intervention is required.
* **Prerequisites:** AI-driven automation platform with visual understanding and adaptive capabilities; Sophisticated monitoring for detecting subtle interaction failures; Framework for automated testing and validation of adapted scripts; Secure execution environment.

### 5.3 Data Extraction (Unstructured Knowledge Distillation)

* **Problem Definition:** Critical business data needed for automation (e.g., specific prices, dates, contract terms, identifiers) is often buried within large volumes of unstructured text in provider emails and documents, requiring inefficient and error-prone manual data entry or complex, brittle template-based extraction.
* **Intended Outcome:** Essential information buried in unstructured communications is automatically extracted and structured with high fidelity, fueling downstream processes without manual data entry.
* **Frequency/Scale:** Continuous processing of millions of diverse communications annually, requiring adaptation to evolving formats and complex linguistic variations.
* **Intermediate Approach (Extraction Admin as Operator):** Systems rely on manually crafted rules and partial LLM-based fine-tuned on domain knowledge with zero-shot or few-shot extractions. Humans continue to manually validate and correct extracted data.
* **Target Approach (Extraction Admin as Supervisor):** AI automatically detects recurring patterns and applies constantly adapting extractions based on various quality signals. Humans primarily focus on defining extraction goals and validating high-impact or low-confidence results.
* **Prerequisites:** Scalable communication processing pipeline; Access to better, potentially extraction-specialized LLMs; Framework for defining target extraction schemas; Efficient human-in-the-loop validation and feedback system.

### 5.4 Extracted Data Normalisation & Validation (Semantic Integrity Enforcement)

* **Problem Definition:** Data obtained from external sources, particularly via extraction from unstructured communications, frequently suffers from inconsistencies in format (dates, numbers), units, terminology, and often contains plausible but incorrect values, leading to downstream processing errors if not rigorously checked.
* **Intended Outcome:** Data entering core systems from external sources possesses verified semantic integrity and contextual validity, conforming to internal standards and passing rigorous plausibility checks.
* **Frequency/Scale:** Applied continuously to millions of incoming data points from all external interactions.
* **Intermediate Approach (Extraction Admin as Operator):** Admins implement and maintain explicit data transformation logic and validation rules (range checks, format checks) for known data fields and sources. Requires significant manual effort to cover all variations.
* **Target Approach (Admin as Supervisor):** AI systems automatically assess data quality against learned patterns, contextual plausibility (e.g., comparing extracted prices against market benchmarks or historical values), and internal standards. They automatically correct common inconsistencies and flag only significant anomalies or high-risk validation failures for human attention.
* **Prerequisites:** Internal data standards dictionaries; AI models for data profiling, anomaly detection, and automated correction; Configurable validation rule engine; Workflow for managing validation exceptions.

### 5.5 Provider Notification (Adaptive Outbound Communication)

* **Problem Definition:** Sending necessary communications (e.g., orders, cancellations) to providers is often inefficient and error-prone due to varying channel requirements (API, specific email, portal forms), formatting needs, and lack of reliable delivery confirmation, sometimes requiring manual intervention.
* **Intended Outcome:** Required outbound communications are autonomously generated, formatted, addressed, and verifiably delivered via the optimal channel for each provider and context.
* **Frequency/Scale:** Orchestrating hundreds of thousands of outbound transmissions annually, adapting to evolving provider requirements for submission channels and formats.
* **Intermediate Approach (Provider Admin as Operator):** System uses statically configured templates and primary channels per provider. Delivery failures or format issues require manual investigation and resubmission.
* **Target Approach (Admin as Supervisor):** AI selects optimal delivery channel based on configuration, real-time success rates, and message type. Generative AI crafts contextually appropriate messages from dynamic templates. Automated systems monitor delivery status, handle retries across channels, and parse acknowledgments or error responses.
* **Prerequisites:** Provider communication capability knowledge base; Generative AI for contextual message composition; Multi-channel communication gateway with delivery tracking; Adaptive routing and error handling logic.

## 6. End-User Service Domain

### 6.1 Self-Servicing (Seamless User Agency)

* **Problem Definition:** Users currently lack direct, intuitive digital channels to independently manage many routine aspects of their contracts (e.g., updating details, performing standard actions), forcing them to contact support agents for tasks that could be automated, increasing costs and potentially user friction.
* **Intended Outcome:** Users achieve their goals effortlessly through self-service interfaces that feel proactive, intuitive, and completely comprehensive, resorting to human support only for truly exceptional circumstances.
* **Frequency/Scale:** Aims to become the dominant mode of interaction for millions of users across dozens of use cases.
* **Intermediate Approach (Service Admin as Operator):** A portal or app offers forms and functions for a limited set of explicitly implemented common tasks, linking to predefined backend processes.
* **Target Approach (Admin as Supervisor):** AI-driven interfaces (conversational or graphical) understand user intent in natural language, dynamically generate the necessary interaction flow, autonomously execute backend actions via APIs, and provide context-aware guidance, creating a fluid and comprehensive self-service capability that learns and expands over time.
* **Prerequisites:** Comprehensive service management backend APIs; Secure user authentication; Advanced conversational AI and/or dynamic UI generation platform; Deep integration with user data and context.

### 6.2 Service Interactions (Cognitive Support Augmentation)

* **Problem Definition:** Human support colleagues spend a disproportionate amount of time handling repetitive inquiries, searching for information across disparate systems, and performing routine tasks, limiting their availability for complex problem-solving and relationship building, impacting customer experience.
* **Intended Outcome:** Human colleagues, amplified by AI partners, resolve complex user issues with speed, accuracy, and empathy, focusing their cognitive efforts on complex problem-solving, nuanced judgment, and relationship building.
* **Frequency/Scale:** Augmenting potentially hundreds of thousands of human interactions annually, aiming for order-of-magnitude improvements in agent productivity and customer satisfaction.
* **Intermediate Approach (Service Admin as Operator):** Service colleagues have an summarised interaction-history with the user and AI proactively provides service colleagues with relevant information, suggesting optimal responses or action. Users can interact with chatbots to handle basic topics. Routing is based on simple rules.
* **Target Approach (Admin as Supervisor):** An AI "copilot" continuously analyzes the interaction context (conversation, user data, system state), automating related backend tasks, and summarizing cases, functioning as an intelligent cognitive partner.
* **Prerequisites:** Integrated AI-enabled service workflow; AI models for real-time conversation analysis, knowledge retrieval, and response suggestion; Comprehensive knowledge base accessible via APIs; Backend system integration for task automation; Omni-channel communication platform.

### 6.3 User Notification (Predictive & Prescriptive Engagement)

* **Problem Definition:** Current user notifications are often generic, event-driven, and delivered via static channels, leading to missed opportunities to proactively guide users towards required actions or beneficial outcomes.
* **Intended Outcome:** Users receive a continuous stream of uniquely valuable, context-aware communications that actively anticipate needs and guide users towards better outcomes with minimal perceived noise or interruption, fostering deep engagement and loyalty.
* **Frequency/Scale:** Requires sophisticated orchestration of potentially billions of individualized, context-sensitive micro-communications annually.
* **Intermediate Approach (Service Admin as Operator):** Admins configure event-based triggers and segment-based broadcasts using standardized templates. Basic personalization and channel preference are supported.
* **Target Approach (Admin as Supervisor):** Predictive AI models continuously analyze user states and contexts to identify moments for intervention or information delivery. Generative AI crafts hyper-personalized messages. Reinforcement learning optimizes the selection of content, timing, and channel for each user and communication goal based on maximizing engagement or desired action rates. Human service colleagues focus their time and energy on building true long-term relationships with our users.
* **Prerequisites:** Rich, real-time user activity and context data; Advanced predictive modeling and personalization engines; Multi-channel notification delivery platform; Real-time event streaming architecture; Reinforcement learning framework for communication strategy optimization.

## 7. Growth Domain

### 7.1 Expert Guidance (Generative Knowledge Authority)

* **Problem Definition:** Creating and maintaining a truly comprehensive, accurate, and up-to-date knowledge base covering the complexities of multiple consumer service markets requires immense, often prohibitive, manual research and content creation effort, limiting SwitchUp's ability to establish itself as the definitive expert resource.
* **Intended Outcome:** SwitchUp is the indispensable, AI-powered knowledge resource for consumers navigating service markets, establishing brand authority and organic user acquisition.
* **Frequency/Scale:** Requires continuous generation and maintenance of a vast, interconnected knowledge base and associated content, significantly outpacing traditional manual content creation capabilities.
* **Intermediate Approach (Growth Admin as Operator):** Human experts manually research and instruct AI for content creation, leveraging analytics tools for topic identification.
* **Target Approach (Admin as Supervisor):** AI systems continuously monitor markets, regulations, and internal data, identifying knowledge gaps and emerging trends. Generative AI researches and drafts comprehensive, data-rich content (articles, reports, interactive tools), which human experts review, edit, and augment with unique strategic insights before publication.
* **Prerequisites:** Access to diverse, high-quality data streams (market, regulatory, internal); Advanced generative AI models specialized in analysis and content creation; Human-AI collaborative content creation and curation platforms; Sophisticated content management and knowledge graph infrastructure.

### 7.2 User Onboarding (Autonomous Conversion Synthesis)

* **Problem Definition:** The current static, one-size-fits-all user onboarding process suffers from significant drop-off rates as it fails to adequately address the diverse concerns, motivations, technical hurdles and fears faced by different types of potential users, leading to lost relationship opportunities.
* **Intended Outcome:** The onboarding process adapts fluidly to each individual user, creating a uniquely compelling and efficient path to activation that leverages highest possible conversion rates across all user segments and acquisition channels.
* **Frequency/Scale:** Applies to millions of potential user interactions; requires real-time adaptation and optimization for every session.
* **Intermediate Approach (Growth Admin as Operator):** Humans design and A/B test variations of largely static onboarding funnels using aggregate analytics data to make incremental improvements.
* **Target Approach (Admin as Supervisor):** AI agents manage the end-to-end onboarding experience in real time. They personalize content, interaction patterns, and process steps based on deep, individualized user understanding (source, behavior, inferred intent). Reinforcement learning continuously discovers optimal onboarding strategies for different user profiles, far exceeding human A/B testing capabilities.
* **Prerequisites:** Real-time user behavior and expectation understanding; AI platform for dynamic personalization and journey orchestration; Advanced reinforcement learning framework for optimization; Flexible backend registration and activation APIs.
