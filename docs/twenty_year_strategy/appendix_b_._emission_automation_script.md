# Appendix B: Emission Automation Script

## Overview

This appendix provides production-ready automation solutions for DetourCoin's daily emission execution. The emission process must run reliably every 24 hours to mint new tokens according to the configured emission rate.

**Key Requirements:**
- Execute daily at consistent time (recommended: 12:00 UTC)
- Handle missed executions (catch-up emissions)
- Comprehensive error handling and alerting
- Secure private key management
- Cost-efficient operation (~$0-10/month)
- Monitoring and logging for transparency

---

## Table of Contents

1. [Architecture Options](#architecture-options)
2. [Option A: AWS Lambda (Recommended)](#option-a-aws-lambda-recommended)
3. [Option B: Self-Hosted Script](#option-b-self-hosted-script)
4. [Option C: Chainlink Automation](#option-c-chainlink-automation)
5. [Monitoring & Alerting](#monitoring--alerting)
6. [Security Best Practices](#security-best-practices)
7. [Troubleshooting Guide](#troubleshooting-guide)

---

## Architecture Options

### Comparison Matrix

| Feature | AWS Lambda | Self-Hosted | Chainlink |
|---------|-----------|-------------|-----------|
| **Cost** | ~$0/month | $5-20/month | ~$5/month |
| **Reliability** | 99.95% | Depends on infra | 99.9%+ |
| **Setup Complexity** | Medium | Low | High |
| **Maintenance** | Minimal | Medium | Minimal |
| **Decentralization** | Centralized | Centralized | Decentralized |
| **Private Key Risk** | AWS KMS | Local file | No key needed |

**Recommendation for Years 1-5:** Start with **AWS Lambda** for reliability and minimal cost. Migrate to **Chainlink Automation** at Year 5 launch for increased decentralization and trustlessness.

---

## Option A: AWS Lambda (Recommended)

### Architecture Overview

```
CloudWatch Events (Cron) 
    ↓
Lambda Function
    ↓
Polygon Network → DetourCoin Contract
    ↓
CloudWatch Logs + SNS Alerts
```

### Benefits
- **Serverless**: No infrastructure to manage
- **Free Tier**: Likely runs entirely on AWS free tier
- **Reliable**: AWS handles uptime and scaling
- **Secure**: KMS for key encryption
- **Monitored**: Built-in CloudWatch integration

---

### Complete Implementation

#### 1. Project Structure

```
detourcoin-emission/
├── src/
│   ├── emission.js          # Main emission logic
│   ├── config.js            # Configuration
│   └── utils/
│       ├── logger.js        # Logging utilities
│       ├── alerts.js        # Alert functions
│       └── metrics.js       # Metrics tracking
├── test/
│   └── emission.test.js     # Unit tests
├── package.json
├── serverless.yml           # Serverless Framework config
└── README.md
```

#### 2. Core Emission Script

**src/emission.js**
```javascript
const { ethers } = require('ethers');
const { DynamoDB } = require('@aws-sdk/client-dynamodb');
const { DynamoDBDocument } = require('@aws-sdk/lib-dynamodb');
const { SNS } = require('@aws-sdk/client-sns');
const logger = require('./utils/logger');
const { sendAlert } = require('./utils/alerts');
const { recordMetrics } = require('./utils/metrics');

// Configuration
const POLYGON_RPC = process.env.POLYGON_RPC_URL;
const CONTRACT_ADDRESS = process.env.DETOURCOIN_CONTRACT_ADDRESS;
const PRIVATE_KEY = process.env.EMISSION_WALLET_PRIVATE_KEY; // Encrypted in KMS
const SNS_TOPIC_ARN = process.env.SNS_ALERT_TOPIC;
const DYNAMODB_TABLE = process.env.EMISSIONS_TABLE;

// Contract ABI (minimal - only what we need)
const DETOURCOIN_ABI = [
  "function emitScheduledTokens() external returns (uint256)",
  "function emissionRate() external view returns (uint256)",
  "function lastEmissionTime() external view returns (uint256)",
  "function totalSupply() external view returns (uint256)",
  "function MAX_SUPPLY() external view returns (uint256)",
  "function daysSinceLastEmission() external view returns (uint256)",
  "function pendingEmission() external view returns (uint256)",
  "event TokensEmitted(address indexed recipient, uint256 amount, uint256 daysElapsed, uint256 newTotalSupply, uint256 timestamp)"
];

// Initialize clients
const provider = new ethers.providers.JsonRpcProvider(POLYGON_RPC);
const wallet = new ethers.Wallet(PRIVATE_KEY, provider);
const contract = new ethers.Contract(CONTRACT_ADDRESS, DETOURCOIN_ABI, wallet);
const dynamoDB = DynamoDBDocument.from(new DynamoDB({}));
const sns = new SNS({});

/**
 * Main Lambda handler
 * Executes daily DetourCoin emission
 */
exports.handler = async (event, context) => {
  const executionId = context.requestId;
  const startTime = Date.now();
  
  logger.info('Emission execution started', {
    executionId,
    timestamp: new Date().toISOString()
  });

  try {
    // Pre-flight checks
    await performPreflightChecks();
    
    // Execute emission
    const result = await executeEmission(executionId);
    
    // Record successful execution
    await recordExecution(executionId, result, 'SUCCESS');
    
    // Record metrics
    await recordMetrics({
      executionId,
      duration: Date.now() - startTime,
      status: 'SUCCESS',
      ...result
    });
    
    logger.info('Emission execution completed successfully', result);
    
    return {
      statusCode: 200,
      body: JSON.stringify({
        success: true,
        executionId,
        ...result
      })
    };
    
  } catch (error) {
    logger.error('Emission execution failed', {
      executionId,
      error: error.message,
      stack: error.stack
    });
    
    // Record failed execution
    await recordExecution(executionId, { error: error.message }, 'FAILED');
    
    // Send alert
    await sendAlert({
      severity: 'HIGH',
      title: 'DetourCoin Emission Failed',
      message: `Execution ${executionId} failed: ${error.message}`,
      timestamp: new Date().toISOString(),
      details: {
        error: error.message,
        stack: error.stack
      }
    });
    
    // Return error but don't throw (allow Lambda to complete)
    return {
      statusCode: 500,
      body: JSON.stringify({
        success: false,
        executionId,
        error: error.message
      })
    };
  }
};

/**
 * Performs pre-flight checks before emission
 */
async function performPreflightChecks() {
  logger.info('Running pre-flight checks...');
  
  // 1. Check wallet balance (need MATIC for gas)
  const balance = await wallet.getBalance();
  const balanceInMatic = ethers.utils.formatEther(balance);
  
  logger.info('Wallet balance', { balance: balanceInMatic, unit: 'MATIC' });
  
  if (balance.lt(ethers.utils.parseEther('0.1'))) {
    throw new Error(`Insufficient MATIC balance: ${balanceInMatic} MATIC. Need at least 0.1 MATIC.`);
  }
  
  // 2. Check network connectivity
  const blockNumber = await provider.getBlockNumber();
  logger.info('Network status', { blockNumber, network: 'Polygon' });
  
  // 3. Check if emission is due
  const daysSinceLastEmission = await contract.daysSinceLastEmission();
  logger.info('Days since last emission', { days: daysSinceLastEmission.toString() });
  
  if (daysSinceLastEmission.eq(0)) {
    throw new Error('Emission not yet due. Less than 24 hours since last emission.');
  }
  
  // 4. Check pending emission amount
  const pendingAmount = await contract.pendingEmission();
  const pendingFormatted = ethers.utils.formatEther(pendingAmount);
  logger.info('Pending emission', { amount: pendingFormatted, unit: 'DTC' });
  
  if (pendingAmount.eq(0)) {
    throw new Error('No pending emission. Supply may be at maximum.');
  }
  
  // 5. Estimate gas cost
  const gasPrice = await provider.getGasPrice();
  const gasPriceGwei = ethers.utils.formatUnits(gasPrice, 'gwei');
  logger.info('Current gas price', { price: gasPriceGwei, unit: 'gwei' });
  
  // All checks passed
  logger.info('Pre-flight checks completed successfully');
}

/**
 * Executes the token emission
 */
async function executeEmission(executionId) {
  logger.info('Executing token emission...');
  
  // Get pre-emission state
  const preSupply = await contract.totalSupply();
  const preSupplyFormatted = ethers.utils.formatEther(preSupply);
  
  // Estimate gas
  const gasEstimate = await contract.estimateGas.emitScheduledTokens();
  const gasPrice = await provider.getGasPrice();
  const estimatedCost = gasEstimate.mul(gasPrice);
  const estimatedCostMatic = ethers.utils.formatEther(estimatedCost);
  
  logger.info('Gas estimation', {
    gasLimit: gasEstimate.toString(),
    gasPrice: ethers.utils.formatUnits(gasPrice, 'gwei'),
    estimatedCost: estimatedCostMatic,
    unit: 'MATIC'
  });
  
  // Execute emission with retry logic
  let tx;
  let retries = 3;
  
  while (retries > 0) {
    try {
      tx = await contract.emitScheduledTokens({
        gasLimit: gasEstimate.mul(120).div(100), // Add 20% buffer
        gasPrice: gasPrice.mul(110).div(100) // Add 10% to ensure inclusion
      });
      
      logger.info('Transaction submitted', {
        hash: tx.hash,
        from: tx.from,
        to: tx.to,
        gasLimit: tx.gasLimit.toString()
      });
      
      break; // Success, exit retry loop
      
    } catch (error) {
      retries--;
      if (retries === 0) throw error;
      
      logger.warn('Transaction failed, retrying...', {
        retriesLeft: retries,
        error: error.message
      });
      
      // Wait 10 seconds before retry
      await new Promise(resolve => setTimeout(resolve, 10000));
      
      // Refresh gas price
      gasPrice = await provider.getGasPrice();
    }
  }
  
  // Wait for confirmation
  logger.info('Waiting for transaction confirmation...');
  const receipt = await tx.wait(2); // Wait for 2 confirmations
  
  logger.info('Transaction confirmed', {
    blockNumber: receipt.blockNumber,
    gasUsed: receipt.gasUsed.toString(),
    status: receipt.status === 1 ? 'SUCCESS' : 'FAILED'
  });
  
  if (receipt.status !== 1) {
    throw new Error('Transaction reverted on-chain');
  }
  
  // Parse emission event
  const emissionEvent = receipt.events?.find(e => e.event === 'TokensEmitted');
  
  if (!emissionEvent) {
    throw new Error('TokensEmitted event not found in transaction receipt');
  }
  
  const {
    amount,
    daysElapsed,
    newTotalSupply
  } = emissionEvent.args;
  
  const amountFormatted = ethers.utils.formatEther(amount);
  const newSupplyFormatted = ethers.utils.formatEther(newTotalSupply);
  const actualCost = receipt.gasUsed.mul(receipt.effectiveGasPrice);
  const actualCostMatic = ethers.utils.formatEther(actualCost);
  
  // Calculate supply change
  const supplyIncrease = newTotalSupply.sub(preSupply);
  const supplyIncreaseFormatted = ethers.utils.formatEther(supplyIncrease);
  
  const result = {
    transactionHash: receipt.transactionHash,
    blockNumber: receipt.blockNumber,
    amountEmitted: amountFormatted,
    daysElapsed: daysElapsed.toString(),
    totalSupply: newSupplyFormatted,
    previousSupply: preSupplyFormatted,
    supplyIncrease: supplyIncreaseFormatted,
    gasUsed: receipt.gasUsed.toString(),
    gasCost: actualCostMatic,
    timestamp: new Date().toISOString()
  };
  
  logger.info('Emission executed successfully', result);
  
  // Send success notification
  await sendAlert({
    severity: 'INFO',
    title: 'DetourCoin Emission Successful',
    message: `Emitted ${amountFormatted} DTC (${daysElapsed} days)`,
    timestamp: new Date().toISOString(),
    details: result
  });
  
  return result;
}

/**
 * Records execution to DynamoDB for audit trail
 */
async function recordExecution(executionId, result, status) {
  try {
    await dynamoDB.put({
      TableName: DYNAMODB_TABLE,
      Item: {
        executionId,
        timestamp: new Date().toISOString(),
        status,
        result: JSON.stringify(result),
        ttl: Math.floor(Date.now() / 1000) + (365 * 24 * 60 * 60) // 1 year retention
      }
    });
    
    logger.info('Execution recorded to DynamoDB', { executionId, status });
  } catch (error) {
    logger.error('Failed to record execution', {
      executionId,
      error: error.message
    });
    // Don't throw - this is non-critical
  }
}
```

---

#### 3. Logging Utility

**src/utils/logger.js**
```javascript
/**
 * Structured logging for CloudWatch
 */
class Logger {
  constructor() {
    this.context = {
      service: 'detourcoin-emission',
      version: process.env.VERSION || '1.0.0'
    };
  }

  info(message, metadata = {}) {
    console.log(JSON.stringify({
      level: 'INFO',
      message,
      ...this.context,
      ...metadata,
      timestamp: new Date().toISOString()
    }));
  }

  warn(message, metadata = {}) {
    console.warn(JSON.stringify({
      level: 'WARN',
      message,
      ...this.context,
      ...metadata,
      timestamp: new Date().toISOString()
    }));
  }

  error(message, metadata = {}) {
    console.error(JSON.stringify({
      level: 'ERROR',
      message,
      ...this.context,
      ...metadata,
      timestamp: new Date().toISOString()
    }));
  }
}

module.exports = new Logger();
```

---

#### 4. Alert Utility

**src/utils/alerts.js**
```javascript
const { SNS } = require('@aws-sdk/client-sns');
const logger = require('./logger');

const sns = new SNS({});
const SNS_TOPIC_ARN = process.env.SNS_ALERT_TOPIC;

/**
 * Sends alert via SNS
 */
async function sendAlert({ severity, title, message, timestamp, details }) {
  try {
    const alertMessage = {
      severity,
      title,
      message,
      timestamp,
      details: JSON.stringify(details, null, 2)
    };

    await sns.publish({
      TopicArn: SNS_TOPIC_ARN,
      Subject: `[${severity}] DetourCoin: ${title}`,
      Message: JSON.stringify(alertMessage, null, 2)
    });

    logger.info('Alert sent', { severity, title });
  } catch (error) {
    logger.error('Failed to send alert', {
      error: error.message,
      severity,
      title
    });
  }
}

module.exports = { sendAlert };
```

---

#### 5. Configuration

**serverless.yml**
```yaml
service: detourcoin-emission

provider:
  name: aws
  runtime: nodejs18.x
  region: us-east-1
  timeout: 300  # 5 minutes max
  memorySize: 256
  
  environment:
    POLYGON_RPC_URL: ${env:POLYGON_RPC_URL}
    DETOURCOIN_CONTRACT_ADDRESS: ${env:DETOURCOIN_CONTRACT_ADDRESS}
    EMISSION_WALLET_PRIVATE_KEY: ${env:EMISSION_WALLET_PRIVATE_KEY}
    SNS_ALERT_TOPIC: !Ref EmissionAlertTopic
    EMISSIONS_TABLE: !Ref EmissionsTable
    VERSION: ${self:custom.version}
  
  iam:
    role:
      statements:
        - Effect: Allow
          Action:
            - dynamodb:PutItem
            - dynamodb:GetItem
            - dynamodb:Query
          Resource: !GetAtt EmissionsTable.Arn
        - Effect: Allow
          Action:
            - sns:Publish
          Resource: !Ref EmissionAlertTopic
        - Effect: Allow
          Action:
            - kms:Decrypt
          Resource: ${env:KMS_KEY_ARN}

functions:
  emitTokens:
    handler: src/emission.handler
    events:
      # Run daily at 12:00 UTC
      - schedule:
          rate: cron(0 12 * * ? *)
          enabled: true
          description: Daily DetourCoin emission
    
    # Retry configuration
    maximumRetryAttempts: 2
    
    # DLQ for failed executions
    onError: !GetAtt EmissionDLQ.Arn

resources:
  Resources:
    # DynamoDB table for execution audit trail
    EmissionsTable:
      Type: AWS::DynamoDB::Table
      Properties:
        TableName: detourcoin-emissions
        BillingMode: PAY_PER_REQUEST
        AttributeDefinitions:
          - AttributeName: executionId
            AttributeType: S
          - AttributeName: timestamp
            AttributeType: S
        KeySchema:
          - AttributeName: executionId
            KeyType: HASH
          - AttributeName: timestamp
            KeyType: RANGE
        TimeToLiveSpecification:
          AttributeName: ttl
          Enabled: true
    
    # SNS topic for alerts
    EmissionAlertTopic:
      Type: AWS::SNS::Topic
      Properties:
        DisplayName: DetourCoin Emission Alerts
        TopicName: detourcoin-emission-alerts
        Subscription:
          - Endpoint: ${env:ALERT_EMAIL}
            Protocol: email
          - Endpoint: ${env:ALERT_PHONE}  # Optional SMS
            Protocol: sms
    
    # Dead Letter Queue for failed executions
    EmissionDLQ:
      Type: AWS::SQS::Queue
      Properties:
        QueueName: detourcoin-emission-dlq
        MessageRetentionPeriod: 1209600  # 14 days
    
    # CloudWatch Dashboard
    EmissionDashboard:
      Type: AWS::CloudWatch::Dashboard
      Properties:
        DashboardName: DetourCoin-Emissions
        DashboardBody: !Sub |
          {
            "widgets": [
              {
                "type": "metric",
                "properties": {
                  "metrics": [
                    ["AWS/Lambda", "Invocations", {"stat": "Sum"}],
                    [".", "Errors", {"stat": "Sum"}],
                    [".", "Duration", {"stat": "Average"}]
                  ],
                  "period": 86400,
                  "stat": "Sum",
                  "region": "${AWS::Region}",
                  "title": "Emission Function Metrics"
                }
              }
            ]
          }

plugins:
  - serverless-offline
  - serverless-dotenv-plugin

custom:
  version: 1.0.0
```

---

#### 6. Deployment

**package.json**
```json
{
  "name": "detourcoin-emission",
  "version": "1.0.0",
  "description": "Automated DetourCoin token emission service",
  "main": "src/emission.js",
  "scripts": {
    "deploy": "serverless deploy",
    "deploy:prod": "serverless deploy --stage prod",
    "invoke": "serverless invoke -f emitTokens",
    "logs": "serverless logs -f emitTokens --tail",
    "test": "jest",
    "lint": "eslint src/"
  },
  "dependencies": {
    "@aws-sdk/client-dynamodb": "^3.400.0",
    "@aws-sdk/client-sns": "^3.400.0",
    "@aws-sdk/lib-dynamodb": "^3.400.0",
    "ethers": "^5.7.2"
  },
  "devDependencies": {
    "serverless": "^3.34.0",
    "serverless-offline": "^13.0.0",
    "serverless-dotenv-plugin": "^6.0.0",
    "jest": "^29.6.0",
    "eslint": "^8.48.0"
  }
}
```

---

#### 7. Setup Instructions

**Step 1: Install Dependencies**
```bash
npm install -g serverless
npm install
```

**Step 2: Configure AWS Credentials**
```bash
aws configure
# Enter your AWS Access Key ID, Secret Access Key, and default region
```

**Step 3: Create .env File**
```bash
# .env
POLYGON_RPC_URL=https://polygon-rpc.com
DETOURCOIN_CONTRACT_ADDRESS=0x...
EMISSION_WALLET_PRIVATE_KEY=0x...  # Will encrypt with KMS
ALERT_EMAIL=joe@cleverdevel.com
ALERT_PHONE=+1234567890  # Optional
KMS_KEY_ARN=arn:aws:kms:...  # Create KMS key first
```

**Step 4: Encrypt Private Key with KMS**
```bash
# Create KMS key
aws kms create-key --description "DetourCoin Emission Wallet"

# Encrypt private key
aws kms encrypt \
  --key-id alias/detourcoin-emission \
  --plaintext "your-private-key-here" \
  --output text \
  --query CiphertextBlob

# Use encrypted value in environment variables
```

**Step 5: Deploy to AWS**
```bash
# Test deployment
serverless deploy --stage dev

# Production deployment
serverless deploy --stage prod
```

**Step 6: Verify Deployment**
```bash
# Test invocation
serverless invoke -f emitTokens --stage prod

# Monitor logs
serverless logs -f emitTokens --stage prod --tail
```

---

### Cost Analysis

**AWS Lambda Costs (Monthly):**
- Function executions: 30 per month
- Duration: ~30 seconds each
- Memory: 256 MB
- **Total: $0.00** (within free tier)

**DynamoDB Costs:**
- Writes: 30 per month
- Reads: ~100 per month
- Storage: < 1 GB
- **Total: $0.00** (within free tier)

**SNS Costs:**
- Email notifications: 30-60 per month
- **Total: $0.00** (within free tier)

**Total Monthly Cost: ~$0.00**

---

## Option B: Self-Hosted Script

For those preferring full control and simpler setup.

### Node.js Script with PM2

**emission-cron.js**
```javascript
const { ethers } = require('ethers');
const cron = require('node-cron');
const winston = require('winston');
const nodemailer = require('nodemailer');
const fs = require('fs').promises;
const path = require('path');

// Configuration
const CONFIG = {
  RPC_URL: process.env.POLYGON_RPC_URL || 'https://polygon-rpc.com',
  CONTRACT_ADDRESS: process.env.DETOURCOIN_CONTRACT || '',
  PRIVATE_KEY: process.env.EMISSION_WALLET_KEY || '',
  CRON_SCHEDULE: '0 12 * * *', // Daily at 12:00 UTC
  LOG_FILE: './logs/emissions.log',
  DATA_FILE: './data/emissions.json',
  ALERT_EMAIL: process.env.ALERT_EMAIL || '',
  SMTP_HOST: process.env.SMTP_HOST || '',
  SMTP_USER: process.env.SMTP_USER || '',
  SMTP_PASS: process.env.SMTP_PASS || ''
};

// Logger setup
const logger = winston.createLogger({
  level: 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.json()
  ),
  transports: [
    new winston.transports.File({ filename: CONFIG.LOG_FILE }),
    new winston.transports.Console({
      format: winston.format.simple()
    })
  ]
});

// Email transporter
const emailTransporter = nodemailer.createTransport({
  host: CONFIG.SMTP_HOST,
  port: 587,
  secure: false,
  auth: {
    user: CONFIG.SMTP_USER,
    pass: CONFIG.SMTP_PASS
  }
});

// Contract ABI (minimal)
const ABI = [
  "function emitScheduledTokens() external returns (uint256)",
  "function daysSinceLastEmission() external view returns (uint256)",
  "function pendingEmission() external view returns (uint256)"
];

// Initialize provider and contract
const provider = new ethers.providers.JsonRpcProvider(CONFIG.RPC_URL);
const wallet = new ethers.Wallet(CONFIG.PRIVATE_KEY, provider);
const contract = new ethers.Contract(CONFIG.CONTRACT_ADDRESS, ABI, wallet);

/**
 * Main emission execution
 */
async function executeEmission() {
  const startTime = Date.now();
  const executionId = `emission-${Date.now()}`;
  
  logger.info('Starting emission execution', { executionId });
  
  try {
    // Check if emission is due
    const daysSince = await contract.daysSinceLastEmission();
    
    if (daysSince.eq(0)) {
      logger.info('Emission not due yet - skipping');
      return { skipped: true, reason: 'Not yet 24 hours' };
    }
    
    // Check pending amount
    const pending = await contract.pendingEmission();
    const pendingFormatted = ethers.utils.formatEther(pending);
    
    logger.info('Pending emission', { amount: pendingFormatted });
    
    // Execute
    const tx = await contract.emitScheduledTokens({
      gasLimit: 200000
    });
    
    logger.info('Transaction submitted', { hash: tx.hash });
    
    const receipt = await tx.wait(2);
    
    const duration = Date.now() - startTime;
    
    const result = {
      executionId,
      transactionHash: receipt.transactionHash,
      blockNumber: receipt.blockNumber,
      gasUsed: receipt.gasUsed.toString(),
      duration,
      timestamp: new Date().toISOString()
    };
    
    // Save to file
    await saveExecution(result);
    
    // Send success email
    await sendEmail({
      subject: 'DetourCoin Emission Successful',
      text: `Emission executed successfully.\n\nDetails:\n${JSON.stringify(result, null, 2)}`
    });
    
    logger.info('Emission completed successfully', result);
    
    return result;
    
  } catch (error) {
    logger.error('Emission failed', {
      executionId,
      error: error.message,
      stack: error.stack
    });
    
    // Send failure email
    await sendEmail({
      subject: '🚨 DetourCoin Emission FAILED',
      text: `Emission failed!\n\nError: ${error.message}\n\nStack:\n${error.stack}`
    });
    
    throw error;
  }
}

/**
 * Saves execution record to JSON file
 */
async function saveExecution(result) {
  try {
    const dataPath = path.resolve(CONFIG.DATA_FILE);
    
    // Read existing data
    let data = [];
    try {
      const existing = await fs.readFile(dataPath, 'utf8');
      data = JSON.parse(existing);
    } catch (e) {
      // File doesn't exist yet
    }
    
    // Append new result
    data.push(result);
    
    // Keep last 365 days only
    data = data.slice(-365);
    
    // Write back
    await fs.writeFile(dataPath, JSON.stringify(data, null, 2));
    
    logger.info('Execution saved to file');
  } catch (error) {
    logger.error('Failed to save execution', { error: error.message });
  }
}

/**
 * Sends email alert
 */
async function sendEmail({ subject, text }) {
  if (!CONFIG.ALERT_EMAIL) {
    logger.warn('No alert email configured');
    return;
  }
  
  try {
    await emailTransporter.sendMail({
      from: CONFIG.SMTP_USER,
      to: CONFIG.ALERT_EMAIL,
      subject: `DetourCoin: ${subject}`,
      text
    });
    
    logger.info('Email sent', { subject });
  } catch (error) {
    logger.error('Failed to send email', { error: error.message });
  }
}

/**
 * Health check function
 */
async function healthCheck() {
  try {
    // Check wallet balance
    const balance = await wallet.getBalance();
    const balanceMatic = ethers.utils.formatEther(balance);
    
    if (balance.lt(ethers.utils.parseEther('0.1'))) {
      await sendEmail({
        subject: '⚠️ Low MATIC Balance Warning',
        text: `Emission wallet balance is low: ${balanceMatic} MATIC\n\nPlease top up soon.`
      });
    }
    
    // Check network connectivity
    await provider.getBlockNumber();
    
    logger.info('Health check passed', { balance: balanceMatic });
  } catch (error) {
    logger.error('Health check failed', { error: error.message });
  }
}

// Schedule emission (daily at 12:00 UTC)
cron.schedule(CONFIG.CRON_SCHEDULE, () => {
  logger.info('Cron triggered');
  executeEmission().catch(error => {
    logger.error('Cron execution failed', { error: error.message });
  });
});

// Schedule health check (every 6 hours)
cron.schedule('0 */6 * * *', () => {
  healthCheck().catch(error => {
    logger.error('Health check failed', { error: error.message });
  });
});

logger.info('Emission service started', {
  schedule: CONFIG.CRON_SCHEDULE,
  contract: CONFIG.CONTRACT_ADDRESS
});

// Keep process alive
process.on('SIGTERM', () => {
  logger.info('SIGTERM received, shutting down gracefully');
  process.exit(0);
});
```

**Setup with PM2:**
```bash
# Install PM2
npm install -g pm2

# Start service
pm2 start emission-cron.js --name detourcoin-emission

# Auto-restart on reboot
pm2 startup
pm2 save

# Monitor
pm2 monit

# View logs
pm2 logs detourcoin-emission
```

---

## Option C: Chainlink Automation

*Coming in separate document due to length*

---

## Monitoring & Alerting

### CloudWatch Dashboard Setup

```javascript
// Create comprehensive dashboard
const dashboard = {
  "widgets": [
    {
      "type": "metric",
      "properties": {
        "metrics": [
          ["AWS/Lambda", "Invocations", {"label": "Total Executions"}],
          [".", "Errors", {"label": "Failed Executions", "color": "#d62728"}],
          [".", "Duration", {"stat": "Average", "label": "Avg Duration"}],
          [".", "ConcurrentExecutions", {"label": "Concurrent Runs"}]
        ],
        "period": 86400,
        "stat": "Sum",
        "region": "us-east-1",
        "title": "Emission Function Health",
        "yAxis": {"left": {"min": 0}}
      }
    },
    {
      "type": "log",
      "properties": {
        "query": `SOURCE '/aws/lambda/detourcoin-emission-prod-emitTokens'
                  | fields @timestamp, @message
                  | filter @message like /ERROR/
                  | sort @timestamp desc
                  | limit 20`,
        "region": "us-east-1",
        "title": "Recent Errors"
      }
    }
  ]
}
```

### Alert Rules

**CloudWatch Alarms:**
```yaml
# serverless.yml additions
resources:
  Resources:
    EmissionFailureAlarm:
      Type: AWS::CloudWatch::Alarm
      Properties:
        AlarmName: DetourCoin-Emission-Failures
        AlarmDescription: Alert when emission fails
        MetricName: Errors
        Namespace: AWS/Lambda
        Statistic: Sum
        Period: 300
        EvaluationPeriods: 1
        Threshold: 1
        ComparisonOperator: GreaterThanThreshold
        AlarmActions:
          - !Ref EmissionAlertTopic
    
    WalletBalanceLowAlarm:
      Type: AWS::CloudWatch::Alarm
      Properties:
        AlarmName: DetourCoin-Low-Balance
        AlarmDescription: Alert when wallet balance is low
        # Custom metric from Lambda
        MetricName: WalletBalance
        Namespace: DetourCoin
        Statistic: Minimum
        Period: 3600
        EvaluationPeriods: 1
        Threshold: 0.1  # 0.1 MATIC
        ComparisonOperator: LessThanThreshold
        AlarmActions:
          - !Ref EmissionAlertTopic
```

---