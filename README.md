# Personalized Newsletter System: Solution Proposal

## Executive Summary

A system to deliver personalized investment newsletters by matching research updates with members' portfolio assets. Weekly emails will contain only relevant research for each member's investments.

### Overview
Each member has a spreadsheet containing portfolio assets. Research data is maintained in a separate sheet with weekly updates for each asset. A central sheet aggregates member data for processing.

### Key Requirements
- **Deadline**: October 30, 2025
- **Current User Base**: ~1,000 members
- **Future Scale**: Expandable for growth
- **Data Structure**:
  - Member portfolio information in Google Sheets
  - Centralized research updates sheet
  - Personalized weekly newsletter content
- **Technology Constraints**:
  - Google Sheets for data storage
  - HubSpot for email delivery

## Short-Term Solution

### High-Level Design

The short-term solution leverages Google Sheets, Google Apps Script, and HubSpot to create an MVP that meets the deadline while handling up to 1,000 members efficiently.



### Step-by-Step System Workflow

#### First Step: Customer Portfolio Sheet Access
The workflow begins with accessing Customer Portfolio Sheets, which contain each member's investment assets.
- **Implementation**: Dedicated Google Drive folder structure with proper permissions
- **Technical Component**: Folder monitoring system using Google Drive API
- **Relation to Architecture**: Forms the input layer of the system

#### Second Step: Research Sheet Access
In parallel with the first step, the system accesses the Research Sheet maintained by the research team.
- **Implementation**: Centralized Google Sheet with standardized structure
- **Technical Component**: Direct API access to Research Updates Sheet and will cache the data for each week one time so that foe each members we don't need to access the sheet each time. 
- **Key Consideration**: Maintaining a consistent format for processing
- **Relation to Architecture**: Provides the content data for personalization

#### Third Step: Sync Member Portfolio Assets Data
The system synchronizes portfolio data from Customer Sheets into the Master Portfolio Sheet.
- **Implementation**: Hybrid approach combining metadata monitoring and selective triggers
- **Technical Component**: Google Apps Script synchronization engine
- **Key Consideration**: Handling concurrent updates and maintaining data integrity
- **Relation to Architecture**: Core data aggregation layer connecting inputs to processing

#### Fourth Step: Match Data
This critical step matches each member's portfolio assets against relevant research updates.
- **Implementation**: Efficient lookup algorithms with in-memory data structures
- **Technical Component**: will use caching to store the data once and used by for each members assets data during email sending
- **Key Consideration**: Performance optimization for large data volumes
- **Relation to Architecture**: Central processing logic for personalization


#### Sixth Step: Send to HubSpot
The final step involves delivering the personalized content to HubSpot for email distribution.
- **Implementation**: API integration with proper authentication and error handling
- **Technical Component**: HubSpot Single Send API connector which will populate the data in the email templates to send to the users
- **Key Consideration**: Delivery reliability and compliance with rate limits
- **Relation to Architecture**: Output layer that delivers content to members


#### Technology Selection 

Google Apps Script is selected for the short-term solution due to its native integration with Google Sheets and ability to meet the deadline with the least complexity.

#### Key Components:

1. **Member Portfolio Sheets**: Individual Google Sheets for each member containing their asset lists (First Step)
2. **Research Updates Sheet**: Weekly research data maintained by the research team (Second Step)
3. **Master Portfolio Sheet**: Centralized sheet that aggregates all member data (Third Step)
4. **Google Apps Script**: Scripts for data synchronization and newsletter generation (Fourth and Fifth Steps)
5. **HubSpot API**: Email delivery with personalization.

### Low-Level Design

#### 1. Data Structure

**Member Portfolio Sheet**
```
| Member ID | Assets   | Associate | Notes | Interests |
|-----------|----------|-----------|-------|-----------|
| email     | SOL      |           |       | AI        |
|           | ETH      |           |       | Memecoins |
```

**Research Updates Sheet** (Second Step: Research Sheet Access)
```
| Assets | Default Message                        | Oct 13 | Oct 20 | Oct 27    | Nov 3 |
|--------|---------------------------------------|--------|--------|-----------|-------|
| BTC    | Nothing new to report, see YSL Hub... | -      | -      | BTC ETFs... | -     |
| ETH    | Nothing new to report, see YSL Hub... | -      | -      | -         | -     |
| Pendle | Nothing new to report, see YSL Hub... | -      | Pendle... | -         | -     |
```

**Master Portfolio Sheet** (created by sync script)
```
| MemberID           | BTC | ETH | SOL | USDC | DOGE | ... | LastSynced          | SourceSheetURL          |
|-------------------|-----|-----|-----|------|------|-----|---------------------|------------------------|
| john@example.com  | 1   | 1   | 0   | 0    | 1    | ... | 2025-10-15T14:30:00 | https://docs.google... |
| sarah@example.com | 0   | 0   | 1   | 1    | 0    | ... | 2025-10-15T14:30:00 | https://docs.google... |
```

This structure is efficient because:
1. There are a finite number of cryptocurrencies
2. Each member needs only one row for storing assets data
3. Reduces data redundancy and improves lookup efficiency
4. Minimizes concurrency conflicts during simultaneous updates

#### 2. Synchronization Process

**A. Initial Setup**:

1. Create a Master Portfolio Sheet to aggregate all member portfolio data
2. Create a "Member Registry" tab within the Master Sheet to track all member sheets
3. Write an Apps Script project attached to the Master Portfolio Sheet to manage all synchronization
4. Configure the Apps Script to read all member sheets from the registry and populate the master sheet
5. Set up a system for registering new member sheets and adding them to the registry

**B. Member Sheet Registry**:

The Member Registry tab in the Master Sheet keeps track of all individual member sheets with enhanced metadata for scalability:

```
| SheetID        | MemberEmail       | SheetName  | LastSynced          | SyncStatus | CreatedDate         | ModifiedDate        | Priority |
|---------------|-------------------|------------|---------------------|------------|---------------------|---------------------|----------|
| 1Abc...XYZ    | john@example.com  | Portfolio  | 2025-10-15T14:30:00 | SUCCESS    | 2025-10-01T10:00:00 | 2025-10-15T14:30:00 | 1        |
| 2Def...123    | sarah@example.com | Portfolio  | 2025-10-15T14:35:00 | SUCCESS    | 2025-10-05T11:30:00 | 2025-10-15T14:35:00 | 2        |
```

This enhanced registry enables:
- Dynamic discovery of all member sheets without hardcoding IDs
- Tracking sync status for each member sheet
- Timestamp-based incremental processing for scalability
- Prioritization of syncs based on modification time or importance
- Adding new member sheets without modifying code
- Admin interface for managing member sheets and troubleshooting

**C. Apps Script Project Structure**:

A single Apps Script project attached to the Master Portfolio Sheet manages all synchronization functions:

- Centralized codebase with unified permissions
- Utilizes the Member Registry to dynamically access all member sheets
- Maintains read access to member sheets and read/write access to the Master Sheet


**E. Member Sheet Registration - Simple Folder Monitoring**:

The system uses a straightforward folder monitoring approach for collecting new member spreadsheets. This implements the **First Step: Customer Portfolio Sheet Access** in the workflow diagram.

**Folder-Based Member Sheet Discovery:**
1. **Dedicated Google Drive Folder**: 
   - All member spreadsheets will be stored in a central Google Drive folder
   - Basic permission controls ensure only authorized users can add files

2. **Daily Scanning Process**:
   - A scheduled script runs daily to check for new spreadsheets in the folder
   - New files are automatically registered in the Member Registry
   - Once registered, member data becomes available for processing

**F. Simple Synchronization Process**:

This component implements the **Third Step: Sync Portfolio Data** in the workflow diagram. A straightforward daily synchronization process keeps the Master Portfolio Sheet up to date with all member data.

**Daily Sync Process Implementation:**

1. **Scheduled Daily Script**:
   - A Google Apps Script runs once daily on a time-based trigger
   - The script processes both new and existing member spreadsheets

2. **Processing New Members**:
   - Identifies newly added spreadsheets in the designated folder
   - Extracts member email and asset information
   - Adds new members to the Master Portfolio Sheet

3. **Checking for Changes**:
   - For existing members, opens each spreadsheet
   - Reviews the specific sheet containing asset information
   - Updates the Master Portfolio Sheet when changes are detected

4. **Simple Batch Processing**:
   - Processes files in small batches to avoid timeout issues
   - Maintains basic logs of sync activity
   - Flags any errors for manual review

This simplified approach prioritizes reliability and ease of maintenance over real-time updates, which is appropriate for the weekly newsletter use case where instant synchronization is not required.
```

#### 3. Newsletter Generation Process

This process covers the **Fourth Step: Match Data** and **Fifth Step: Generate Personalized Content** in the workflow diagram, where Customer Portfolio data is matched with Research Updates to generate personalized newsletters.

**Implementation Approach:**
1. **Simple Data Matching**:
   - For each member, identify which assets in their portfolio have relevant research updates
   - Prepare only the necessary data for their personalized newsletter

2. **Using HubSpot API**:
   - Use pre-configured email templates set up in HubSpot
   - Send only the dynamic content (matched research updates) to populate these templates
   - Leverage HubSpot's template system for consistent formatting and design

3. **Processing Efficiency**:
   - Process members in manageable batches to avoid timeout issues
   - Use simple caching to improve performance
```

#### 4. HubSpot Integration

This component implements the **Sixth Step: Send to HubSpot** in the workflow diagram, delivering personalized newsletters to members through the HubSpot email API.

**Implementation Approach:**
1. **Simple API Integration**:
   - Use HubSpot's Single Send API to deliver personalized emails
   - Authenticate using API key for secure access
   - Utilize pre-configured email templates in HubSpot

2. **Delivery Process**:
   - Generate personalized email content based on member's portfolio matches
   - Send content and recipient information directly to HubSpot API
   - Track delivery status for monitoring purposes

**Considerations:**
- Batch processing to respect HubSpot's rate limits (10 requests/second)
- Pre-defined templates will be used to ensure consistent formatting
- HubSpot will handle unsubscribe management and email compliance
- Simple retry mechanism for failed deliveries
```

#### 5. Error Handling and Retry Mechanism

This component handles the **Failure Handling** aspects shown as "Pass" or "Fail" decision points throughout the workflow diagram, ensuring reliable operation and recovery from errors at each step.



### Limitations and Considerations

1. **Google Apps Script Execution Limits**:
   - 6-minute timeout per execution
   - Need for batch processing (50-100 members per batch)
   - Daily quota limits on API calls

2. **Synchronization Challenges**:
   - Ensuring all member sheet changes are captured
   - Handling conflicts or corrupted data
   - Concurrent update management (see Concurrency Handling section below)

3. **Email Delivery**:
   - HubSpot Single Send API rate limits
   - Monitoring email deliverability

4. **Data Structure Considerations**:
   - The column-based Master Sheet structure significantly improves efficiency
   - Limited by the total number of cryptocurrencies
   - Each member requires only a single row regardless of portfolio size
   - Minimizes Google Sheets' 5 million cell limit impact
   - Column-based structure allows atomic updates to a single member row, reducing concurrent update conflicts

## Long-Term Solution

### High-Level Design

The long-term solution transitions from Google Apps Script to a more scalable architecture using BigQuery for data storage a, while still preserving Google Sheets as the input interface. This architecture ensures true data ownership and prepares the system for future AI and LLM integration.

#### Technology Selection Rationale

**BigQuery**:
   - Full data ownership with structured SQL access
   - Designed for analytics on large datasets
   - Unlimited scalability for growing member base
   - Strong integration with AI/ML services for future enhancements


#### Key Components:

1. **Member Portfolio Sheets**: Remain as input interface for members
2. **Google Apps Script**: Handles sync from sheets to BigQuery
3. **BigQuery**: Central data warehouse for all portfolio and research data
4. **HubSpot API**: Primary email delivery service for consistency with short-term solution
5. **Amazon Pinpoint (Optional)**: Cost-effective alternative for scaling beyond 30,000+ members

### Low-Level Design

#### 1. Data Structure in BigQuery

The long-term solution uses a properly normalized database structure in BigQuery with four main tables designed for scalability and analytics:

**Members Table Design**:
- Stores core member information (ID, email, name)
- Tracks member status and important timestamps
- Serves as the central reference for all member-related operations
- Optimized for fast lookups and joins

**Portfolios Table Design**:
- Implements a one-to-many relationship with members
- Each row represents one asset in a member's portfolio
- Maintains source information linking back to Google Sheets
- Includes synchronization metadata for tracking freshness

**Research Table Design**:
- Organized by asset and date dimensions
- Stores both weekly updates and default fallback messages
- Includes attribution data for auditing purposes
- Structured for efficient filtering by date ranges

**Newsletter Log Table Design**:
- Comprehensive delivery tracking for each newsletter
- Records success/failure status and error details
- Stores content for compliance and troubleshooting
- Designed for efficient reporting and analytics

#### 2. Sheet to BigQuery Synchronization

**A. Google Apps Script Sync**:

**Implementation Approach:**
1. **Member Sheet Discovery**:
   - Use the Member Registry to identify all portfolio sheets
   - Implement delta detection to identify sheets with changes
   - Group sheets by modification time for efficient processing

2. **Batch Processing Architecture**:
   - Process sheets in configurable batches of ~100 for optimal performance
   - Implement checkpointing to handle large member bases
   - Apply priority processing for newly created or modified sheets
   - Support both full and incremental synchronization modes

3. **Data Extraction Pattern**:
   - Extract member information and asset lists from each sheet
   - Transform Google Sheets data into BigQuery-compatible format
   - Generate unique IDs for new portfolio entries
   - Apply data validation and cleansing rules

4. **BigQuery Integration**:
   - Authenticate via OAuth with appropriate scopes
   - Use BigQuery streaming API for real-time updates
   - Implement batch insertions for efficiency
   - Apply proper error handling and retries

**Optimizations:**
- Implement change detection to sync only modified sheets
- Use BigQuery's streaming buffer for performance
- Apply compression for large data transfers
- Implement connection pooling for API efficiency
- Cache authentication tokens for reduced overhead

**Limitations and Constraints:**
- Google Apps Script's 6-minute execution limit requires batching
- BigQuery API has quotas and rate limits
- Authentication token expiration requires refresh handling
- Large datasets may require multiple synchronization passes
- Network latency can impact real-time synchronization
```

**B. Scheduled Syncs and Triggers**:

**Implementation Approach:**
1. **Multi-Tiered Sync Strategy**:
   - Daily full synchronization during off-peak hours
   - Hourly incremental syncs for recently modified sheets
   - Real-time change triggers for critical updates
   - Manual trigger capability for admin-initiated syncs

2. **Synchronization Workflow**:
   - Scheduled Cloud Functions/Cloud Scheduler jobs
   - Event-based triggers using Google Drive change notifications
   - Webhook integration for custom synchronization events
   - Progressive scanning with continuation tokens

3. **Monitoring and Alerting**:
   - Comprehensive sync logs in BigQuery
   - Performance metrics dashboards in Looker Studio
   - Alert system for sync failures or anomalies
   - Admin notification for critical issues

**Optimizations:**
- Implement differential synchronization to minimize data transfer
- Use persistent checkpoint tracking to handle interruptions
- Apply intelligent scheduling based on usage patterns
- Implement parallel processing for large synchronization jobs

**Limitations and Constraints:**
- Google Drive change notifications have limitations on notification frequency
- Cloud Scheduler has minimum interval constraints
- Event-driven architectures may require additional infrastructure
- Cross-system synchronization adds complexity to monitoring

**C. Handling Concurrent Updates**:

Unlike the short-term solution, BigQuery is designed to handle high-volume concurrent operations. This provides significant advantages when many Google Sheets change simultaneously:

**Implementation Approach:**
1. **Concurrency Management**:
   - Leverage BigQuery's native atomic insertion capabilities
   - Implement idempotent operations to prevent duplicates
   - Use transaction-like patterns for multi-step operations
   - Apply version control for conflict detection

2. **Resilience Strategy**:
   - Implement smart retry logic with exponential backoff
   - Track partial failures for selective reprocessing
   - Apply circuit breakers to prevent cascade failures
   - Use deadletter queues for failed operations

3. **Performance Optimization**:
   - Utilize BigQuery's streaming buffer for high-throughput ingestion
   - Implement batch operations for efficiency
   - Use parallelization for independent synchronization tasks
   - Apply resource pooling for API connections

**Advantages over Short-Term Solution:**
1. **Superior Atomicity**: Each batch insertion is atomic, preventing partial updates
2. **Higher Concurrency**: Handles thousands of concurrent operations with ease
3. **No Central Bottleneck**: Distributed architecture eliminates single points of failure
4. **Burst Handling**: Streaming buffer absorbs traffic spikes efficiently
5. **Built-in Resilience**: Client libraries provide sophisticated retry mechanisms
```

#### 3. Newsletter Generation with BigQuery and AWS

**A. Query for Newsletter Generation**:

**Implementation Approach:**
1. **Data Retrieval Strategy**:
   - Use optimized SQL queries to join member, portfolio, and research data
   - Implement filtering to include only active members
   - Apply fallback logic for missing research updates
   - Structure query results for efficient email generation

2. **Query Optimization Techniques**:
   - Leverage BigQuery's column-oriented storage for performance
   - Apply proper indexing and partitioning strategies
   - Use materialized views for frequently accessed data patterns
   - Implement query caching where appropriate

3. **Data Processing Pipeline**:
   - Execute parameterized queries based on current date
   - Apply data transformation for email-friendly formats
   - Group results by member for personalized content generation
   - Implement data validation and sanitization

**Optimizations:**
- Use BigQuery's parallel processing capabilities
- Apply query cost optimization techniques
- Implement efficient join strategies
- Use BigQuery's query caching mechanisms
- Apply column selection to minimize data transfer

**Limitations and Constraints:**
- Complex queries may have higher computational costs
- Very large result sets require pagination or streaming
- Query performance depends on table optimization
- Cross-region queries may introduce latency

**B. Cloud-Based Newsletter Processing**:

**Implementation Approach:**
1. **Serverless Architecture**:
   - Deploy cloud functions triggered on schedule (Google Cloud Functions or AWS Lambda)
   - Configure appropriate memory and timeout settings
   - Set up monitoring and logging

2. **BigQuery Integration**:
   - Connect to BigQuery using authorized service accounts
   - Retrieve personalized newsletter data for all members
   - Transform data into email-friendly format
   - Organize content by member for batch processing

3. **Email Generation Process**:
   - Group data by member to create personalized content collections
   - Apply HTML templates with consistent branding
   - Insert dynamic content based on portfolio matches

4. **Email Delivery Options**:

   **Primary: HubSpot API Integration**
   - Continue using HubSpot API for consistency with short-term solution
   - Implement batch processing to handle volume within rate limits
   - Leverage existing HubSpot templates and contact management

   **Alternative: Amazon Pinpoint (Cost-Effective Option)**
   - Significant cost savings at scale (approximately $42/month vs $3,600/month for HubSpot)
   - Campaign-based email distribution
   - Comprehensive delivery metrics and analytics

5. **Monitoring and Feedback Loop**:
   - Record detailed delivery metrics in BigQuery
   - Track open rates and engagement
   - Implement automated alerts for delivery issues

**Optimizations:**
- Use connection pooling for API efficiency
- Implement parallel processing for large member bases
- Apply compression for template storage
- Implement caching for frequently used assets

**Limitations and Constraints:**
- Cloud functions have execution time limits
- Memory constraints for large data processing
- Need to manage different API rate limits depending on chosen email service
```


This represents significant potential cost savings at scale, which may be considered as member numbers grow beyond 30,000.

## Conclusion

This document outlines two complementary approaches to the personalized newsletter system:

1. **Short-Term Solution**: Google Sheets + Apps Script + HubSpot 
   - Quick to implement before the October 30 deadline
   - Selected Google Apps Script over  for speed and native integration
   - Leverages existing Google Sheets infrastructure
   - Suitable for the current scale of operations
   - Efficient column-based Master Sheet structure

2. **Long-Term Solution**: Google Sheets + BigQuery + HubSpot 
   - Scalable architecture that preserves Google Sheets as input interface
   - True data ownership with BigQuery as the central warehouse
   - Continued use of HubSpot API with option to use cost-effective alternatives if needed
   - AI/LLM-ready data structure for future enhancements
