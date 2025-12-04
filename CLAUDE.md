# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

---

## Project Overview

**Trip Kit (트립 키트)** - AI-powered vibe-driven travel platform that curates emotional travel experiences.

**Tagline**: "당신은 티켓만 끊으세요. 여행의 '분위기(Vibe)'는 우리가 챙겨드립니다."

**Core Philosophy**: This is NOT an efficiency-focused travel planner. Trip Kit designs complete aesthetic travel experiences by extracting user's emotional "vibe" and matching it to:
- Hidden local spots (not tourist traps)
- Film camera aesthetics and styling
- Complete creative direction (outfit + props + camera settings)

---

## Project Structure & Architecture

### Documentation-First Approach
This repository currently contains comprehensive planning documentation before implementation. All technical decisions and product requirements are documented in `docs/`:

```
docs/
├── PRD_TripKit_MVP.md           # Product Requirements (Vibe-focused positioning)
├── TRD_TripKit_MVP.md           # Technical Requirements (Architecture & stack)
├── API_Documentation.md         # RESTful API specs (5 vibe-driven endpoints)
└── AI_Integration_Guide.md      # LangGraph + GPT-4 + DALL-E 3 patterns
```

### Planned System Architecture (Vibe-Driven Design)

**3-Layer Architecture**:
1. **Client Layer** (Next.js 14+)
   - Vibe Chat Interface (AI Dialog)
   - Concept Selector (Flâneur/Film Log/Midnight)
   - Hidden Spot Gallery

2. **API Layer** (Next.js API Routes)
   - `/api/chat` - Vibe extraction through LangGraph
   - `/api/recommendations/destinations` - Vibe-matched destinations
   - `/api/recommendations/hidden-spots` - Local-favorite locations
   - `/api/generate/image` - Film aesthetic preview images
   - `/api/recommendations/styling` - Complete styling package

3. **Service Layer**
   - **LangGraph Vibe Analyzer**: Mood → Aesthetic → Duration → Interests
   - **GPT-4 Vibe Matcher**: Matches emotional preferences to destinations
   - **DALL-E 3 Vibe Visualizer**: Film-aesthetic image generation

### Key Technology Decisions (from TRD)

**Frontend**: Next.js 14.2+, React 18+, TypeScript 5.0+, Tailwind CSS, Zustand
**Backend**: Next.js API Routes (serverless)
**AI/ML**: LangGraph, LangChain, OpenAI SDK (GPT-4, DALL-E 3)
**Deployment**: Vercel (MVP), Session storage only (no DB initially)

---

## Environment Setup

### Prerequisites
- Python 3.9+ (for any Python-based prototypes)
- Node.js 20+ (when Next.js implementation begins)
- OpenAI API key
- Anthropic API key (optional)

### Initial Setup (from README.md)
```bash
# Clone repository
git clone https://github.com/KernelAcademy-AICamp/ai-camp-3rd-chatbot-pjt-mint_loop.git
cd mintloop

# Create virtual environment (Python components)
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

# Install dependencies (when requirements.txt exists)
pip install -r requirements.txt

# Create .env file
cat > .env << EOF
OPENAI_API_KEY="YOUR_OPENAI_API_KEY"
ANTHROPIC_API_KEY="YOUR_ANTHROPIC_API_KEY"
EOF
```

### Next.js Setup (when implementation starts)
```bash
# Install Node dependencies
npm install

# Run development server
npm run dev

# Build for production
npm run build

# Run tests
npm test
```

---

## Development Guidelines

### Vibe-Driven Design Principles

When implementing features, always prioritize:

1. **Emotional Preference Extraction**: Use empathetic, warm language in AI prompts
2. **Hidden Spot Curation**: Avoid mainstream tourist attractions; focus on local favorites
3. **Film Aesthetic Authenticity**: Generate images that look like real analog film photography (not digital filters)
4. **Complete Creative Direction**: Every recommendation should be actionable (what to wear, which camera, exact settings)

### AI Prompt Engineering Strategy

**System Prompt Philosophy** (from AI_Integration_Guide.md):
```typescript
const VIBE_SYSTEM_PROMPT = `
You are Trip Kit's Vibe Curator—an empathetic AI specialist in emotional travel design.

Core Philosophy:
- Travel is not just about WHERE to go, but HOW it FEELS and how to CAPTURE it
- Focus on EMOTION over EFFICIENCY, VIBE over ITINERARY
- Use evocative language: "vibe," "aesthetic," "mood," "feel"
`;
```

**Key Patterns**:
- **Vibe Extraction**: Progressive questioning (Mood → Aesthetic → Duration → Interests)
- **Destination Matching**: Use `buildVibeMatchedDestinationPrompt()` with emotional tone
- **Image Generation**: Film stock simulation with authentic grain and vignetting

### API Design Conventions

All API endpoints follow vibe-driven naming:
- NOT: `/recommendations` → YES: `/recommendations/vibe-matched`
- Include `matchReason` explaining emotional alignment
- Provide `vibeProfile` with emotionalTone, visualStyle, energyLevel

### Documentation Standards

When updating documentation:
- Use Korean/English bilingual descriptions for key concepts
- Include emoji indicators (🎭🎨⏱️💫🎬) for visual clarity
- Add comparison tables showing Trip Kit vs Traditional Travel AI
- Emphasize "Why this matters for vibe curation" in technical decisions

---

## Core Concepts Reference

### Three Aesthetic Concepts
1. **플라뇌르 (Flâneur)**: "지도 없이 걷는 낭만" - Urban wandering, literary aesthetic
2. **필름 로그 (Film Log)**: "빈티지 감성 기록" - Retro photography, analog feel
3. **미드나잇 (Midnight)**: "과거 예술가와의 조우" - Artistic time travel, bohemian

### Film Stock Profiles (for image generation)
- **Kodak ColorPlus 200**: Warm, saturated tones (budget-friendly)
- **Kodak Portra 400**: Natural skin tones, subtle colors
- **Fujifilm Superia 400**: Vibrant colors, cool-neutral
- **Ilford HP5 Plus**: B&W, high contrast, classic

### User Preference Schema
```typescript
interface UserPreferences {
  mood: 'romantic' | 'adventurous' | 'nostalgic' | 'peaceful';
  aesthetic: 'urban' | 'nature' | 'vintage' | 'modern';
  duration: 'short' | 'medium' | 'long';
  interests: ('photography' | 'food' | 'art' | 'history' | 'nature')[];
  concept?: 'flaneur' | 'filmlog' | 'midnight';
}
```

---

## Working with Documentation

### When Updating Requirements
1. **PRD Changes**: Update user stories, success metrics, feature priorities
2. **TRD Changes**: Update architecture diagrams, technology stack decisions
3. **API Docs**: Keep endpoint specs synchronized with PRD/TRD
4. **AI Guide**: Update prompt templates when conversation flow changes

### Documentation Cross-References
- PRD defines WHAT and WHY → TRD defines HOW
- API Docs specify contracts → AI Guide shows implementation patterns
- All documents must maintain "vibe-driven" philosophy consistency

---

## Project Timeline & Status

**Current Phase**: Planning & Documentation (Week 0)
**Target**: 1-Week MVP Sprint

### MVP Scope (from PRD)
**P0 (Critical)**:
- AI Chatbot for vibe extraction (LangGraph)
- Concept selection system (3 themes)
- Hidden spot recommendations (GPT-4)

**P1 (High)**:
- AI image generation (DALL-E 3 with film aesthetic)
- Film camera & styling recommendations

**Out of MVP Scope**:
- O2O rental/delivery system
- User authentication
- Payment integration
- Social features

---

## Important Notes

### API Keys & Security
- Store all API keys in `.env` file (never commit)
- Use environment variables: `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`
- MVP uses session storage only (no persistent database)

### Performance Targets (from TRD)
- API Response Time (chat): <3s
- API Response Time (recommendations): <5s
- Image Generation: <15s (nice-to-have)
- Page Load Time: <3s

### Cost Estimation (from AI_Integration_Guide.md)
- Complete user flow: ~$0.335 per user
- Monthly cost for 1,000 users: ~$335
- Optimize with caching (60-70% reduction)

---

## Git Workflow

### Commit Message Format
```
<type>(<scope>): <subject>

<body>

🤖 Generated with [Claude Code](https://claude.com/claude-code)
Co-Authored-By: Claude <noreply@anthropic.com>
```

**Types**: `feat`, `fix`, `docs`, `refactor`, `test`, `chore`
**Scopes**: `prd`, `trd`, `api`, `ai`, `docs`

### Branch Strategy
- `main`: Production-ready documentation and code
- Feature branches: `feature/<feature-name>`
- Documentation updates: Commit directly to `main` (current phase)

---

## Reference Links

- **OpenAI API**: https://platform.openai.com/docs/api-reference
- **LangGraph Docs**: https://langchain-ai.github.io/langgraph/
- **Next.js 14**: https://nextjs.org/docs
- **DALL-E Best Practices**: https://platform.openai.com/docs/guides/images
