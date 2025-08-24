# PLANNING.md - Product Commercial Generator

## Project Overview

**AI Agent Hackathon 2025** - Product Commercial Generator  
**Theme**: "AI Agent が、現実を豊かにする" (AI Agents that enrich reality)  
**Submission Deadline**: September 24, 2025 (23:59 JST)

### Core Concept
Interactive multi-agent AI system that transforms a single product image into a professional commercial video through real-time conversations with three specialized agents.

## Architecture & Goals

### System Architecture
```
User Input (Product Image + Description) 
    ↓
Agent 1: Product Intelligence (Vertex AI Gemini Pro Vision + Chat)
    ↓ 
Agent 2: Creative Director (Gemini + Imagen API)
    ↓
Agent 3: Video Producer (Veo API)
    ↓
Final Commercial Video Output
```

### Agent Responsibilities
- **Agent 1 (Product Intelligence)**: Analyzes uploaded product images using Gemini Pro Vision, then engages in real-time chat to refine product features, target audience, and positioning based on user feedback
- **Agent 2 (Creative Director)**: Uses Gemini for conversation and Imagen API for asset generation. Presents multiple commercial style options, discusses scene structure, and generates visual assets based on interactive user feedback
- **Agent 3 (Video Producer)**: Uses Veo API for video generation and presents narration scripts, audio options, and pacing preferences through chat. Combines all assets into the final commercial based on user's production choices

## Tech Stack & Constraints

### Frontend Stack
- **Framework**: Next.js 14+ with App Router
- **Language**: TypeScript (strict mode)
- **Styling**: Tailwind CSS
- **Real-time**: WebSocket Client for agent chat interactions
- **Forms**: React Hook Form + Zod validation
- **Animations**: Framer Motion
- **Internationalization**: next-intl for Japanese/English support

### Backend Stack  
- **Framework**: Next.js API Routes (App Router)
- **Runtime**: Node.js 18+ (via Next.js)
- **Language**: TypeScript
- **Real-time**: WebSocket API for interactive agent chat
- **File Handling**: Next.js file uploads + Sharp
- **Containerization**: Docker

### Google Cloud Services (REQUIRED)
- **Core APIs**: Vertex AI Gemini Pro Vision, Imagen API, Veo API, Text-to-Speech API
- **Deployment**: Cloud Run (MANDATORY for judging)
- **Storage**: Cloud Storage + Firestore
- **Functions**: Cloud Functions for processing
- **Caching**: Cloud Memorystore (Redis) for rate limiting
- **Monitoring**: Cloud Monitoring + Logging

### Development Tools
- **Testing**: Jest + Testing Library + Playwright
- **Code Quality**: ESLint + Prettier + TypeScript strict
- **CI/CD**: Cloud Build
- **Version Control**: Git with conventional commits

## File Structure & Modularity

### Project Structure
```
/
├── src/
│   ├── app/                 # Next.js App Router
│   │   ├── api/             # API routes (backend functionality)
│   │   │   ├── agents/      # Agent API endpoints
│   │   │   │   ├── product-intelligence/
│   │   │   │   ├── creative-director/
│   │   │   │   └── video-producer/
│   │   │   ├── upload/      # File upload endpoints
│   │   │   └── status/      # Processing status endpoints
│   │   ├── (routes)/        # Frontend pages
│   │   ├── globals.css      # Global styles
│   │   └── layout.tsx       # Root layout
│   ├── components/          # Reusable UI components
│   ├── lib/                 # Utility functions & services
│   │   ├── agents/          # Agent implementations
│   │   │   ├── product-intelligence/
│   │   │   ├── creative-director/
│   │   │   └── video-producer/
│   │   ├── services/        # External API integrations
│   │   ├── database/        # Database utilities
│   │   └── utils/           # Helper functions
│   ├── hooks/               # Custom React hooks
│   ├── types/               # TypeScript type definitions
│   └── i18n/                # Internationalization
│       ├── messages/        # Translation files
│       │   ├── en.json      # English translations
│       │   └── ja.json      # Japanese translations
│       └── config.ts        # i18n configuration
├── functions/               # Cloud Functions (heavy processing)
│   ├── commercial-processing/  # Commercial video generation functions
│   └── media-optimization/     # Product image and asset optimization
├── public/                  # Static assets
├── tests/                   # Test files
│   ├── unit/                # Unit tests
│   ├── integration/         # Integration tests  
│   └── e2e/                 # End-to-end tests
├── infrastructure/          # Pulumi infrastructure as code
│   ├── index.ts             # Main Pulumi program
│   ├── storage.ts           # Cloud Storage buckets
│   ├── compute.ts           # Cloud Run + Cloud Functions
│   ├── database.ts          # Firestore configuration
│   ├── monitoring.ts        # Logging + alerts
│   └── iam.ts              # Service accounts + permissions
├── deploy/                  # Deployment configurations
│   ├── Dockerfile
│   └── cloudbuild.yaml
├── package.json
├── next.config.js
└── tailwind.config.js
```

### Agent Module Structure
Each agent follows consistent patterns in `src/lib/agents/`:
```
lib/agents/product-intelligence/
├── agent.ts                 # Main agent logic
├── tools.ts                 # Agent-specific tools  
├── prompts.ts              # System prompts
├── types.ts                # Agent-specific types
└── index.ts                # Public exports

# Corresponding API route
app/api/agents/product-intelligence/
└── route.ts                # Next.js API route handler
```

### Code Modularity Rules
- **File Size Limit**: Maximum 500 lines per file
- **Single Responsibility**: Each module has one clear purpose
- **Dependency Injection**: Services injected via constructor/parameters
- **Error Boundaries**: Each agent has isolated error handling
- **Relative Imports**: Use relative imports within packages

## Style & Conventions

### TypeScript Standards
- **Strict Mode**: All TypeScript strict flags enabled
- **Type Definitions**: Comprehensive interfaces for all data
- **Error Types**: Custom error classes for different failure modes
- **Validation**: Runtime validation with Zod schemas

### Code Style
- **Formatter**: Prettier with 2-space indentation
- **Linting**: ESLint with strict TypeScript rules
- **Naming**: camelCase for variables, PascalCase for types/components
- **Comments**: JSDoc for all public functions and classes

### Git Conventions
- **Commit Format**: Conventional commits (feat:, fix:, docs:, etc.)
- **Branch Strategy**: Feature branches from main
- **PR Requirements**: Code review + tests passing

### API Design
- **REST Conventions**: Standard HTTP methods and status codes  
- **Error Handling**: Consistent error response format
- **Validation**: Request/response validation with schemas
- **Documentation**: OpenAPI/Swagger documentation

## Critical Requirements & Constraints

### Technical Requirements
- **MUST use Vertex AI Gemini API** (not AI Studio)
- **MUST deploy on Google Cloud** for judging eligibility
- **Budget**: $300 Google Cloud credits (monitor usage closely)
- **Processing Time**: Target < 10 minutes per video
- **Success Rate**: > 90% successful generations

### Performance Targets
- **API Response**: < 2 seconds for status updates
- **File Upload**: < 30 seconds for photo upload
- **Video Generation**: 5-8 minutes total processing time
- **Concurrent Users**: Support 10+ simultaneous sessions

### Cost Optimization
- **Estimated Cost per Commercial**: $1.81-2.01
  - Gemini Pro Vision + Chat: $0.20-0.40
  - Imagen API: $0.10-0.20  
  - Veo API: $1.50-1.40
  - Text-to-Speech (narration): $0.01
- **Budget Monitoring**: Alerts at 50%, 75%, 90% usage
- **Resource Management**: Auto-cleanup of temp files

### Security & Compliance
- **Authentication**: No user registration required (demo app)
- **File Security**: Signed URLs for Cloud Storage access
- **API Keys**: Never expose credentials in frontend
- **Data Privacy**: Automatic deletion of user content after 24 hours

### Rate Limiting Strategy (Demo App)
- **IP-Based Limits**: 1 commercial per IP per hour, max 3 per day
- **Concurrent Processing**: Max 5 users processing simultaneously
- **Session Controls**: 10-minute max chat per agent, 30-second auto-proceed timeouts
- **Cost-Aware Dynamic Limits**: Rate limits adjust based on remaining budget
  - Budget > $200: 1/hour, 3/day per IP
  - Budget > $100: 1/2hours, 2/day per IP  
  - Budget < $100: 1/day per IP, judge whitelist enabled
- **Demo Features**: VIP bypass codes for judges, queue system with wait times
- **Implementation**: Redis (Cloud Memorystore) + Next.js middleware + IP geolocation
- **Budget Protection**: Real-time cost tracking with automatic preservation mode

### Internationalization (i18n) Support
- **Supported Languages**: Japanese (ja) and English (en)
- **Framework**: next-intl for Next.js App Router
- **Language Detection**: Browser locale detection with manual override
- **UI Localization**: Complete interface translation (buttons, labels, messages)
- **Agent Chat**: Multilingual agent conversations
  - Gemini Pro Vision: Analyze images and respond in user's language
  - Creative Director: Present style options in Japanese/English
  - Video Producer: Narration scripts in both languages
- **Content Localization**: 
  - Error messages and validation text
  - Chat prompts and agent responses
  - Commercial style descriptions and options
- **Implementation**:
  - Route-based locale handling (/en/*, /ja/*)
  - JSON translation files with nested keys
  - Type-safe translation keys with TypeScript
  - Fallback to English for missing translations
- **GCP Integration**: Text-to-Speech API supports both languages for narration

## Development Timeline

### Week 1 (Aug 12-18): Foundation
- [ ] Google Cloud project setup and API enablement
- [ ] Development environment configuration
- [ ] Agent 1 (Product Intelligence) implementation
- [ ] Basic frontend with product image upload and WebSocket chat
- [ ] Internationalization setup (next-intl configuration)

### Week 2 (Aug 19-25): Interactive Chat & Visual Processing  
- [ ] Agent 2 (Creative Director) implementation
- [ ] Real-time chat system with WebSocket
- [ ] Imagen API integration for asset generation
- [ ] Multilingual agent chat implementation (Japanese/English)

### Week 3 (Aug 26-Sep 1): Video Generation
- [ ] Agent 3 (Video Producer) implementation  
- [ ] Veo API integration and commercial video composition
- [ ] End-to-end interactive commercial generation demo

### Week 4 (Sep 2-8): Polish & Features
- [ ] UI/UX improvements with full Japanese/English localization
- [ ] Error handling and recovery
- [ ] Feature complete system with bilingual support

### Week 5 (Sep 9-15): Production Ready
- [ ] Performance optimization
- [ ] Load testing and scaling
- [ ] Security audit and fixes

### Week 6 (Sep 16-24): Submission
- [ ] Final polish and bug fixes
- [ ] Demo video creation
- [ ] Zenn article writing
- [ ] Submission preparation

## Testing Strategy

### Unit Testing
- **Coverage Target**: > 80% for core business logic
- **Framework**: Jest with TypeScript support
- **Mocking**: Mock external API calls and file operations
- **Test Structure**: Arrange-Act-Assert pattern

### Integration Testing  
- **API Testing**: Test complete request/response cycles
- **Agent Testing**: Test agent interactions and handoffs
- **Storage Testing**: Verify file upload/download flows

### End-to-End Testing
- **Framework**: Playwright for browser automation
- **Scenarios**: Complete user workflows from upload to video
- **Performance**: Load testing with realistic file sizes

### Test Commands
```bash
npm run test              # Unit tests
npm run test:integration  # Integration tests  
npm run test:e2e         # End-to-end tests
npm run test:coverage    # Coverage report
```

## Deployment & Infrastructure

### Pulumi Infrastructure as Code
- **Language**: TypeScript (consistent with application stack)
- **Provider**: @pulumi/gcp for Google Cloud Platform
- **State Management**: Pulumi Cloud (free tier) or GCS backend
- **Resource Organization**: Modular structure with separate files per service type

### Core GCP Resources
- **Cloud Run Service**: Main Next.js application (mandatory for judging)
- **Cloud Storage Buckets**: User uploads, processed media, temporary files
- **Firestore Database**: User sessions, processing status, metadata storage
- **Cloud Functions**: Heavy processing tasks (video generation, optimization)
- **IAM Service Accounts**: Secure inter-service communication
- **Cloud Build**: Automated CI/CD pipeline integration
- **Cloud Monitoring**: Cost alerts and performance tracking

### Deployment Commands
```bash
# Infrastructure management
pulumi up                    # Deploy/update infrastructure
pulumi preview               # Preview changes before deploy
pulumi destroy               # Tear down resources
pulumi stack ls              # List deployment stacks

# Environment-specific deployments
pulumi up --stack dev        # Development environment
pulumi up --stack staging    # Staging environment  
pulumi up --stack prod       # Production environment
```

### Cloud Run Deployment
- **Container Registry**: Google Container Registry
- **Scaling**: 0 to 10 instances based on demand
- **Memory**: 2GB per instance for media processing
- **CPU**: 2 vCPU for optimal performance
- **Environment Variables**: Managed through Pulumi configuration

### Environment Management
- **Development**: Local development with Cloud SDK + Pulumi dev stack
- **Staging**: Cloud Run staging environment + Pulumi staging stack
- **Production**: Production Cloud Run deployment + Pulumi prod stack
- **Cost Control**: Separate billing alerts per environment

### Monitoring & Observability
- **Logging**: Structured logging with Cloud Logging
- **Metrics**: Custom metrics for processing times and success rates
- **Alerts**: Error rate and budget threshold alerts
- **Tracing**: Request tracing for debugging

## Risk Mitigation

### Technical Risks
- **API Quotas**: Monitor usage, implement exponential backoff
- **Processing Failures**: Robust retry logic and fallback options
- **File Size Limits**: Validate and compress uploads
- **Memory Issues**: Streaming processing for large files

### Timeline Risks
- **Feature Creep**: Strict scope management, MVP-first approach  
- **Integration Issues**: Early integration testing
- **Performance Problems**: Continuous performance monitoring
- **Deployment Issues**: Early and frequent deployments

### Demo Strategy
- **Backup Content**: Pre-generated sample videos
- **Multiple Scenarios**: 2-3 different story types prepared
- **Technical Backup**: Recorded demo video as fallback
- **Practice Runs**: Multiple rehearsals before submission

## Success Metrics

### Technical Metrics
- Processing time < 10 minutes per video
- Success rate > 90%
- User satisfaction score > 4/5
- Cost per generation < $2.00

### Business Metrics
- Judges find the demo compelling
- Clear value proposition demonstrated  
- Technical sophistication appreciated
- Emotional engagement achieved

## Reference Implementation

### GCP TypeScript Guides
Reference the comprehensive guides in `05-gcp-reference/`:
- **Vertex AI**: `もうAPIコール管理で迷わない！Google Vertex AIをTypeScriptで完全制御する次世代AI開発ガイド.md`
- **Imagen**: `もう画像生成で迷わない！GCP ImagenをTypeScriptで完全制御する次世代AI画像生成システム構築ガイド.md`  
- **Veo**: `もう動画生成で迷わない！Google DeepMind VeoをTypeScriptで完全制御する次世代AI動画制作ガイド.md`
- **Cloud Run**: `もうサーバー管理で迷わない！Google Cloud Run をTypeScriptで完全制御する実践ガイド.md`

These guides provide production-ready patterns for:
- Error handling and retry logic
- Cost optimization and monitoring  
- Security best practices
- Performance optimization

## Next Steps

1. **Environment Setup**: Initialize Google Cloud project and enable required APIs
2. **Repository Structure**: Create the folder structure as defined above
3. **Development Environment**: Set up local development with proper tooling
4. **Agent 1 Implementation**: Start with Story Analyzer as the foundation
5. **Continuous Integration**: Set up automated testing and deployment pipelines

---

**Note**: This planning document should be referenced at the start of every development session to ensure consistency with project goals, constraints, and architectural decisions.