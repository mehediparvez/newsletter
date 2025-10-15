# Personalized Investment Newsletter System: Solution Proposal

## Executive Summary

This document outlines both short-term and long-term solutions for developing a personalized newsletter system that matches research updates with members' investment portfolios. The system will send weekly emails to members containing only the research updates relevant to assets in their portfolios.

### Key Requirements
- **Deadline**: October 30, 2025
- **Current User Base**: ~1,000 members
- **Future Scale**: Up to 30,000+ members
- **Data Structure**:
  - Each member has their own Google Sheet with portfolio information
  - Research team aggregates updates in a central sheet
  - Weekly newsletters must contain personalized content
- **Technology Constraints**:
  - Must use Google Sheets for member portfolios
  - Preference for HubSpot for email delivery in short-term

## Short-Term Solution (1,000 Members)

### High-Level Design

The short-term solution leverages Google Sheets, Google Apps Script, and HubSpot to create an MVP that meets the October 30 deadline while handling up to 1,000 members efficiently.

![Short-Term Architecture](https://i.imgur.com/example1.png) *(Diagram placeholder)*

#### Technology Selection Rationale

After evaluating multiple automation options:

1. **Google Apps Script** (Selected): 
   - Native integration with Google Sheets
   - No additional costs or third-party dependencies
   - Familiar to most Google Workspace users
   - Faster implementation timeline

2. **Make (Integromat)**: 
   - More visual workflow builder
   - Better for complex multi-system integrations
   - Monthly subscription costs
   - Additional learning curve

3. **n8n**:
   - Self-hosted option provides better data ownership
   - Open-source with fair-code license
   - Requires server setup and maintenance
   - More complex implementation timeline

Google Apps Script was selected for the short-term solution due to its native integration with Google Sheets and ability to meet the October 30 deadline with the least complexity.

#### Key Components:

1. **Member Portfolio Sheets**: Individual Google Sheets for each member containing their asset lists
2. **Master Portfolio Sheet**: Centralized sheet that aggregates all member data
3. **Research Updates Sheet**: Weekly research data maintained by the research team
4. **Google Apps Script**: Scripts for data synchronization and newsletter generation
5. **HubSpot API**: Email delivery with personalization and unsubscribe tracking

### Low-Level Design

#### 1. Data Structure

**Member Portfolio Sheet**
```
| Member ID | Assets   | Associate | Notes | Interests |
|-----------|----------|-----------|-------|-----------|
| email     | SOL      |           |       | AI        |
|           | ETH      |           |       | Memecoins |
```

**Research Updates Sheet**
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

This column-based structure is efficient because:
1. There are a finite number of cryptocurrencies (~30+)
2. Each member needs only one row instead of multiple rows
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

The Member Registry tab in the Master Sheet keeps track of all individual member sheets:

```
| SheetID                                | MemberEmail       | SheetName       | LastSynced          | SyncStatus |
|---------------------------------------|-------------------|----------------|---------------------|------------|
| 1Abc...XYZ                            | john@example.com  | Portfolio      | 2025-10-15T14:30:00 | SUCCESS    |
| 2Def...123                            | sarah@example.com | Portfolio      | 2025-10-15T14:35:00 | SUCCESS    |
```

This registry enables:
- Dynamic discovery of all member sheets without hardcoding IDs
- Tracking sync status for each member sheet
- Adding new member sheets without modifying code
- Admin interface for managing member sheets and troubleshooting

**C. Apps Script Project Structure**:

The synchronization logic resides in a single Apps Script project that is bound to the Master Portfolio Sheet:

1. **Central Management**: All code is maintained in one place for easier updates and maintenance
2. **Permission Model**: The script runs with the permissions of the Master Portfolio Sheet owner
3. **Member Sheet Registry**: A separate sheet tab in the Master Sheet tracks all member sheets
4. **Dynamic Sheet Access**: No hardcoding of sheet IDs - all member sheets are discovered and tracked in the registry
5. **Cross-Sheet Access**: The script has read access to member sheets and read/write access to the Master Sheet

**D. Member Sheet Registration**:

There are two approaches to handle member sheet registration:

1. **Registration Form**:
```javascript
// Simple web app for registering member sheets
function doGet() {
  const html = HtmlService.createHtmlOutputFromFile('RegistrationForm')
    .setTitle('Register Member Portfolio');
  return html;
}

function registerMemberSheet(memberEmail, sheetUrl) {
  try {
    const sheet = SpreadsheetApp.openByUrl(sheetUrl);
    const sheetId = sheet.getId();
    const sheetName = sheet.getSheets()[0].getName();
    
    // Add to registry
    const masterSheet = SpreadsheetApp.getActiveSpreadsheet();
    const registrySheet = masterSheet.getSheetByName("Member Registry");
    registrySheet.appendRow([
      sheetId,
      memberEmail,
      sheetName,
      new Date().toISOString(),
      "PENDING"
    ]);
    
    // Perform initial sync
    const assets = extractAssets(sheet.getSheets()[0]);
    const portfolioSheet = masterSheet.getSheetByName("Portfolios");
    updateMasterPortfolio(portfolioSheet, memberEmail, assets, sheetId);
    
    // Update status
    const lastRow = registrySheet.getLastRow();
    registrySheet.getRange(lastRow, 5).setValue("SUCCESS");
    
    return { success: true, message: "Member sheet registered successfully" };
  } catch (e) {
    return { success: false, message: "Error: " + e.message };
  }
}
```

2. **Folder Monitor**:
```javascript
// Scheduled function to scan a specific folder for new member sheets
function scanMemberSheetFolder() {
  const folder = DriveApp.getFolderById(MEMBER_FOLDER_ID);
  const files = folder.getFilesByType(MimeType.GOOGLE_SHEETS);
  const registrySheet = SpreadsheetApp.getActiveSpreadsheet().getSheetByName("Member Registry");
  const registeredIds = getRegisteredSheetIds(registrySheet);
  
  while (files.hasNext()) {
    const file = files.next();
    const sheetId = file.getId();
    
    // Skip already registered sheets
    if (registeredIds.includes(sheetId)) continue;
    
    try {
      const sheet = SpreadsheetApp.openById(sheetId);
      const memberEmail = extractMemberEmail(sheet);
      
      if (memberEmail) {
        // Register new sheet
        registrySheet.appendRow([
          sheetId,
          memberEmail,
          sheet.getName(),
          new Date().toISOString(),
          "AUTO-REGISTERED"
        ]);
        
        // Perform initial sync
        const assets = extractAssets(sheet.getSheets()[0]);
        const portfolioSheet = SpreadsheetApp.getActiveSpreadsheet().getSheetByName("Portfolios");
        updateMasterPortfolio(portfolioSheet, memberEmail, assets, sheetId);
      }
    } catch (e) {
      Logger.log(`Error processing sheet ${sheetId}: ${e.message}`);
    }
  }
}
```

**E. Trigger-Based Sync**:

```javascript
// This code resides in the Apps Script project attached to the Master Portfolio Sheet
// Triggered when a registered member sheet is edited
function onMemberSheetEdit(e) {
  const sheet = e.source.getActiveSheet();
  const sheetId = sheet.getId();
  
  // Check if this is a registered member sheet
  const masterSheet = SpreadsheetApp.getActiveSpreadsheet();
  const registrySheet = masterSheet.getSheetByName("Member Registry");
  const registryData = registrySheet.getDataRange().getValues();
  
  let memberEmail = null;
  for (let i = 1; i < registryData.length; i++) {
    if (registryData[i][0] === sheetId) {
      memberEmail = registryData[i][1];
      break;
    }
  }
  
  if (!memberEmail) {
    Logger.log(`Unregistered sheet edited: ${sheetId}`);
    return;
  }
  
  // Get all assets from this sheet
  const assets = extractAssets(sheet);
  
  // Update the master sheet (accessible within the same project)
  const portfolioSheet = masterSheet.getSheetByName("Portfolios");
  updateMasterPortfolio(portfolioSheet, memberEmail, assets, sheetId);
  
  // Update sync timestamp in registry
  for (let i = 1; i < registryData.length; i++) {
    if (registryData[i][0] === sheetId) {
      registrySheet.getRange(i+1, 4).setValue(new Date().toISOString());
      registrySheet.getRange(i+1, 5).setValue("SUCCESS");
      break;
    }
  }
  
  // Log the sync
  Logger.log(`Updated portfolio for ${memberEmail} with ${assets.length} assets`);
}
```

#### 3. Newsletter Generation Process

**A. Weekly Newsletter Workflow**:

1. Get the current week's date column from research sheet
2. Load the Master Portfolio Sheet data into memory
3. Create a fast lookup data structure for research updates

```javascript
// This function runs in the Apps Script project attached to the Master Portfolio Sheet
// Typically triggered weekly by a time-based trigger
// Uses the Member Registry to track delivery status
function generateWeeklyNewsletters() {
  // Get research data for current week
  const researchSheet = SpreadsheetApp.openById(RESEARCH_SHEET_ID).getActiveSheet();
  const researchData = researchSheet.getDataRange().getValues();
  const headers = researchData[0];
  
  // Find the current week's column
  const currentWeek = "Oct 27"; // Replace with dynamic date calculation
  const weekColumnIndex = headers.indexOf(currentWeek);
  
  // Build research lookup map
  const researchMap = {};
  for (let i = 1; i < researchData.length; i++) {
    const asset = researchData[i][0];
    const update = researchData[i][weekColumnIndex] || researchData[i][1]; // Use default if no update
    researchMap[asset] = update;
  }
  
  // Load master portfolio data using the column-based structure
  const masterSheet = SpreadsheetApp.openById(MASTER_SHEET_ID).getSheetByName("Portfolios");
  const masterData = masterSheet.getDataRange().getValues();
  const masterHeaders = masterData[0]; // First row contains asset names as column headers
  
  // Generate and send personalized newsletters
  const BATCH_SIZE = 50;
  
  // Process members in batches (each row is now a complete member portfolio)
  for (let i = 1; i < masterData.length; i += BATCH_SIZE) {
    const batchEnd = Math.min(i + BATCH_SIZE, masterData.length);
    const memberBatch = [];
    
    for (let j = i; j < batchEnd; j++) {
      const memberRow = masterData[j];
      const memberEmail = memberRow[0]; // First column is member email
      
      // Extract assets this member has (value of 1 in the asset's column)
      const memberAssets = [];
      for (let k = 1; k < masterHeaders.length; k++) {
        // Skip metadata columns (LastSynced, SourceSheetURL)
        if (masterHeaders[k] !== "LastSynced" && masterHeaders[k] !== "SourceSheetURL") {
          // If value is 1, member has this asset
          if (memberRow[k] === 1) {
            memberAssets.push(masterHeaders[k]); // Add asset name to member's portfolio
          }
        }
      }
      
      memberBatch.push({email: memberEmail, assets: memberAssets});
    }
    
    processBatch(memberBatch, researchMap);
  }
}
```

**B. Newsletter Content Generation**:

```javascript
function generateNewsletterForMember(email, assets, researchMap) {
  let content = getNewsletterTemplate(); // Get the base template
  
  // Add personalized asset updates
  let assetUpdates = "";
  for (const asset of assets) {
    if (researchMap[asset]) {
      assetUpdates += `<h3>${asset}</h3><p>${researchMap[asset]}</p>`;
    }
  }
  
  content = content.replace("{{MEMBER_EMAIL}}", email)
                   .replace("{{ASSET_UPDATES}}", assetUpdates)
                   .replace("{{DATE}}", new Date().toLocaleDateString());
  
  return content;
}
```

#### 4. HubSpot Integration

```javascript
function sendNewsletterViaHubspot(email, content) {
  const payload = {
    emailId: HUBSPOT_TEMPLATE_ID, // Your HubSpot email template ID
    message: {
      to: email
    },
    contactProperties: [
      { name: "email", value: email }
    ],
    customProperties: {
      newsletter_content: content
    }
  };
  
  try {
    const response = UrlFetchApp.fetch('https://api.hubapi.com/marketing/v4/email/single-send', {
      method: 'post',
      headers: {
        'Authorization': `Bearer ${HUBSPOT_API_KEY}`,
        'Content-Type': 'application/json'
      },
      payload: JSON.stringify(payload)
    });
    
    return JSON.parse(response.getContentText());
  } catch (e) {
    Logger.log(`Error sending to ${email}: ${e.message}`);
    return { error: e.message, email: email };
  }
}
```

#### 5. Error Handling and Retry Mechanism

```javascript
function processBatch(memberBatch, researchMap) {
  const results = [];
  
  // With column-based structure, memberBatch is an array of objects with email and assets properties
  for (const member of memberBatch) {
    const email = member.email;
    const assets = member.assets;
    const content = generateNewsletterForMember(email, assets, researchMap);
    
    try {
      const result = sendNewsletterViaHubspot(email, content);
      results.push({ email: email, status: "success", result: result });
    } catch (e) {
      results.push({ email: email, status: "error", error: e.message });
    }
  }
  
  // Save results to log sheet for tracking and retry
  logResults(results);
  
  // Schedule retry for failed sends
  const failed = results.filter(r => r.status === "error");
  if (failed.length > 0) {
    scheduleRetry(failed);
  }
}
```

### Implementation Timeline

1. **Week 1 (Oct 15-21)**:
   - Create Master Portfolio Sheet structure
   - Develop initial sync script
   - Set up change triggers
   - Test data synchronization for a subset of users

2. **Week 2 (Oct 22-30)**:
   - Complete newsletter generation logic
   - Integrate with HubSpot API
   - Implement error handling and retries
   - Test end-to-end with sample data
   - Deploy for production use

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
   - Limited by the total number of cryptocurrencies (finite set of ~30-100 assets)
   - Each member requires only a single row regardless of portfolio size
   - Minimizes Google Sheets' 5 million cell limit impact
   - Column-based structure allows atomic updates to a single member row, reducing concurrent update conflicts

### Concurrency Handling

A critical challenge in the short-term solution is handling concurrent updates to the Master Portfolio Sheet. Consider a scenario where 100 Google Sheets are modified simultaneously:

#### Potential Issues:
- **Race Conditions**: Multiple Apps Script triggers attempting to update the Master Sheet simultaneously
- **Data Corruption**: One update might overwrite another's changes
- **Quota Exhaustion**: Many simultaneous executions could quickly hit Google Apps Script quotas
- **Timeout Failures**: Complex updates might exceed the 6-minute execution limit

#### Recommended Approach:

1. **Implement a Queue System** (in the Master Sheet Apps Script project):
   ```javascript
   function queueSheetSync(sheetId, memberEmail) {
     // Instead of directly updating the master sheet
     const queueSheet = SpreadsheetApp.openById(QUEUE_SHEET_ID).getSheetByName("SyncQueue");
     queueSheet.appendRow([
       new Date().toISOString(),
       sheetId,
       memberEmail,
       "PENDING"
     ]);
   }
   
   // Separate time-triggered function that runs every 5 minutes
   // This runs in the Master Sheet Apps Script project
   function processSyncQueue() {
     const queueSheet = SpreadsheetApp.openById(QUEUE_SHEET_ID).getSheetByName("SyncQueue");
     const queue = queueSheet.getDataRange().getValues();
     
     // Process pending syncs (10 at a time to avoid timeouts)
     let processed = 0;
     for (let i = 1; i < queue.length && processed < 10; i++) {
       if (queue[i][3] === "PENDING") {
         const sheetId = queue[i][1];
         const memberEmail = queue[i][2];
         
         try {
           syncSheetToMaster(sheetId, memberEmail);
           queueSheet.getRange(i+1, 4).setValue("COMPLETED");
         } catch (e) {
           queueSheet.getRange(i+1, 4).setValue("ERROR: " + e.message);
         }
         
         processed++;
       }
     }
   }
   ```

2. **Locking Mechanism**:
   ```javascript
   function updateMasterPortfolio(masterSheet, memberEmail, assets, sheetId) {
     const lock = LockService.getScriptLock();
     try {
       // Wait up to 30 seconds for other processes to complete
       lock.waitLock(30000);
       
       // Column-based approach for optimized data structure
       // Get the row for this member or create a new one
       let memberRow = findMemberRow(masterSheet, memberEmail);
       if (!memberRow) {
         // Add a new row for this member
         memberRow = masterSheet.getLastRow() + 1;
         masterSheet.getRange(memberRow, 1).setValue(memberEmail);
       }
       
       // Update the asset columns (1=has asset, 0=doesn't have)
       const headerRow = masterSheet.getRange(1, 1, 1, masterSheet.getLastColumn()).getValues()[0];
       for (const asset of assets) {
         const assetCol = headerRow.indexOf(asset);
         if (assetCol > 0) {  // Found the asset column
           masterSheet.getRange(memberRow, assetCol + 1).setValue(1);
         } else {
           // New asset not in the sheet yet - add a new column
           const newCol = masterSheet.getLastColumn() + 1;
           masterSheet.getRange(1, newCol).setValue(asset);
           masterSheet.getRange(memberRow, newCol).setValue(1);
         }
       }
       
       // Update metadata columns
       const lastSyncedCol = headerRow.indexOf("LastSynced") + 1;
       const sourceUrlCol = headerRow.indexOf("SourceSheetURL") + 1;
       masterSheet.getRange(memberRow, lastSyncedCol).setValue(new Date().toISOString());
       masterSheet.getRange(memberRow, sourceUrlCol).setValue(`https://docs.google.com/spreadsheets/d/${sheetId}`);
       
     } catch (e) {
       Logger.log(`Could not obtain lock: ${e.message}`);
       throw new Error("Too many concurrent updates. Please try again later.");
     } finally {
       lock.releaseLock();
     }
   }
   ```

3. **Conflict Resolution Strategy**:
   - Implement timestamp-based last-write-wins
   - Consider maintaining a history of changes for auditing
   - Log detailed information about conflicts for manual resolution if needed

This approach distributes updates over time and ensures that even with many simultaneous sheet modifications, the Master Portfolio Sheet remains consistent and accurate.

## Long-Term Solution (30,000+ Members)

### High-Level Design

The long-term solution transitions from Google Apps Script to a more scalable architecture using BigQuery for data storage and Amazon Pinpoint for email delivery, while still preserving Google Sheets as the input interface. This architecture ensures true data ownership and prepares the system for future AI and LLM integration.

![Long-Term Architecture](https://i.imgur.com/example2.png) *(Diagram placeholder)*

#### Technology Selection Rationale

1. **BigQuery**:
   - Full data ownership with structured SQL access
   - Designed for analytics on large datasets
   - Unlimited scalability for growing member base
   - Strong integration with AI/ML services for future enhancements

2. **Looker Studio**:
   - Direct integration with BigQuery
   - Custom dashboards for monitoring system performance
   - Visual analytics for portfolio trends and newsletter effectiveness
   - Shareable reports for stakeholders

3. **AWS Services vs. All-Google Solution**:
   - AWS Pinpoint offers significantly lower costs at scale than alternatives
   - Hybrid architecture leverages strengths of both platforms
   - Future-proof design with multiple integration options

#### Key Components:

1. **Member Portfolio Sheets**: Remain as input interface for members
2. **Google Apps Script**: Handles sync from sheets to BigQuery
3. **BigQuery**: Central data warehouse for all portfolio and research data
4. **Looker Studio**: Analytics dashboards and reporting
5. **Amazon Pinpoint**: Scalable email delivery service
6. **AWS Lambda**: Serverless functions for data processing and newsletter generation

### Low-Level Design

#### 1. Data Structure in BigQuery

**Members Table**
```sql
CREATE TABLE `project.newsletter.members` (
  member_id STRING,
  email STRING,
  name STRING,
  create_date TIMESTAMP,
  status STRING,
  last_updated TIMESTAMP
);
```

**Portfolios Table**
```sql
CREATE TABLE `project.newsletter.portfolios` (
  portfolio_id STRING,
  member_id STRING,
  asset STRING,
  type STRING,
  source_sheet_url STRING,
  last_synced TIMESTAMP
);
```

**Research Table**
```sql
CREATE TABLE `project.newsletter.research` (
  asset STRING,
  week_date DATE,
  update STRING,
  default_message STRING,
  created_by STRING,
  created_at TIMESTAMP
);
```

**Newsletter Log Table**
```sql
CREATE TABLE `project.newsletter.newsletter_logs` (
  log_id STRING,
  member_id STRING,
  send_date TIMESTAMP,
  status STRING,
  error_message STRING,
  content STRING
);
```

#### 2. Sheet to BigQuery Synchronization

**A. Google Apps Script Sync**:

```javascript
function syncSheetsToBigQuery() {
  // Get all member sheets
  const memberSheets = getAllMemberSheets();
  
  // Process in batches
  const BATCH_SIZE = 100;
  for (let i = 0; i < memberSheets.length; i += BATCH_SIZE) {
    const batch = memberSheets.slice(i, i + BATCH_SIZE);
    processBatchSync(batch);
  }
}

function processBatchSync(sheetBatch) {
  // Extract data from each sheet
  const portfolioData = [];
  
  for (const sheetInfo of sheetBatch) {
    const sheet = SpreadsheetApp.openById(sheetInfo.id).getSheetByName(sheetInfo.sheetName);
    const data = sheet.getDataRange().getValues();
    
    // Extract member ID and assets
    const memberId = extractMemberId(data);
    const assets = extractAssets(data);
    
    // Add to batch data
    for (const asset of assets) {
      portfolioData.push({
        portfolio_id: Utilities.getUuid(),
        member_id: memberId,
        asset: asset,
        type: 'Portfolio',
        source_sheet_url: sheetInfo.url,
        last_synced: new Date().toISOString()
      });
    }
  }
  
  // Send to BigQuery using API
  const payload = {
    rows: portfolioData
  };
  
  // Use BigQuery API to insert data
  UrlFetchApp.fetch(`https://bigquery.googleapis.com/bigquery/v2/projects/${PROJECT_ID}/datasets/${DATASET_ID}/tables/${TABLE_ID}/insertAll`, {
    method: 'post',
    headers: {
      'Authorization': `Bearer ${getAccessToken()}`,
      'Content-Type': 'application/json'
    },
    payload: JSON.stringify(payload)
  });
}
```

**B. Scheduled Syncs and Triggers**:

1. Daily full sync from all sheets to BigQuery
2. Change triggers for real-time updates
3. Error logging and retry mechanisms

**C. Handling Concurrent Updates**:

Unlike the short-term solution, BigQuery is designed to handle high-volume concurrent operations. When 100 or even 1,000 Google Sheets change simultaneously:

1. **Atomicity**: BigQuery's insertAll API maintains atomicity for each batch insertion
2. **Higher Concurrency Limits**: BigQuery can handle thousands of concurrent write operations
3. **No Central Bottleneck**: Each sheet sync operates independently without contention
4. **Streaming Buffer**: BigQuery's streaming buffer absorbs burst traffic
5. **Automatic Retries**: The BigQuery client library implements exponential backoff and retries

Example of resilient BigQuery sync code:

```javascript
function syncSheetToBigQuery(sheetId, sheetData) {
  const retries = 3;
  let attempt = 0;
  
  while (attempt < retries) {
    try {
      // Prepare data as before
      const portfolioData = preparePortfolioData(sheetId, sheetData);
      
      // Use BigQuery insertAll API with automatic retries
      const result = BigQuery.Datasets.Tables.insertAll({
        rows: portfolioData.map(item => ({ json: item }))
      }, PROJECT_ID, DATASET_ID, TABLE_ID);
      
      // Check for partial failures and handle if needed
      if (result.insertErrors && result.insertErrors.length > 0) {
        logPartialFailures(result.insertErrors);
      }
      
      return;
    } catch (e) {
      attempt++;
      if (attempt >= retries) {
        logSyncFailure(sheetId, e.message);
        throw e;
      }
      
      // Exponential backoff
      Utilities.sleep(Math.pow(2, attempt) * 1000);
    }
  }
}
```

#### 3. Newsletter Generation with BigQuery and AWS

**A. Query for Newsletter Generation**:

```sql
-- Get all member portfolios matched with current research
SELECT 
  m.member_id,
  m.email,
  p.asset,
  IFNULL(r.update, r.default_message) as asset_update
FROM 
  `project.newsletter.members` m
JOIN 
  `project.newsletter.portfolios` p ON m.member_id = p.member_id
LEFT JOIN 
  `project.newsletter.research` r ON p.asset = r.asset AND r.week_date = CURRENT_DATE()
WHERE 
  m.status = 'ACTIVE'
ORDER BY 
  m.email, p.asset;
```

**B. AWS Lambda Function for Newsletter Processing**:

```javascript
// AWS Lambda function (Node.js)
exports.handler = async (event) => {
    const { BigQuery } = require('@google-cloud/bigquery');
    const AWS = require('aws-sdk');
    const pinpoint = new AWS.Pinpoint();
    
    // Initialize BigQuery client
    const bigquery = new BigQuery();
    
    // Get current date in YYYY-MM-DD format
    const today = new Date().toISOString().split('T')[0];
    
    // Query to get all newsletter content
    const query = `
      SELECT 
        m.member_id,
        m.email,
        p.asset,
        IFNULL(r.update, r.default_message) as asset_update
      FROM 
        \`${process.env.PROJECT_ID}.newsletter.members\` m
      JOIN 
        \`${process.env.PROJECT_ID}.newsletter.portfolios\` p ON m.member_id = p.member_id
      LEFT JOIN 
        \`${process.env.PROJECT_ID}.newsletter.research\` r ON p.asset = r.asset AND r.week_date = '${today}'
      WHERE 
        m.status = 'ACTIVE'
      ORDER BY 
        m.email, p.asset;
    `;
    
    // Run the query
    const [rows] = await bigquery.query({query});
    
    // Group by member
    const memberUpdates = {};
    for (const row of rows) {
        if (!memberUpdates[row.email]) {
            memberUpdates[row.email] = {
                memberId: row.member_id,
                email: row.email,
                updates: []
            };
        }
        
        memberUpdates[row.email].updates.push({
            asset: row.asset,
            update: row.asset_update
        });
    }
    
    // Send emails via Amazon Pinpoint
    const campaignId = `newsletter-${today}`;
    const applicationId = process.env.PINPOINT_APP_ID;
    
    // Process in batches
    const members = Object.values(memberUpdates);
    const BATCH_SIZE = 100;
    
    for (let i = 0; i < members.length; i += BATCH_SIZE) {
        const batch = members.slice(i, i + BATCH_SIZE);
        await sendBatchEmails(pinpoint, applicationId, campaignId, batch);
    }
    
    return {
        statusCode: 200,
        body: JSON.stringify(`Processed ${members.length} newsletters`)
    };
};

async function sendBatchEmails(pinpoint, applicationId, campaignId, members) {
    for (const member of members) {
        // Generate email content
        const emailContent = generateEmailContent(member);
        
        // Send via Pinpoint
        const params = {
            ApplicationId: applicationId,
            MessageRequest: {
                Addresses: {
                    [member.email]: {
                        ChannelType: 'EMAIL'
                    }
                },
                MessageConfiguration: {
                    EmailMessage: {
                        FromAddress: 'newsletter@example.com',
                        SimpleEmail: {
                            Subject: {
                                Data: `Your Weekly Investment Updates - ${new Date().toLocaleDateString()}`
                            },
                            HtmlPart: {
                                Data: emailContent
                            }
                        }
                    }
                }
            }
        };
        
        try {
            await pinpoint.sendMessages(params).promise();
            
            // Log success to BigQuery
            await logEmailStatus(member.memberId, member.email, 'SENT');
        } catch (error) {
            console.error(`Error sending to ${member.email}:`, error);
            
            // Log error to BigQuery
            await logEmailStatus(member.memberId, member.email, 'ERROR', error.message);
        }
    }
}

function generateEmailContent(member) {
    let content = getEmailTemplate(); // Get base template
    let updatesHtml = '';
    
    for (const update of member.updates) {
        updatesHtml += `
            <div class="asset-update">
                <h3>${update.asset}</h3>
                <p>${update.update}</p>
            </div>
        `;
    }
    
    return content
        .replace('{{MEMBER_EMAIL}}', member.email)
        .replace('{{DATE}}', new Date().toLocaleDateString())
        .replace('{{UPDATES}}', updatesHtml);
}
```

#### 4. Amazon Pinpoint vs HubSpot Cost Comparison

For 30,000 weekly newsletters:

**Amazon Pinpoint**:
- Email sending: 30,000 × $0.0001 = $3/week ≈ $12/month
- Endpoint targeting: 25,000 × $0.0012 = $30/month
- **Total**: ~$42/month (without dashboard)

**HubSpot Marketing Hub Enterprise**:
- Starting at $3,600/month
- Includes other marketing features

#### 5. Migration Strategy

1. **Phase 1: Parallel Systems**
   - Keep short-term solution running
   - Set up BigQuery tables and sync pipeline
   - Test with a subset of users

2. **Phase 2: Gradual Migration**
   - Move data from Master Sheet to BigQuery
   - Implement AWS Lambda functions for processing
   - Test Amazon Pinpoint email delivery

3. **Phase 3: Complete Transition**
   - Move all users to the new system
   - Retain Google Sheets as input interface only
   - Monitor and optimize the pipeline

### Implementation Timeline

1. **Month 1-2**:
   - Set up BigQuery infrastructure
   - Develop data sync from Google Sheets to BigQuery
   - Build initial AWS Lambda functions

2. **Month 3**:
   - Configure Amazon Pinpoint
   - Implement newsletter generation logic
   - Test with a subset of users

3. **Month 4**:
   - Complete migration of all users
   - Set up monitoring and alerting
   - Documentation and training

### Benefits of Long-Term Solution

1. **Scalability**:
   - Can easily handle 30,000+ members
   - Optimized database queries instead of sheet-by-sheet processing

2. **Cost Efficiency**:
   - Significant cost savings with Amazon Pinpoint vs HubSpot at scale
   - Pay-per-use model for most AWS services

3. **Performance**:
   - BigQuery processes complex queries in seconds
   - No timeout limitations as with Google Apps Script

4. **Reliability**:
   - Fault-tolerant architecture with retry mechanisms
   - Comprehensive logging for troubleshooting

5. **Maintainability**:
   - Clearly separated concerns (data storage, processing, delivery)
   - Standard cloud architecture patterns

6. **True Data Ownership**:
   - Full control over data storage, structure, and access
   - No vendor lock-in with standardized SQL data format
   - Ability to migrate or extend to other systems if needed
   - Custom data retention policies and compliance controls

7. **AI/LLM Integration Ready**:
   - Structured data format optimal for machine learning models
   - BigQuery's direct integration with AI services
   - Clean, normalized data structure ready for LLM training or queries
   - Potential for future AI-driven portfolio analysis and personalization

## Conclusion

This document outlines two complementary approaches to the personalized newsletter system:

1. **Short-Term Solution**: Google Sheets + Apps Script + HubSpot for 1,000 members
   - Quick to implement before the October 30 deadline
   - Selected Google Apps Script over Make/n8n for speed and native integration
   - Leverages existing Google Sheets infrastructure
   - Dynamic member sheet registry for easy onboarding
   - Suitable for the current scale of operations
   - Efficient column-based Master Sheet structure

2. **Long-Term Solution**: Google Sheets + BigQuery + Looker Studio + AWS for 30,000+ members
   - Scalable architecture that preserves Google Sheets as input interface
   - True data ownership with BigQuery as the central warehouse
   - Advanced analytics and reporting through Looker Studio
   - Significantly more cost-effective at scale
   - AI/LLM-ready data structure for future enhancements
   - Robust and maintainable for long-term growth

The column-based Master Sheet structure provides several advantages:
- Reduced data redundancy with one row per member
- More efficient concurrent updates with fewer conflicts
- Faster lookups and processing for newsletter generation
- Better scalability within the Google Sheets environment
- Simpler transition path to the long-term BigQuery solution

By implementing the short-term solution now with the optimized data structure and planning for the gradual transition to the long-term architecture, the newsletter system can grow smoothly with the business without disrupting operations or requiring a complete overhaul. The approach balances immediate needs (October 30 deadline) with long-term considerations like data ownership, scalability, and AI-readiness.
