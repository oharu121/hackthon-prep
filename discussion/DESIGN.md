# Product Commercial Generator - Technical Design Document

## 🎯 Project Overview

**AI Agent Hackathon 2025 Project**  
**Product Commercial Generator** - A multi-agent AI system that transforms a single product image into a professional commercial video using three specialized agents working in sequence.

```
Product Image → Agent 1 → Agent 2 → Agent 3 → Commercial Video
    Input    →Analysis→ Visuals → Production→    Output
```

## 📋 System Architecture

### High-Level Flow
1. **User uploads product image** through web interface
2. **Agent 1 (Product Analyzer)** analyzes product and generates marketing insights
3. **Agent 2 (Creative Director)** creates additional visual assets based on analysis
4. **Agent 3 (Video Producer)** combines all assets into final commercial video
5. **User receives** professional commercial ready for use

### Core Technology Stack
- **Frontend**: React/Next.js, TypeScript, Tailwind CSS
- **Backend**: Node.js, Express.js, Google Cloud Functions
- **Storage**: Google Cloud Storage, Firestore
- **AI APIs**: Vertex AI Gemini, Imagen API, Veo API
- **Deployment**: Google Cloud Run
- **Development**: TypeScript throughout for type safety

## 🔍 Agent 1: Product Analyzer (Vertex AI Gemini)

### Responsibilities
- Analyze uploaded product image for marketing insights
- Extract product features, benefits, and positioning
- Determine target audience and brand strategy
- Generate commercial concept and visual style guide

### Technical Implementation
```typescript
interface ProductAnalysis {
  productType: string;
  category: string;
  features: string[];
  benefits: string[];
  targetAudience: {
    demographics: string[];
    psychographics: string[];
    useCases: string[];
  };
  brandPosition: {
    pricePoint: 'budget' | 'mid-range' | 'premium';
    lifestyle: string[];
    emotionalTriggers: string[];
  };
  visualStyle: {
    colorPalette: string[];
    mood: string;
    environment: string[];
  };
  commercialConcept: {
    tone: string;
    messaging: string[];
    callToAction: string;
  };
}
```

### Processing Details
- **Model**: Vertex AI Gemini Pro Vision
- **Processing Time**: ~2-3 seconds
- **Input**: Product image + optional description
- **Output**: Structured JSON analysis object
- **Cost**: ~$0.002 per analysis

## 🎨 Agent 2: Creative Director (Imagen API)

### Responsibilities
- Generate additional visual assets based on Agent 1's analysis
- Create lifestyle scenes showing product in use
- Generate background environments and supporting visuals
- Ensure visual consistency across all generated assets

### Asset Generation Strategy
```typescript
interface AssetCollection {
  lifestyleScenes: {
    url: string;
    description: string;
    aspectRatio: '16:9' | '1:1' | '9:16';
  }[];
  backgrounds: {
    url: string;
    style: string;
    mood: string;
  }[];
  detailShots: {
    url: string;
    focus: string;
  }[];
  brandingElements: {
    url: string;
    type: 'logo' | 'pattern' | 'texture';
  }[];
}
```

### Processing Details
- **Model**: Imagen 2/3 (latest available)
- **Processing Time**: ~30-45 seconds for full collection
- **Assets Generated**: 3-5 lifestyle scenes, 2-3 backgrounds, 2-3 detail shots
- **Quality Control**: Automated filtering for brand appropriateness
- **Cost**: ~$0.02 per image (5 images average = $0.10)

## 🎬 Agent 3: Video Producer (Veo API)

### Responsibilities
- Combine all assets into cohesive commercial video
- Add text overlays, transitions, and motion graphics
- Control pacing and commercial structure
- Generate multiple format outputs (15s/30s/60s)

### Video Structure Template
```
0:00-0:03  Product Hero Shot + Brand Intro
0:03-0:12  Lifestyle Scenes + Feature Highlights  
0:12-0:18  Product Benefits + Social Proof
0:18-0:30  Call to Action + Contact Info
```

### Technical Implementation
```typescript
interface CommercialOutput {
  duration: 15 | 30 | 60; // seconds
  format: 'MP4';
  resolution: '1080p' | '4K';
  aspectRatio: '16:9' | '1:1' | '9:16';
  hasAudio: boolean;
  hasCaptions: boolean;
  brandingLevel: 'minimal' | 'moderate' | 'heavy';
  downloadUrl: string;
  thumbnailUrl: string;
}
```

### Processing Details
- **Model**: Veo 2 (latest video generation)
- **Processing Time**: ~3-5 minutes for 30s commercial
- **Template System**: Pre-designed commercial structures
- **Quality Tiers**: Express (faster) vs Premium (higher quality)
- **Cost**: ~$1.50 per 30s video

## 🏗️ System Architecture

### Backend Infrastructure
```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Cloud Run     │    │ Cloud Functions │    │ Cloud Storage   │
│   (Main API)    │◄──►│  (AI Agents)    │◄──►│  (Media Files)  │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │                       │
         └───────────────────────┼───────────────────────┘
                                 ▼
                    ┌─────────────────┐
                    │   Firestore     │
                    │  (Metadata)     │
                    └─────────────────┘
```

### API Endpoints
```typescript
// Main application endpoints
POST /api/commercial/generate     // Start commercial generation
GET  /api/commercial/:id/status   // Check generation progress
GET  /api/commercial/:id/download // Download completed commercial
DELETE /api/commercial/:id        // Delete commercial and assets

// Individual agent endpoints (internal)
POST /api/agents/analyzer        // Agent 1: Product analysis
POST /api/agents/creative        // Agent 2: Asset generation
POST /api/agents/producer        // Agent 3: Video production
```

### Database Schema (Firestore)
```typescript
interface CommercialJob {
  id: string;
  userId?: string;
  status: 'queued' | 'analyzing' | 'generating_assets' | 'producing_video' | 'completed' | 'failed';
  createdAt: Timestamp;
  updatedAt: Timestamp;
  
  // Input
  originalImageUrl: string;
  productDescription?: string;
  preferences: {
    duration: 15 | 30 | 60;
    style: 'corporate' | 'lifestyle' | 'energetic' | 'minimal';
    aspectRatio: '16:9' | '1:1' | '9:16';
  };
  
  // Agent outputs
  analysis?: ProductAnalysis;
  assets?: AssetCollection;
  finalVideo?: CommercialOutput;
  
  // Processing metadata
  processingTimeMs: number;
  costUsd: number;
  errorMessage?: string;
}
```

## 🚀 Implementation Plan

### Phase 1: Foundation (Week 1)
1. **Google Cloud Setup**
   - Enable Vertex AI, Imagen, Veo APIs
   - Set up service accounts and authentication
   - Configure Cloud Storage buckets

2. **Basic Infrastructure**
   - Node.js/Express backend with TypeScript
   - React/Next.js frontend setup
   - Basic file upload interface
   - Cloud Functions deployment pipeline

3. **Agent 1 Implementation**
   - Vertex AI Gemini client setup
   - Product analysis prompt engineering
   - Image processing and analysis pipeline
   - Structured output validation

### Phase 2: Visual Generation (Week 2)
1. **Agent 2 Development**
   - Imagen API integration
   - Asset generation pipeline
   - Style consistency framework
   - Batch processing for multiple assets

2. **Agent Integration**
   - Data flow between Agent 1 and Agent 2
   - Asset storage and management
   - Quality control and validation

### Phase 3: Video Production (Week 3)
1. **Agent 3 Implementation**
   - Veo API integration
   - Video composition system
   - Commercial template engine
   - Text overlay and motion graphics

2. **End-to-End Pipeline**
   - Complete three-agent orchestration
   - Progress tracking and user feedback
   - Error handling and recovery

### Phase 4: Polish & Demo (Week 4)
1. **User Experience**
   - Frontend interface refinement
   - Real-time progress indicators
   - Video preview and download features

2. **Demo Preparation**
   - Sample product selection
   - Demo script and flow
   - Performance optimization

## 📊 Performance & Cost Targets

### Processing Metrics
- **Total Time**: <6 minutes end-to-end
- **Agent 1**: <5 seconds (analysis)
- **Agent 2**: <60 seconds (asset generation)
- **Agent 3**: <300 seconds (video production)
- **Success Rate**: >95%

### Cost Structure (per commercial)
- **Gemini Analysis**: ~$0.002
- **Imagen Assets**: ~$0.10 (5 images)
- **Veo Video**: ~$1.50
- **Infrastructure**: ~$0.01
- **Total**: ~$1.61 per commercial

### Quality Targets
- **Video Resolution**: 1080p minimum, 4K preferred
- **Professional Quality**: Suitable for business use
- **User Satisfaction**: >4.0/5.0 rating
- **Demo Reliability**: 100% success in controlled environment

## 🔧 Development Environment

### Prerequisites
- Node.js 18+
- TypeScript 5+
- Google Cloud SDK
- GCP Project with billing enabled

### Local Development Setup
```bash
# Clone repository
git clone <repository-url>
cd hackthon-idea

# Install dependencies
npm install

# Configure environment
cp .env.example .env
# Edit .env with your GCP credentials and project details

# Start development servers
npm run dev          # Frontend (Next.js)
npm run dev:api      # Backend (Express)
npm run dev:functions # Cloud Functions locally
```

### Environment Variables
```env
# Google Cloud Configuration
GCP_PROJECT_ID=your-project-id
GCP_LOCATION=us-central1
GOOGLE_APPLICATION_CREDENTIALS=path/to/service-account.json

# API Endpoints
VERTEX_AI_ENDPOINT=us-central1-aiplatform.googleapis.com
IMAGEN_API_ENDPOINT=us-central1-aiplatform.googleapis.com

# Storage
CLOUD_STORAGE_BUCKET=your-bucket-name
FIRESTORE_DATABASE=(default)

# Application
NODE_ENV=development
PORT=3000
API_BASE_URL=http://localhost:3001
```

## 🧪 Testing Strategy

### Unit Tests
- Agent prompt validation
- API response parsing
- Cost calculation accuracy
- Error handling scenarios

### Integration Tests
- End-to-end agent pipeline
- File upload/download flows
- Database operations
- API endpoint functionality

### Performance Tests
- Processing time benchmarks
- Concurrent user handling
- API quota management
- Memory usage optimization

## 🚨 Risk Management

### Technical Risks
- **API Quota Limits**: Monitor usage, implement queuing
- **Processing Failures**: Robust error handling, fallback options
- **Cost Overruns**: Budget monitoring, usage alerts
- **Performance Issues**: Caching, optimization, scaling

### Mitigation Strategies
- **Fallback Assets**: Stock images/videos for generation failures
- **Quality Tiers**: Multiple processing options (fast vs premium)
- **Progressive Enhancement**: Show intermediate results
- **Comprehensive Testing**: Edge cases and error scenarios

## 📈 Success Criteria

### Technical Achievements
- ✅ Complete three-agent pipeline functional
- ✅ Professional-quality commercial output
- ✅ Sub-6-minute processing time
- ✅ >95% success rate
- ✅ Stable deployment on Google Cloud

### Demo Objectives
- ✅ Live generation during 5-minute presentation
- ✅ Handle 3+ different product categories
- ✅ Professional output quality
- ✅ Engaging user experience
- ✅ Clear business value proposition

### Submission Requirements
- ✅ Working system deployed on Google Cloud
- ✅ Comprehensive Zenn article with technical details
- ✅ Demo video showcasing capabilities
- ✅ Clean, documented GitHub repository
- ✅ Submission by September 24, 2025 (23:59 JST)

## 🔗 Reference Resources

### GCP Implementation Guides
Located in `05-gcp-reference/` directory:
- Vertex AI Gemini TypeScript implementation
- Imagen API TypeScript guide
- Veo API TypeScript integration
- Cloud Run deployment patterns

### Development Timeline
- Detailed week-by-week schedule in `01-planning/timeline/product-commercial-timeline.md`
- Critical milestones and risk mitigation strategies
- Resource allocation and cost management

### Architecture Documentation  
- Multi-agent system design in `02-development/architecture/product-commercial-agent-architecture.md`
- Agent responsibilities and communication patterns
- Technical specifications and API integrations

This design document provides the complete technical roadmap for building the Product Commercial Generator system within the hackathon timeline.