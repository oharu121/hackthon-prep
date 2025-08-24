## FEATURE:

Interactive multi-agent AI system that transforms a single product image into a professional commercial video through real-time conversations with three specialized agents:

- **Agent 1 (Product Intelligence)**: Uses Vertex AI Gemini Pro Vision to analyze uploaded product images, then engages in real-time chat to refine product features, target audience, and positioning based on user feedback and preferences.
- **Agent 2 (Creative Director)**: Uses Gemini for conversation and Imagen API for asset generation. Presents multiple commercial style options, discusses scene structure, and generates visual assets based on interactive user feedback.
- **Agent 3 (Video Producer)**: Uses Veo API for video generation and presents narration scripts, audio options, and pacing preferences through chat. Combines all assets into the final commercial based on user's production choices.

**Interactive Flow:**
1. **Upload & Analyze**: User uploads image + optional description → Agent 1 analyzes and discusses findings
2. **Style Selection**: Agent 2 presents commercial style options → User provides feedback → Scene breakdown discussion
3. **Production Choices**: Agent 3 shows narration script and production options → User selects preferences → Final video generation

The system provides a React/Next.js frontend with WebSocket-based real-time chat, TypeScript throughout, Node.js/Express backend with session management, and Google Cloud infrastructure for processing and storage.

## EXAMPLES:

In the `examples/` folder, explore the agent architecture patterns:

- `examples/pydantic-ai/` - Reference implementations for agent structures, tool integration, and testing patterns that can be adapted for the three commercial generation agents.
- `examples/agent-factory-with-subagents/` - Multi-agent coordination patterns useful for orchestrating the three-agent pipeline.
- `discussion/02-development/architecture/` - Contains detailed agent architecture documentation and communication patterns.
- `05-gcp-reference/` - TypeScript implementation guides for Vertex AI Gemini, Imagen API, Veo API, and Cloud Run deployment.

Don't copy these examples directly as they're for different projects, but use them to understand multi-agent coordination, GCP API integration patterns, and TypeScript best practices.

## DOCUMENTATION:

Google Cloud Platform APIs and services to be referenced:
- Vertex AI Gemini Pro Vision API: https://cloud.google.com/vertex-ai/docs/generative-ai/model-reference/gemini
- Vertex AI Gemini Pro (text) for conversational agents: https://cloud.google.com/vertex-ai/docs/generative-ai/chat/chat-prompts
- Imagen API documentation: https://cloud.google.com/vertex-ai/docs/generative-ai/image/overview
- Veo API documentation: https://deepmind.google/technologies/veo/
- Cloud Storage API: https://cloud.google.com/storage/docs/apis
- Firestore documentation (for session/chat state): https://cloud.google.com/firestore/docs
- Cloud Functions TypeScript guide: https://cloud.google.com/functions/docs/writing
- Cloud Run deployment: https://cloud.google.com/run/docs
- WebSocket implementation patterns: https://socket.io/docs/v4/
- Text-to-Speech API (for narration generation): https://cloud.google.com/text-to-speech/docs

Reference the detailed technical specifications in `discussion/DESIGN.md` for complete system architecture, API schemas, and processing workflows.

## OTHER CONSIDERATIONS:

- **Interactive Chat Architecture**: Implement WebSocket connections for real-time agent conversations. Include session state management across agent handoffs and context preservation between chat phases.
- **Cost Management**: Monitor API usage closely - estimated $1.81-2.01 per commercial (Gemini conversations: $0.20-0.40, Imagen: $0.10, Veo: $1.50). Implement usage tracking and budget alerts for extended chat sessions.
- **Processing Time**: Target 8-12 minutes total including interactive phases. Implement smart defaults and 30-second timeout fallbacks if user doesn't respond, allowing agents to proceed automatically.
- **Fallback Strategy**: Dual-mode system - interactive chat for full experience, automatic fallback processing to ensure demo reliability. Each agent must be able to proceed without user input.
- **Session Management**: Robust state persistence for multi-phase conversations. Handle WebSocket disconnections, chat history, and agent handoff scenarios gracefully.
- **Error Handling**: Comprehensive fallback strategies for API failures, quota limits, chat timeouts, and quality control. Include intermediate result preservation and automatic recovery modes.
- **Security**: Never expose GCP service account credentials. Implement secure WebSocket authentication and session validation. Use Cloud IAM and service accounts properly.
- **TypeScript Throughout**: Maintain type safety across frontend, backend, Cloud Functions, and WebSocket communications. Define comprehensive interfaces for agent outputs and chat states.
- **Quality Control**: Automated filtering for brand appropriateness and content quality. Validate user inputs and provide guided conversation flows to prevent inappropriate content generation.
- **Scalability**: Design for concurrent chat sessions and multiple users. Consider connection pooling for WebSockets and queuing for resource-intensive operations.
- **Demo Readiness**: System must work reliably during live 5-minute hackathon presentation with <95% success rate. Prepare sample conversations and quick-select options for smooth demo flow.
- **User Experience**: Implement typing indicators, message timestamps, quick-reply buttons alongside free-form chat, and clear visual handoffs between agents to maintain engagement.
