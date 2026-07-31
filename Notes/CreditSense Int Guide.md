# CreditSense AI Underwriting System: Interview & Architecture Guide

## 1. Overview
This guide provides a comprehensive breakdown of the architecture, workflows, and logical separation of concerns within the **CreditSense AI underwriting system**. It focuses on how the RAG-based **Compliance Agent** and the Machine Learning-based **Risk Agent** operate in tandem while maintaining strictly distinct responsibilities. This document is optimized for interview preparation, providing realistic scenarios and technical depth.

---

## 2. Knowledge Base and Policy Ingestion

### 2.1 Origin of Policy Documents
The foundational knowledge base for the RAG (Retrieval-Augmented Generation) Compliance Agent consists of sample regulatory and banking policy documents. These simulate real-world constraints such as:
- Anti-Money Laundering (AML) compliance guidelines.
- Reserve Bank of India (RBI) Know Your Customer (KYC) directives.
- Internal bank loan eligibility and underwriting policies.
- Customer Due Diligence (CDD) procedures.

*Interview Tip:* If asked about the authenticity of the documents, clarify: "For demonstration purposes, I utilized curated policy documents modeled strictly after authentic banking compliance rules."

### 2.2 Vector Database Architecture (ChromaDB)
A common misconception is that PDFs are stored directly in the database. In reality, the vector database stores mathematically represented text segments. 
- **Chunking:** Documents are divided into manageable chunks (e.g., 200–500 words).
- **Embeddings:** Each chunk is processed through a Sentence Transformer model to generate a dense vector representation.
- **Metadata:** Essential context is preserved alongside the vector, including:
  - Source document name (e.g., `RBI_KYC_Guidelines.pdf`)
  - Specific section or clause numbers (e.g., `Section 3.2`)
  - Page numbers

---

## 3. The Compliance Agent Workflow

The Compliance Agent's primary role is to ensure regulatory adherence rather than assessing financial risk.

### 3.1 Step-by-Step Inference Process
1. **Query Formulation:** A user submits a loan application, which is formulated into a natural language or structured query.
2. **Embedding Generation:** The Compliance Agent generates a vector embedding of this query.
3. **Semantic Retrieval:** ChromaDB performs a similarity search, returning the most semantically relevant policy chunks.
4. **LLM Evaluation:** A Large Language Model (e.g., Llama 3) analyzes the applicant's data against the retrieved policy chunks.
5. **Decision & Citation:** The LLM outputs a compliance decision backed by specific citations from the source documents.

### 3.2 Practical Example: KYC Verification
**Scenario:** A user applies for a ₹15,00,000 loan with PAN and Aadhaar but lacks an Income Tax Return (ITR).

**Retrieved Policy:** 
> "Income Tax Returns for the previous two financial years are mandatory for loans above ₹10,00,000. Applications missing mandatory KYC documents must be routed to manual review."

**LLM Output:** 
- **Compliance Status:** Manual Review Required
- **Reason:** The applicant has not submitted Income Tax Returns, which are mandatory for loans above ₹10 lakh according to the RBI KYC policy.
- **Source:** `RBI_KYC_Guidelines.pdf`, Section 3.2

---

## 4. Advanced Scenario: Anti-Money Laundering (AML) Detection

The LLM does *not* detect money laundering autonomously. Instead, it evaluates applicant behavior against retrieved AML rules to identify policy violations.

### 4.1 Scenario Analysis
**Applicant Profile:** Earns ₹6,00,000 annually.
**Transaction Behavior:** Three cash deposits of ₹9.8L, ₹9.7L, and ₹9.9L within five days.

**Retrieved AML Policies:** 
- *Rule 1:* Multiple deposits just below the ₹10 lakh reporting threshold suggest structuring (smurfing).
- *Rule 2:* Transactions inconsistent with declared income mandate enhanced due diligence.

### 4.2 Output Generation
The Compliance Agent correlates the deposits with the structuring rule and the income disparity rule.

**LLM Output:**
- **AML Status:** Suspicious (High Risk)
- **Reasons:** Repeated cash deposits below ₹10 lakh threshold; deposits inconsistent with declared annual income; possible structuring to avoid mandatory reporting.
- **Recommendation:** Escalate to Manual AML Investigation.
- **Sources:** AML Policy Section 4.2, Section 6.1

*Interview Tip:* Emphasize that in production systems, AML also involves deterministic rules, pattern matching, and dedicated anomaly detection models. The LLM’s role is to explain and flag based on the RAG policy retrieval.

---

## 5. Separation of Concerns: Risk Agent vs. Compliance Agent

A critical architectural decision in CreditSense is the decoupling of financial risk assessment from regulatory compliance.

### 5.1 The Risk Agent
- **Purpose:** Predicts the probability of default and assigns a risk tier.
- **Mechanism:** Utilizes a predictive Machine Learning model (e.g., Gradient Boosting) trained on historical financial data (credit score, income, Debt-to-Income ratio).
- **Primary Question:** *"Can we trust this customer to repay the loan?"*

### 5.2 The Compliance Agent
- **Purpose:** Ensures legal, regulatory, and policy alignment.
- **Mechanism:** Utilizes RAG (ChromaDB + Llama 3) to verify applications against KYC, AML, and internal policies.
- **Primary Question:** *"Are we legally permitted to approve this loan?"*

### 5.3 Why Decouple?
Banks intentionally separate credit risk and compliance because an applicant can be financially strong but legally ineligible (e.g., high credit score but missing PAN), or financially sound but exhibiting suspicious behavior (e.g., excellent repayment history but engaging in potential money laundering). 
An orchestrator (such as LangGraph) ultimately synthesizes outputs from both agents to render a final, auditable underwriting decision.
