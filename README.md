# System Design Repository

Hi there 👋

This repository is a place where I collect and document different **system design ideas** that I find interesting or useful in real-world software architecture.

Most of these designs are inspired by problems that appear in modern platforms — SaaS products, fintech systems, real‑time collaboration tools, data pipelines, and other distributed systems.

I created this repo mainly to:

- keep notes on architecture patterns I like
- experiment with different system design approaches
- share ideas that might be useful for other engineers or system analysts

Some designs are simplified, some go deeper depending on the topic. The goal isn’t to be perfect, but to explore **how complex systems can be structured and scaled**.

If you're interested in system architecture, preparing for system design interviews, or just enjoy thinking about large systems — feel free to explore the catalog below.

---

### Larger / More Serious Systems

These designs represent **larger real-world platforms** where architecture decisions become more complex.

| #   | System                        | What It Covers                                                 | Category          | Complexity | Link                                                                     |
| --- | ----------------------------- | -------------------------------------------------------------- | ----------------- | ---------- | ------------------------------------------------------------------------ |
| 1   | E-Commerce Platform           | Product catalog, shopping cart, checkout, and order management | Commerce Platform | High       | [Go](./)                                                                 |
| 2   | E-Wallet Platform             | Wallet balances, top-ups, transfers, and transaction records   | FinTech           | High       | [Go](./Larger%20System/08_E-Wallet%20Platform/02-System-Architecture.md) |
| 3   | Social Network Platform       | User graph, feeds, interactions, and content distribution      | Social Platform   | Very High  | Pending                                                                  |
| 4   | Hiring & Recruitment Platform | Job postings, candidate pipeline, interview workflow, hiring   | HR Tech Platform  | Medium     | Pending                                                                  |

## Prompt

Prompt for generating for example business requirement specifications for enterprise SaaS products.

```TEXT
You are a Senior Product Manager and Business Analyst responsible for writing detailed Business Requirement Specifications for enterprise SaaS software products.

Your task is to generate a **complete and highly detailed Requirement Specification**.

You will receive ONLY the **Product Name or Project Name**.

From the product name, intelligently infer the product concept, business domain, and the type of problems the system is designed to solve.

Your responsibility is to define **what the system is, what the system must support, what capabilities it must provide, and what business rules govern it**.

IMPORTANT RULES:

- Focus ONLY on **business requirements and system behavior**
- DO NOT include any of the following:
  - UI design
  - Screen layout
  - API design
  - Database schema
  - Microservices architecture
  - Technical implementation
  - User stories
  - Agile artifacts
- Do NOT reduce the level of detail
- Each section must contain **clear, comprehensive, and explicit explanations**
- The document must read like a **professional requirement specification used in enterprise software projects**

The goal is to clearly describe **WHAT the system must do and WHAT business capabilities it must support.**

---

INPUT

Product / Project Name: [Product Name]  
What It Covers: [What It Covers]

Assume the product is:

• Enterprise SaaS Platform  
• Multi-tenant capable  
• Designed for organizational usage  
• Built to support complex operational workflows  

---

# SECTION 1 — SYSTEM OVERVIEW

Provide a detailed explanation of:

1. What the system is  
2. The primary purpose of the system  
3. The operational environment where the system will be used  
4. The type of organizations or industries that would use this system  
5. The types of operational activities the system supports  
6. The high-level capabilities the system must provide  
7. The scope of the system  

Clearly describe how the system fits into a real business environment.

---

# SECTION 2 — BUSINESS OBJECTIVES

Describe the key business objectives that the system must achieve.

Explain:

• Operational goals the system supports  
• Efficiency improvements expected  
• Process automation goals  
• Transparency or visibility improvements  
• Decision-support capabilities  

Each objective must explain **why the system capability is necessary**.

---

# SECTION 3 — PROBLEMS THE SYSTEM SOLVES

Provide a detailed analysis of the business problems the system addresses.

Include:

• Current operational inefficiencies  
• Lack of visibility or tracking  
• Manual processes that should be automated  
• Communication gaps  
• Risk or compliance problems  
• Scalability limitations  

Explain how the system resolves these issues.

---

# SECTION 4 — SYSTEM CAPABILITIES

Define the **major capabilities** that the system must provide.

Present the capabilities in a **table format** with the following columns:

| Capability Name | Detailed Description | Business Purpose |
| --------------- | -------------------- | ---------------- |

Provide **15–25 major capabilities**.

Each capability must clearly explain:

• What the system must allow organizations to do  
• What operational activities it enables  
• Why the capability is necessary for the business  

Descriptions must be **detailed and operationally meaningful**, not short summaries.

---

# SECTION 5 — DETAILED FUNCTIONAL REQUIREMENTS

Provide **highly detailed functional requirements** describing how the system must behave.

Present all requirements in the following **table format**:

| Requirement ID | Requirement Name | Detailed Requirement Description | Conditions / Triggers | Expected Outcome |
| -------------- | ---------------- | -------------------------------- | --------------------- | ---------------- |

Guidelines:

• Requirement ID must follow the format **FR-001, FR-002, FR-003**  
• Requirement Name should clearly summarize the capability  
• Detailed Requirement Description must explain the system behavior clearly  
• Conditions / Triggers describe when the requirement applies  
• Expected Outcome describes the correct system result  

Examples of requirement categories to include:

• Data creation and management  
• Process tracking  
• Workflow management  
• Approval mechanisms  
• Monitoring and reporting  
• Operational controls  
• Configuration capabilities  
• Data integrity requirements  
• Access restrictions at the business level  

Provide **at least 40–80 detailed requirements**.

Each requirement must be explicit, operationally meaningful, and written with enterprise-level clarity.

---

# SECTION 6 — BUSINESS RULES

Define the business rules governing the system.

These rules describe policies or constraints that control how the system operates.

Examples include:

• Operational policies  
• Approval requirements  
• Assignment logic  
• Process constraints  
• Data validation rules  
• Organizational restrictions  

Each rule must include:

Rule Name  
Description  
Condition under which the rule applies  
Expected behavior  

Provide **15–25 business rules**.

---

# SECTION 7 — OPERATIONAL WORKFLOWS

Describe the **major operational workflows** supported by the system.

Each workflow must include:

Workflow Name  
Purpose of the workflow  
Trigger conditions  
Sequence of operational steps  
Outcome of the workflow  

Explain the process from start to completion.

Provide **5–10 major workflows**.

---

# SECTION 8 — EXCEPTION HANDLING SCENARIOS

Describe possible abnormal or exceptional situations that the system must handle.

Examples:

• Incomplete or invalid data  
• Process interruptions  
• Conflicting actions  
• Unauthorized operations  
• Duplicate activities  
• Process cancellation or reversal  

For each scenario explain:

Scenario description  
Trigger condition  
Expected system behavior  

Provide **10–15 scenarios**.

---

# SECTION 9 — OPERATIONAL CONSTRAINTS

Define important constraints that must be respected during system operation.

Examples:

• Organizational policies  
• Data ownership boundaries  
• Process sequencing limitations  
• Regulatory compliance requirements  
• Operational restrictions  

Explain how these constraints influence system behavior.

---

# SECTION 10 — SUCCESS CRITERIA

Define measurable indicators that demonstrate the system is functioning successfully.

Examples:

• Reduced operational delays  
• Improved task visibility  
• Reduced manual coordination  
• Higher process completion rate  
• Reduced operational errors  

Explain how success will be evaluated.

---

# OUTPUT FORMAT

• Use structured **Markdown headings**  
• Provide **detailed paragraphs** for explanations  
• Use numbered lists where appropriate  
• Use tables where specified  
• Do not shorten explanations  

The document must read like a **formal enterprise requirement specification**.

The output must be **extremely detailed, comprehensive, and immediately usable as a requirement document for software development projects**.

Important in Markdown Code Snippet
```

---

## License

MIT License — free for anyone to use, learn from, modify, and share.

If these designs help you learn something new or inspire your own architecture ideas, that's already a win 🚀
