# Product Commercial Generator - Agent Architecture

## Multi-Agent System Overview

The Product Commercial Generator uses three specialized AI agents working in sequence to transform a single product image into a professional commercial video.

```
Product Image → Agent 1 → Agent 2 → Agent 3 → Commercial Video
    Input    →Analysis→ Visuals → Production→    Output
```

---

## 🔍 Agent 1: Product Analyzer (Vertex AI Gemini)

### Primary Function
Analyzes uploaded product image to extract marketing insights and commercial strategy

### Core Capabilities
- **Product Recognition**: Identify product type, category, materials, colors
- **Feature Extraction**: Key selling points, unique characteristics, quality indicators
- **Target Audience**: Demographics, psychographics, use cases
- **Brand Positioning**: Premium/budget, lifestyle alignment, emotional triggers
- **Commercial Strategy**: Tone, messaging, visual style recommendations

### Input/Output
```
INPUT: Product image + optional product description
OUTPUT: Structured analysis object {
  productType: string,
  features: string[],
  benefits: string[],
  targetAudience: object,
  brandPosition: object,
  visualStyle: object,
  commercialConcept: object
}
```

### Implementation Details
- **Model**: Vertex AI Gemini Pro Vision
- **Prompt Engineering**: Marketing expert persona with commercial production knowledge
- **Processing Time**: ~2-3 seconds
- **Fallback**: Default commercial templates for unrecognized products

---

## 🎨 Agent 2: Creative Director (Imagen API)

### Primary Function
Generates additional visual assets based on Agent 1's analysis

### Core Capabilities
- **Lifestyle Scenes**: Product in use/context scenarios
- **Background Environments**: Studios, lifestyle settings, abstract backgrounds
- **Supporting Visuals**: Ingredient shots, detail views, comparison images
- **Brand Elements**: Logo placements, color schemes, visual branding
- **Mood Boards**: Visual style consistency across generated assets

### Input/Output
```
INPUT: Agent 1 analysis + original product image
OUTPUT: Asset collection {
  lifestyleScenes: Image[],
  backgrounds: Image[],
  detailShots: Image[],
  brandingElements: Image[],
  visualStyle: StyleGuide
}
```

### Generation Strategy
- **Scene Types**: 3-5 lifestyle scenarios per product
- **Aspect Ratios**: 16:9 for video, 1:1 for social, 9:16 for vertical
- **Style Consistency**: Maintained color palette and lighting across assets
- **Quality Control**: Automated filtering for brand-appropriate content

### Implementation Details
- **Model**: Imagen 2 or Imagen 3 (latest available)
- **Batch Processing**: Parallel generation for efficiency
- **Processing Time**: ~30-45 seconds for full asset collection
- **Caching**: Store generated assets for similar product types

---

## 🎬 Agent 3: Video Producer (Veo API)

### Primary Function
Combines all assets into final commercial video with professional production quality

### Core Capabilities
- **Video Composition**: Seamless transitions between scenes and assets
- **Text Overlays**: Dynamic product names, benefits, call-to-actions
- **Motion Graphics**: Animated logos, price reveals, feature callouts
- **Audio Integration**: Background music, sound effects, voiceover placeholders
- **Pacing Control**: Commercial rhythm matching target audience

### Input/Output
```
INPUT: Agent 1 analysis + Agent 2 assets + original product image
OUTPUT: Commercial video {
  duration: 15s | 30s | 60s,
  format: MP4,
  resolution: 1080p | 4K,
  audioTrack: boolean,
  captions: boolean,
  brandingLevel: minimal | moderate | heavy
}
```

### Video Structure Template
```
0:00-0:03  Product Hero Shot + Brand Intro
0:03-0:12  Lifestyle Scenes + Feature Highlights
0:12-0:18  Product Benefits + Social Proof
0:18-0:30  Call to Action + Contact Info
```

### Implementation Details
- **Model**: Veo 2 (latest video generation)
- **Template System**: Pre-designed commercial structures
- **Customization**: Dynamic content insertion based on product type
- **Processing Time**: ~3-5 minutes for 30s commercial
- **Quality Tiers**: Express (lower quality, faster) vs Premium (high quality)

---

## 🔄 Agent Coordination & Workflow

### Orchestration Flow
```python
class CommercialGenerator:
    def generate_commercial(self, product_image, user_preferences):
        # Stage 1: Analysis
        analysis = agent1_analyzer.analyze_product(product_image)
        
        # Stage 2: Visual Assets (parallel processing)
        assets = agent2_creative.generate_assets(analysis, product_image)
        
        # Stage 3: Video Production
        commercial = agent3_producer.create_commercial(
            analysis, assets, user_preferences
        )
        
        return commercial
```

### Error Handling & Fallbacks
- **Agent 1 Failure**: Use generic product analysis template
- **Agent 2 Failure**: Use stock footage and basic backgrounds
- **Agent 3 Failure**: Create simple slideshow with transitions
- **API Limits**: Queue system with user notifications

### Performance Optimization
- **Parallel Processing**: Agents 2 and 3 can process simultaneously where possible
- **Caching Strategy**: Store analysis for similar products
- **Progressive Enhancement**: Show intermediate results to user
- **Quality Settings**: Fast/Standard/Premium processing tiers

---

## 📊 Technical Specifications

### System Requirements
- **Backend**: Node.js with Express.js
- **Processing**: Google Cloud Functions (scaling)
- **Storage**: Cloud Storage for media assets
- **Database**: Firestore for analysis caching
- **Frontend**: React/Next.js for real-time progress

### API Integration
```javascript
// Vertex AI Gemini Setup
const geminiClient = new VertexAI({
  project: 'hackathon-project',
  location: 'us-central1'
});

// Imagen API Setup
const imagenClient = new ImageGenerationServiceClient({
  apiEndpoint: 'us-central1-aiplatform.googleapis.com'
});

// Veo API Setup
const veoClient = new VideoGenerationClient({
  project: 'hackathon-project'
});
```

### Processing Pipeline
1. **Upload & Validation**: Image format, size, content policy check
2. **Agent 1 Processing**: Product analysis (2-3s)
3. **Agent 2 Processing**: Asset generation (30-45s)
4. **Agent 3 Processing**: Video creation (3-5min)
5. **Delivery**: Download link, social media formats

### Cost Optimization
- **Gemini**: ~$0.002 per analysis
- **Imagen**: ~$0.02 per generated image (5 images avg)
- **Veo**: ~$1.50 per 30s video
- **Total Cost**: ~$1.60 per commercial (well within budget)

---

## 🎯 Success Metrics

### Quality Metrics
- **Video Resolution**: 1080p minimum, 4K preferred
- **Processing Success Rate**: >95%
- **User Satisfaction**: >4.0/5.0 rating
- **Brand Accuracy**: Manual review scoring

### Performance Metrics
- **Total Processing Time**: <6 minutes end-to-end
- **Agent 1**: <5 seconds
- **Agent 2**: <60 seconds  
- **Agent 3**: <300 seconds
- **System Availability**: >99% uptime during demo

### Demo Metrics
- **Live Processing**: Successfully generate commercial during 5-minute demo
- **Variety**: Handle 3+ different product categories
- **Visual Quality**: Professional-grade output suitable for business use