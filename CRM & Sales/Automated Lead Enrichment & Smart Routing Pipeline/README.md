# AI Lead Enrichment & Smart CRM Routing Workflow

An AI-powered n8n workflow that automatically validates inbound leads, detects CRM duplicates, enriches company information, calculates lead quality using an AI-powered scoring model, classifies leads into business tiers, routes them to the appropriate CRM or project management platform, and prepares structured notifications for the sales team.

---

## Quick Facts

| Item | Value |
|------|------|
| **Workflow Type** | AI Sales Automation |
| **Primary Purpose** | Lead Validation, Enrichment & Intelligent Routing |
| **Trigger** | Manual Trigger (Mock Data) |
| **AI Pattern** | AI Lead Scoring |
| **Programming** | JavaScript (Code Nodes) |
| **External APIs** | CRM API, Clearbit / ZoomInfo / Apollo (Mock) |
| **Integrations** | Salesforce / HubSpot, Pipedrive / Trello, Google Sheets |
| **Duplicate Detection** | Yes |
| **Lead Classification** | Enterprise • Mid-Market • Low Priority |
| **Development Mode** | Uses mock CRM and enrichment data |

---

# 1. Overview

Sales teams receive hundreds of inbound leads every day from forms, websites, campaigns, and marketing platforms. Unfortunately, not every lead deserves immediate attention. Many are duplicates, incomplete, or have little buying potential.

This workflow automates the entire lead qualification process before a salesperson even reviews the lead.

The workflow begins by receiving a batch of leads and checking whether each company already exists inside the CRM. Duplicate leads are automatically separated and handled independently, preventing redundant records from being created.

Unique leads continue through an enrichment stage where additional business information is retrieved from third-party enrichment providers such as Clearbit, ZoomInfo, or Apollo (mocked in this project). The enriched profile is then analyzed by an AI scoring model that evaluates business potential, company fit, and purchase likelihood.

Based on the calculated score, each lead is classified into one of three business tiers:

- **Enterprise**
- **Mid-Market**
- **Low Priority**

The workflow then routes each qualified lead to the appropriate CRM or sales management platform while simultaneously preparing structured notifications for the sales team.

```
Manual Trigger
      │
      ▼
Mock Lead Data
      │
      ▼
Extract Domain
      │
      ▼
CRM Duplicate Search
      │
      ▼
Duplicate Detector
      ├───────────────┐
      │               │
      ▼               ▼
Duplicates      New Leads
      │               │
Update Sheet     Risk & Fit Scoring
Notify Team             │
                        ▼
              Company Enrichment
                        │
                        ▼
                 Tier Classification
             ┌──────────┼──────────┐
             ▼          ▼          ▼
        Enterprise  Mid-Market  Low Priority
             │          │          │
             └──────Combine Output──────┐
                                        ▼
                               Notification Payload
```

---

# 2. What Problem Does It Solve?

Sales representatives often spend significant time reviewing duplicate submissions, researching companies manually, and deciding which leads deserve immediate attention.

Without automation, multiple problems occur:

- Duplicate companies are repeatedly entered into the CRM.
- Salespeople waste time researching company size and industry.
- High-value prospects are sometimes overlooked.
- Low-quality leads receive the same attention as enterprise opportunities.
- Routing decisions depend on manual judgment.

This workflow eliminates those repetitive tasks by automatically detecting duplicates, enriching company information, evaluating business potential, classifying leads into business tiers, and routing them appropriately before a salesperson becomes involved.

---

# 3. Benefits

- Automatically prevents duplicate CRM entries.
- Eliminates manual company research.
- Standardizes lead qualification across the organization.
- Prioritizes enterprise opportunities immediately.
- Reduces sales response time.
- Improves CRM data quality.
- Creates consistent routing logic.
- Provides scalable lead processing for growing sales teams.
- Easily integrates with existing CRM platforms.

---

# 4. Who Will Use It?

This workflow is ideal for organizations handling large volumes of inbound leads.

Typical users include:

- Sales Teams
- Sales Development Representatives (SDRs)
- Revenue Operations (RevOps)
- Marketing Operations
- CRM Administrators
- Business Development Teams
- Growth Teams

---

# 5. Workflow Breakdown

## `When clicking "Execute workflow"` — Manual Trigger

Starts the workflow during development and testing.

In production, this node could be replaced by:

- Webhook Trigger
- CRM Trigger
- Form Submission
- Schedule Trigger
- API Endpoint

---

## `Test Data` — Code

Generates realistic mock lead data.

Each lead contains information such as:

- Company Name
- Website
- Contact Name
- Industry
- Email
- Revenue
- Employee Count

This allows the workflow to be demonstrated without connecting to production systems.

---

## `Get Domain` — Set

Extracts the company's domain name.

The domain becomes the unique identifier used during duplicate detection and enrichment.

---

## `CRM Search` — HTTP Request

Searches the CRM to determine whether the company already exists.

In production this could connect to:

- Salesforce
- HubSpot
- Microsoft Dynamics
- Zoho CRM

Only companies that do not already exist continue through the qualification pipeline.

---

## `Loop Fixer` — Code

Normalizes the returned CRM search results and ensures each lead continues through the workflow with a consistent data structure.

This helper node simplifies downstream processing and prevents branching issues.

---

## `Duplicate Detector` — Switch

Separates duplicate companies from new leads.

### Duplicate Branch

Existing CRM records are immediately routed for administrative handling.

### New Lead Branch

Unique companies continue into AI qualification.

---

# Duplicate Handling Sub-workflow

## `High Value Duplicate?` — Switch

Not all duplicates should be ignored.

This node determines whether an existing lead has become strategically important.

Examples include:

- Increased company size
- Higher annual revenue
- Executive contact submitted
- Product interest changed

High-value duplicates may still require sales attention.

---

## `Update Sheet Node`

Updates the reporting spreadsheet with duplicate information.

---

## `Message to Team`

Generates an internal notification informing the sales team about an important duplicate lead.

---

## `Update Sheet T3`

Logs standard duplicate records for reporting purposes.

---

# Lead Qualification Sub-workflow

## `The Risk & Fit Scorer` — Code

Calculates an overall lead score.

Factors may include:

- Company Size
- Revenue
- Industry
- Geographic Fit
- Existing Technology Stack
- Buying Potential

This score helps determine overall lead quality.

---

## `Clearbit, ZoomInfo, or Apollo Mock` — HTTP Request

Retrieves additional company information.

The project currently uses mocked enrichment data but is designed to integrate with:

- Clearbit
- ZoomInfo
- Apollo.io

Typical enrichment fields include:

- Employee Count
- Annual Revenue
- Funding
- Industry
- Headquarters
- Technology Stack

---

## `Tier Classifier` — Switch

Determines the final business tier.

Possible outcomes:

### Enterprise

Large organizations requiring immediate sales attention.

---

### Mid-Market

Qualified opportunities suitable for standard sales follow-up.

---

### Low Priority

Small businesses or poorly matched prospects that should enter marketing nurture campaigns.

---

## `Salesforce / HubSpot Node`

Routes Enterprise leads into enterprise CRM pipelines.

---

## `Pipedrive / Trello Node`

Routes Mid-Market opportunities into standard sales workflows.

---

## `Append Sheet T3`

Stores low-priority leads for future marketing campaigns and reporting.

---

## `Combine Output`

Combines all routing outcomes into a single standardized payload.

This simplifies downstream integrations and notification workflows.

---

## `For Notifications`

Formats the final output used for:

- Slack
- Microsoft Teams
- Email
- Discord
- CRM Notifications

This node serves as the final preparation stage before delivery.

---

# Setup Requirements

| Service | Used For | Credential Needed |
|----------|----------|------------------|
| CRM Platform | Duplicate Search | API Token |
| Company Enrichment API | Business Intelligence | API Key |
| Google Sheets | Reporting | OAuth2 |
| Salesforce / HubSpot | Enterprise CRM | OAuth2/API Key |
| Pipedrive / Trello | Opportunity Routing | OAuth2/API Key |

---

# Known Limitations / Next Steps

- Mock lead data should be replaced with real website forms, CRM events, or marketing automation platforms.
- CRM search currently assumes a single CRM source; multi-CRM synchronization could be added.
- Company enrichment is mocked and should be connected to Clearbit, Apollo, or ZoomInfo in production.
- The lead scoring model currently uses predefined logic and could be enhanced with an LLM or machine learning model.
- Duplicate handling could incorporate fuzzy matching for company names and domains.
- Retry logic and centralized error handling should be added for production deployments.

---

# Future Improvements

- AI-generated lead summaries
- Automatic email personalization
- CRM activity history analysis
- Intent data integration
- Buying signal detection
- LinkedIn enrichment
- Predictive revenue scoring
- Automatic meeting scheduling
- Multi-CRM synchronization
- Sales dashboard reporting