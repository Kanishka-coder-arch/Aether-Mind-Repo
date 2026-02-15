# AI Content Assistant - Design Document

## System Architecture

### High-Level Architecture

```
┌─────────────────┐
│  React Frontend │ (S3 + CloudFront)
│   (Port 3000)   │
└────────┬────────┘
         │ HTTPS/REST API
         ▼
┌─────────────────────────────┐
│   Node.js + Express.js      │
│   Backend API Server        │
│   (EC2 / App Runner)        │
└────────┬────────────────────┘
         │
    ┌────┴────┬────────┬──────────┐
    ▼         ▼        ▼          ▼
┌─────────┐ ┌────────┐ ┌──────────┐
│ Bedrock │ │   S3   │ │ Supabase │
│(Claude) │ │Content │ │PostgreSQL│
│ AI + NLP│ │Storage │ │ Database │
└─────────┘ └────────┘ └──────────┘
                                        │
                                        ▼
                                   ┌──────────┐
                                   │ Twitter/ │
                                   │LinkedIn  │
                                   │   API    │
                                   └──────────┘
```

### Component Responsibilities

- **React Frontend**: User interface for content operations
- **Node.js Backend**: API server, business logic, orchestration
- **Amazon Bedrock**: AI content generation, sentiment analysis, and entity extraction
- **Amazon S3**: Raw content storage
- **Supabase (PostgreSQL)**: Metadata, users, publishing history
- **Social Media APIs**: Content publishing to platforms

## Detailed Architecture

### Backend API Design (Node.js + Express)

#### API Endpoints

```
POST   /api/auth/login              - User authentication
POST   /api/auth/register           - User registration

POST   /api/content/generate        - Generate content using Bedrock
POST   /api/content/transform       - Transform content format
GET    /api/content                 - List all content
GET    /api/content/:id             - Get specific content
DELETE /api/content/:id             - Delete content

POST   /api/analyze/sentiment       - Analyze sentiment
POST   /api/analyze/entities        - Extract entities

POST   /api/social/connect          - OAuth connect to platform
GET    /api/social/connections      - List connected platforms
POST   /api/social/publish          - Publish to social media
GET    /api/social/history          - Publishing history

GET    /api/dashboard/stats         - Dashboard statistics
```

#### Backend Modules

```
backend/
├── src/
│   ├── routes/
│   │   ├── auth.js
│   │   ├── content.js
│   │   ├── analyze.js
│   │   └── social.js
│   ├── controllers/
│   │   ├── contentController.js
│   │   ├── analyzeController.js
│   │   └── socialController.js
│   ├── services/
│   │   ├── bedrockService.js      # AWS Bedrock integration (AI + NLP)
│   │   ├── s3Service.js           # S3 storage operations
│   │   ├── supabaseService.js     # Database operations
│   │   └── socialService.js       # Twitter/LinkedIn API
│   ├── middleware/
│   │   ├── auth.js                # JWT verification
│   │   └── errorHandler.js
│   ├── config/
│   │   └── aws.js                 # AWS SDK configuration
│   └── server.js
├── package.json
└── .env
```

### Frontend Architecture (React)

#### Component Structure

```
frontend/
├── src/
│   ├── components/
│   │   ├── ContentGenerator.jsx   # Main generation UI
│   │   ├── ContentList.jsx        # Display content list
│   │   ├── ContentEditor.jsx      # Edit/transform content
│   │   ├── SentimentDisplay.jsx   # Show analysis results
│   │   ├── SocialPublisher.jsx    # Publish to platforms
│   │   └── Dashboard.jsx          # Analytics dashboard
│   ├── services/
│   │   └── api.js                 # Axios API client
│   ├── context/
│   │   └── AuthContext.jsx        # User authentication state
│   ├── App.jsx
│   └── main.jsx
├── package.json
└── .env
```

## Workflow Diagrams

### Content Generation Workflow

```
User Input (Topic, Type, Tone)
         ↓
React Frontend (Form Submission)
         ↓
POST /api/content/generate
         ↓
Node.js Backend
         ↓
    ┌────┴────┐
    ▼         ▼
Bedrock API   Supabase
(Generate)    (Save Metadata)
    │             │
    ▼             ▼
  S3 Bucket   content table
(Store Text)  (id, title, s3_key)
    │             │
    └─────┬───────┘
          ▼
    Return Content ID
          ↓
Frontend displays generated content
```

### Content Analysis Workflow

```
User selects content
         ↓
POST /api/analyze/sentiment
         ↓
Node.js Backend
         ↓
    ┌────┴────┐
    ▼         ▼
Bedrock       Supabase
(Analyze      (Update sentiment)
Sentiment +
Entities)
    │             │
    ▼             ▼
Sentiment     content table
+ Entities    (sentiment, key_phrases)
    │             │
    └─────┬───────┘
          ▼
Return analysis results
          ↓
Frontend displays sentiment + entities
```

### Social Media Publishing Workflow

```
User clicks "Publish to Twitter"
         ↓
POST /api/social/publish
         ↓
Node.js Backend
         ↓
Check social_connections table
         ↓
    ┌────┴────┐
    ▼         ▼
Twitter API   Supabase
(Post Tweet)  (Save history)
    │             │
    ▼             ▼
Tweet ID      publish_history table
    │         (content_id, platform, status)
    └─────┬───────┘
          ▼
Return publish status
          ↓
Frontend shows success message
```

## Deployment Architecture

### Infrastructure Setup

```
┌─────────────────────────────────────────────┐
│              AWS Cloud (us-east-1)          │
│                                             │
│  ┌──────────────┐      ┌─────────────────┐ │
│  │  S3 Bucket   │      │  EC2 Instance   │ │
│  │  (Frontend)  │      │  or App Runner  │ │
│  │              │      │                 │ │
│  │ React Build  │      │ Node.js Backend │ │
│  │   (Static)   │      │   (Port 3000)   │ │
│  └──────────────┘      └─────────────────┘ │
│         │                       │           │
│         │                       │           │
│  ┌──────▼───────┐      ┌────────▼────────┐ │
│  │ CloudFront   │      │  Security Group │ │
│  │   (CDN)      │      │  (Port 3000)    │ │
│  └──────────────┘      └─────────────────┘ │
│                                             │
│  ┌──────────────┐      ┌─────────────────┐ │
│  │  S3 Bucket   │      │   CloudWatch    │ │
│  │  (Content)   │      │   (Logs/Metrics)│ │
│  └──────────────┘      └─────────────────┘ │
└─────────────────────────────────────────────┘

┌─────────────────────────────────────────────┐
│         External Services                   │
│                                             │
│  ┌──────────────┐      ┌─────────────────┐ │
│  │  Supabase    │      │  Twitter/X API  │ │
│  │ (PostgreSQL) │      │  LinkedIn API   │ │
│  └──────────────┘      └─────────────────┘ │
└─────────────────────────────────────────────┘
```

### Deployment Steps

#### Backend Deployment (EC2)
1. Launch EC2 t2.micro instance (free tier)
2. Install Node.js 18+
3. Clone repository
4. Install dependencies: `npm install`
5. Set environment variables
6. Start with PM2: `pm2 start server.js`
7. Configure security group (port 3000)

#### Backend Deployment (App Runner - Alternative)
1. Push code to GitHub
2. Create App Runner service
3. Connect to GitHub repository
4. Configure build settings (Node.js)
5. Set environment variables
6. Auto-deploy on push

#### Frontend Deployment (S3)
1. Build React app: `npm run build`
2. Create S3 bucket with static hosting
3. Upload build files to S3
4. Configure bucket policy for public read
5. Optional: Add CloudFront for CDN

#### Database Setup (Supabase)
1. Create Supabase project
2. Run SQL schema from requirements.md
3. Configure Row-Level Security policies
4. Get connection string and API keys
5. Add to backend environment variables

## Scalability Design

### Horizontal Scaling Strategy

#### Backend Scaling
- **EC2 Approach**: Use Application Load Balancer + Auto Scaling Group
  - Min instances: 1
  - Max instances: 3-5
  - Scale on CPU > 70%
  
- **App Runner Approach**: Auto-scales automatically
  - Min instances: 1
  - Max instances: 10
  - Scales based on concurrent requests

#### Database Scaling
- **Supabase**: Managed PostgreSQL with automatic scaling
- **Connection Pooling**: Use Supabase connection pooler
- **Read Replicas**: Available in paid tiers (post-hackathon)

#### Storage Scaling
- **S3**: Unlimited storage, auto-scales
- **CloudFront**: Global CDN for frontend assets

### Performance Optimization

#### Caching Strategy
- **In-Memory Cache**: Node.js cache for frequently accessed data
- **Database Queries**: Index on user_id, created_at, content_type
- **API Responses**: Cache non-AI endpoints (5 minutes TTL)

#### Async Processing
- **Bedrock Calls**: Async/await with timeout handling
- **Sentiment & entity analysis**: Handled via Bedrock asynchronously
- **Social Publishing**: Queue-based (future: use SQS)

### Load Testing Targets
- 50 concurrent users
- 100 requests/minute
- < 3 second response time (non-AI)
- < 30 second response time (AI generation)

## Monitoring & Observability

### CloudWatch Metrics

#### Application Metrics
- API request count
- Response time (p50, p95, p99)
- Error rate (4xx, 5xx)
- Bedrock API latency
- Database query time

#### Infrastructure Metrics
- EC2 CPU utilization
- Memory usage
- Network I/O
- Disk usage

### Logging Strategy

#### Structured Logging
```javascript
// Example log format
{
  timestamp: "2026-02-15T10:30:00Z",
  level: "info",
  service: "content-api",
  endpoint: "/api/content/generate",
  userId: "uuid",
  duration: 2500,
  status: 200
}
```

#### Log Levels
- **ERROR**: Application errors, API failures
- **WARN**: Rate limits, slow queries
- **INFO**: API requests, user actions
- **DEBUG**: Detailed debugging (dev only)

### Alerting
- **CloudWatch Alarms**:
  - Error rate > 5%
  - API latency > 5 seconds
  - EC2 CPU > 80%
  - Failed Bedrock calls > 10/hour

## Security Architecture

### Authentication Flow

```
User Login
    ↓
POST /api/auth/login (email, password)
    ↓
Backend validates credentials
    ↓
Query Supabase users table
    ↓
Generate JWT token (expires in 24h)
    ↓
Return token to frontend
    ↓
Frontend stores in localStorage
    ↓
Include in Authorization header for all requests
```

### Authorization
- **JWT Middleware**: Verify token on protected routes
- **Supabase RLS**: Row-level security policies
  - Users can only access their own content
  - Users can only see their own social connections

### Data Security
- **Encryption in Transit**: HTTPS/TLS 1.2+
- **Encryption at Rest**: 
  - S3 server-side encryption (SSE-S3)
  - Supabase encrypted storage
- **Secrets Management**: Environment variables (no hardcoding)
- **API Keys**: Stored in backend environment only

### Social Media OAuth Flow

```
User clicks "Connect Twitter"
    ↓
Redirect to Twitter OAuth page
    ↓
User authorizes app
    ↓
Twitter redirects with auth code
    ↓
Backend exchanges code for access token
    ↓
Store tokens in social_connections table (encrypted)
    ↓
Return success to frontend
```

## Cost Optimization

### Resource Optimization

#### Compute
- **EC2 t2.micro**: Free tier (750 hours/month)
- **App Runner**: $0.007/vCPU-hour + $0.0008/GB-hour
- Estimated: $5-10/month

#### Storage
- **S3**: $0.023/GB (first 50TB)
- **Supabase**: Free tier (500MB database, 1GB storage)
- Estimated: $1-2/month

#### AI Services
- **Bedrock (Claude 3)**: ~$0.003/1K input tokens, ~$0.015/1K output tokens
- Estimated: $10-20/month (100 generations/day + sentiment analysis)

#### Total Estimated Cost
- **Development/Demo**: $15-25/month
- **Production (100 users)**: $40-80/month

### Cost Monitoring
- **AWS Cost Explorer**: Daily cost tracking
- **Budget Alerts**: Set $50/month budget
- **Tagging**: Tag all resources with "Project: ContentAI"
- **Cleanup**: Delete unused resources after demo

## Technology Stack Summary

### Frontend
- React.js 18+
- Vite (build tool)
- Tailwind CSS
- Axios
- React Router

### Backend
- Node.js 18+
- Express.js
- AWS SDK v3
- Supabase JS Client
- jsonwebtoken (JWT)
- Twitter API v2 SDK
- LinkedIn API SDK

### Infrastructure
- AWS S3 (frontend + content storage)
- AWS EC2 or App Runner (backend)
- AWS Bedrock (AI generation + NLP analysis)
- AWS CloudWatch (monitoring)
- Supabase (PostgreSQL database)

### Development
- VS Code
- Postman
- Git/GitHub
- AWS CLI
- Supabase CLI

---

**Document Version**: 2.0 (Hackathon Edition)  
**Last Updated**: February 15, 2026  
**Architecture**: Node.js + React + Supabase + AWS  
**Deployment**: EC2/App Runner + S3


