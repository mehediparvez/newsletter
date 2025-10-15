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
### System Architecture & Workflow

The solution leverages Google Sheets, Apps Script, and HubSpot to create an MVP that handles up to 1,000 members efficiently.

#### 1. Data Structure

**Member Portfolio Sheet**
```
| Member ID | Assets | Associate | Notes | Interests |
|-----------|--------|-----------|-------|----------|
| email     | SOL    |           |       | AI       |
|           | ETH    |           |       | Memecoins|
```

**Research Updates Sheet**
```
| Assets | Default Message          | Oct 13 | Oct 20 | Oct 27    |
|--------|--------------------------|--------|--------|-----------|
| BTC    | Nothing new to report... | -      | -      | BTC ETFs...|
| ETH    | Nothing new to report... | -      | -      | -         |
```

**Master Portfolio Sheet** (created by sync script)
```
| MemberID        | BTC | ETH | SOL | LastSynced   | SourceSheetURL |
|----------------|-----|-----|-----|-------------|---------------|
| john@example.com| 1   | 1   | 0   | 2025-10-15  | https://docs...|
```

#### 2. System Workflow

**Step 1: Member Portfolio Access**
- Store member spreadsheets in dedicated Google Drive folder
- Daily script scans folder for new and modified files
- Basic permissions ensure data security

**Step 2: Research Data Access**
- Centralized Google Sheet with standardized structure for research updates
- Cache data weekly to optimize performance
- Structured by asset and date dimensions

**Step 3: Portfolio Data Synchronization**
- Daily Apps Script synchronizes member data to Master Portfolio Sheet
- Registry tracks all member sheets with metadata:
  ```
  | SheetID    | MemberEmail     | LastSynced   | Status  |
  |------------|----------------|--------------|---------|
  | 1Abc...XYZ | john@example.com| 2025-10-15   | SUCCESS |
  ```
- Process in batches (50-100 sheets) to avoid timeout issues
- Updates only when changes detected

**Step 4: Data Matching & Content Generation**
- Match each member's assets against research updates
- Generate personalized content based on matching assets
- Use efficient lookup with in-memory caching

**Step 5: HubSpot Integration & Delivery**
- Send personalized content to HubSpot API
- Populate pre-configured email templates
- Process in batches to respect API rate limits

### Limitations and Considerations

1. **Technical Constraints**:
   - Google Apps Script: 6-minute execution timeout
   - Need for batch processing (50-100 members per batch)
   - HubSpot API rate limits

2. **Data Structure Efficiency**:
   - Column-based Master Sheet optimizes for limited asset types
   - Single row per member regardless of portfolio size
   - Enables atomic updates to minimize concurrent conflicts

## Long-Term Solution

### System Architecture

The long-term solution transitions to a more scalable architecture using BigQuery while preserving Google Sheets as the input interface. This prepares the system for future growth and AI integration.

#### Key Components

1. **Data Storage**: BigQuery replaces Google Sheets as the central data warehouse
2. **Data Flow**: Google Sheets → Apps Script → BigQuery → Cloud Functions → Email Service
3. **Email Options**: 
   - HubSpot API (primary): Consistent with short-term solution
   - Amazon Pinpoint (alternative): Cost-effective at scale 

#### 1. Data Structure

**BigQuery Database with Three Key Tables**:

- **Members Table**: Core member data (ID, email, name, status)
- **Portfolios Table**: One-to-many relationship with members, each row = one asset
- **Research Table**: Weekly updates organized by asset and date

#### 2. Synchronization Process

**Sheet to BigQuery Sync**:
- Apps Script extracts member data from Google Sheets
- Multi-tiered sync strategy (daily full sync, hourly incremental updates)

**Advantages over Short-Term Solution**:
- Higher concurrency: Handles thousands of concurrent operations
- Better atomicity: Prevents partial updates
- Scalable architecture: No central bottleneck

#### 3. Newsletter Generation

**Processing Pipeline**:
- SQL queries join member portfolios with research updates
- Cloud Functions transform data into email-friendly format
- Personalized content created based on portfolio matches

**Email Delivery**:
- HubSpot integration maintains consistency with short-term solution
- Amazon Pinpoint provides a more cost-effective alternative as volume grows


## Conclusion

This proposal outlines a pragmatic approach to implementing a personalized newsletter system:

1. **Short-Term Solution** (by Oct 30, 2025):
   - Google Sheets + Apps Script + HubSpot for 1,000 members
   - Quick implementation with familiar tools
   - Column-based Master Sheet for efficient data handling

2. **Long-Term Solution** (future scalability):
   - Google Sheets frontend + BigQuery backend + Cloud Functions
   - Preserves member experience while enabling growth to 30,000+ members
   - Foundation for future AI/ML integration and cost optimization
