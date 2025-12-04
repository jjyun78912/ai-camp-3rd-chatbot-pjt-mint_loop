# Trip Kit - Technical Requirements Document (TRD)
## MVP Version - 1 Week Sprint

---

## 📋 Document Information

- **Document Version**: 1.0.0
- **Last Updated**: 2025-12-03
- **Project Timeline**: 1 Week (MVP)
- **Related Documents**: [PRD_TripKit_MVP.md](./PRD_TripKit_MVP.md)
- **Author**: Engineering Team
- **Status**: Active Development

---

## 🏗️ System Architecture Overview

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                         Client Layer                         │
│                    (Next.js 14+ App Router)                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │  Chat UI     │  │  Concept     │  │  Location    │     │
│  │  Component   │  │  Selector    │  │  Gallery     │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
└─────────────────────────────────────────────────────────────┘
                            │
                            ↓ (API Routes)
┌─────────────────────────────────────────────────────────────┐
│                      API Layer (Next.js)                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │ /api/chat    │  │ /api/        │  │ /api/        │     │
│  │              │  │ recommend    │  │ generate     │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
└─────────────────────────────────────────────────────────────┘
                            │
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                     Service Layer                            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │  LangGraph   │  │  Destination │  │  Image Gen   │     │
│  │  Chatbot     │  │  Recommender │  │  Service     │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
└─────────────────────────────────────────────────────────────┘
                            │
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                   External Services                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │   OpenAI     │  │   DALL-E 3   │  │   Maps API   │     │
│  │   GPT-4      │  │              │  │  (Optional)  │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
└─────────────────────────────────────────────────────────────┘
                            │
                            ↓
┌─────────────────────────────────────────────────────────────┐
│              Storage Layer (MVP: Session Only)               │
│  ┌──────────────┐  ┌──────────────┐                        │
│  │   Browser    │  │    Redis     │                        │
│  │  SessionStorage│ │  (Optional)  │                        │
│  └──────────────┘  └──────────────┘                        │
└─────────────────────────────────────────────────────────────┘
```

### Architecture Principles
1. **API-First Design**: All features exposed through RESTful APIs
2. **Stateless**: No server-side session storage (browser session only)
3. **Serverless**: Leverage Vercel serverless functions
4. **Modular Services**: Clear separation between chatbot, recommendations, image generation
5. **External API Abstraction**: Wrap external APIs for easier testing & swapping

---

## 🛠️ Technology Stack

### Frontend
| Technology | Version | Purpose |
|------------|---------|---------|
| **Next.js** | 14.2+ | React framework, SSR, API routes |
| **React** | 18+ | UI component library |
| **TypeScript** | 5.0+ | Type safety |
| **Tailwind CSS** | 3.4+ | Styling framework |
| **Zustand** | 4.5+ | State management (lightweight) |
| **React Query** | 5.0+ | Data fetching & caching |

### Backend/API
| Technology | Version | Purpose |
|------------|---------|---------|
| **Next.js API Routes** | 14.2+ | Backend API endpoints |
| **LangGraph** | Latest | Conversation state management |
| **LangChain** | 0.1+ | AI orchestration framework |
| **OpenAI SDK** | 4.0+ | GPT-4, DALL-E 3 integration |

### Infrastructure & DevOps
| Technology | Purpose |
|------------|---------|
| **Vercel** | Hosting, serverless functions, CI/CD |
| **Git/GitHub** | Version control |
| **ESLint/Prettier** | Code quality |
| **Jest/React Testing Library** | Unit testing |

### Optional/Future
| Technology | Purpose | Status |
|------------|---------|--------|
| **Supabase** | PostgreSQL database | Post-MVP |
| **Redis** | Session caching | Post-MVP |
| **MCP Servers** | Context management | Post-MVP |

---

## 📐 System Design

### 1. Conversation Engine (LangGraph)

#### State Graph Structure
```typescript
interface ConversationState {
  step: 'init' | 'mood' | 'aesthetic' | 'duration' | 'interests' | 'complete';
  messages: Message[];
  userPreferences: {
    mood?: string;
    aesthetic?: string;
    duration?: string;
    interests?: string[];
  };
  recommendations?: Destination[];
}

type StateTransition = (state: ConversationState) => ConversationState;
```

#### LangGraph Flow
```
[START]
  ↓
[Welcome Node] → Ask user name/greeting
  ↓
[Mood Node] → "What vibe are you feeling? (romantic/adventurous/nostalgic)"
  ↓
[Aesthetic Node] → "Urban or nature? Modern or vintage?"
  ↓
[Duration Node] → "How long is your trip?"
  ↓
[Interests Node] → "Photography? Food? Art?"
  ↓
[Analysis Node] → Process preferences with GPT-4
  ↓
[Recommendation Node] → Generate 3 destinations
  ↓
[END]
```

#### Key Functions
```typescript
// Core LangGraph setup
const conversationGraph = new StateGraph({
  channels: {
    messages: { reducer: messagesReducer },
    preferences: { reducer: preferencesReducer },
  }
});

// Add nodes
conversationGraph.addNode('mood', moodNode);
conversationGraph.addNode('aesthetic', aestheticNode);
conversationGraph.addNode('duration', durationNode);
conversationGraph.addNode('interests', interestsNode);
conversationGraph.addNode('analyze', analyzeNode);
conversationGraph.addNode('recommend', recommendNode);

// Define transitions
conversationGraph.addEdge('mood', 'aesthetic');
conversationGraph.addEdge('aesthetic', 'duration');
conversationGraph.addEdge('duration', 'interests');
conversationGraph.addEdge('interests', 'analyze');
conversationGraph.addEdge('analyze', 'recommend');

// Compile
const chatbot = conversationGraph.compile();
```

---

### 2. Recommendation Engine

#### Destination Recommendation Algorithm

**Input Processing**:
```typescript
interface UserPreferences {
  mood: 'romantic' | 'adventurous' | 'nostalgic' | 'peaceful';
  aesthetic: 'urban' | 'nature' | 'vintage' | 'modern';
  duration: 'short' | 'medium' | 'long'; // 1-3, 4-7, 8+ days
  interests: ('photography' | 'food' | 'art' | 'history' | 'nature')[];
  concept?: 'flaneur' | 'filmlog' | 'midnight'; // Selected later
}
```

**GPT-4 Prompt Template**:
```typescript
const DESTINATION_PROMPT = `
You are a local travel expert specializing in hidden, non-touristy destinations.

User Preferences:
- Mood: {mood}
- Aesthetic: {aesthetic}
- Duration: {duration} days
- Interests: {interests}
- Selected Concept: {concept}

Generate 3 unique destination recommendations that:
1. Are NOT in top-10 tourist lists
2. Offer photogenic, aesthetic locations
3. Accessible by public transport
4. Safe for solo travelers
5. Match the user's mood and interests
6. Align with the selected concept aesthetic

For each destination, provide:
- Name & city/region
- 2-3 sentence description
- Why it matches user preferences
- Best time to visit
- Photography potential (1-10 score)

Output as JSON array.
`;
```

**Output Structure**:
```typescript
interface Destination {
  id: string;
  name: string;
  city: string;
  country: string;
  description: string;
  matchReason: string;
  bestTimeToVisit: string;
  photographyScore: number; // 1-10
  transportAccessibility: 'easy' | 'moderate' | 'challenging';
  safetyRating: number; // 1-10
  hiddenSpots: HiddenSpot[]; // Populated after selection
}
```

#### Hidden Spot Generation

**Triggered When**: User selects a destination

**GPT-4 Prompt**:
```typescript
const HIDDEN_SPOTS_PROMPT = `
Generate 5-10 hidden, local-favorite locations in {destination} that:
- Are NOT major tourist attractions
- Highly photogenic (film aesthetic)
- Match concept: {concept}
- Accessible and safe

For each location:
- Name & exact address
- Description (50-100 words)
- Best photography time (golden hour, blue hour, etc.)
- Camera angles/tips
- Estimated visit duration
- Nearby amenities (cafes, restrooms)

Output as JSON array.
`;
```

**Output Structure**:
```typescript
interface HiddenSpot {
  id: string;
  name: string;
  address: string;
  coordinates?: { lat: number; lng: number };
  description: string;
  photographyTips: string[];
  bestTimeToVisit: string;
  estimatedDuration: string; // "30min", "1-2hr"
  nearbyAmenities: string[];
  accessibilityNotes: string;
  filmRecommendations: FilmRecommendation[];
}
```

---

### 3. Image Generation Service

#### Architecture
```typescript
interface ImageGenerationRequest {
  locationId: string;
  locationName: string;
  locationDescription: string;
  userPhoto?: string; // Base64 or URL (optional for MVP)
  selectedConcept: 'flaneur' | 'filmlog' | 'midnight';
  filmStock: 'kodak_colorplus' | 'fuji_superia' | 'ilford_hp5';
  outfitStyle: string;
}

interface ImageGenerationResponse {
  imageUrl: string;
  prompt: string;
  generationTime: number; // ms
  status: 'success' | 'pending' | 'failed';
  errorMessage?: string;
}
```

#### DALL-E 3 Integration

**Prompt Engineering Template**:
```typescript
const IMAGE_PROMPT_TEMPLATE = `
Create a high-quality photograph in the style of {filmStock} film.

Scene Description:
{locationDescription}

Subject:
- {genderNeutralPerson} wearing {outfitStyle}
- Holding a vintage film camera (35mm)
- Natural, candid pose
- Looking {direction} with {expression}

Film Aesthetic:
- Film grain texture
- {filmStock} color profile
- Slight vignetting
- Natural lighting ({timeOfDay})
- Bokeh in background

Composition:
- Subject positioned {composition}
- {backgroundElements}
- Depth of field: f/1.8
- Authentic analog film look

Style: Cinematic, nostalgic, highly detailed, professional film photography
`;

// Example filled template
const examplePrompt = `
Create a high-quality photograph in the style of Kodak ColorPlus 200 film.

Scene Description:
A hidden Parisian bookshop alley with vintage storefronts and cobblestone streets, early morning light filtering through the buildings.

Subject:
- A young woman wearing a beige trench coat and vintage beret
- Holding a vintage film camera (35mm Olympus)
- Natural, candid pose
- Looking at a bookshop window with gentle smile

Film Aesthetic:
- Fine grain texture
- Warm, saturated Kodak ColorPlus color profile
- Slight vignetting
- Natural morning light
- Bokeh from background street lights

Composition:
- Subject positioned in right third of frame
- Bookshop window and vintage books in background
- Depth of field: f/1.8
- Authentic analog film look

Style: Cinematic, nostalgic, highly detailed, professional film photography
`;
```

**API Integration**:
```typescript
import OpenAI from 'openai';

const openai = new OpenAI({
  apiKey: process.env.OPENAI_API_KEY,
});

async function generateImage(request: ImageGenerationRequest): Promise<ImageGenerationResponse> {
  const prompt = buildPrompt(request);

  try {
    const response = await openai.images.generate({
      model: 'dall-e-3',
      prompt,
      n: 1,
      size: '1024x1024',
      quality: 'hd',
      style: 'natural', // More photorealistic
    });

    return {
      imageUrl: response.data[0].url,
      prompt,
      generationTime: Date.now() - startTime,
      status: 'success',
    };
  } catch (error) {
    return {
      imageUrl: '',
      prompt,
      generationTime: Date.now() - startTime,
      status: 'failed',
      errorMessage: error.message,
    };
  }
}
```

---

### 4. Styling & Film Recommendations

#### Recommendation Engine Logic

**Input**: Selected location + concept
**Output**: Complete styling package

```typescript
interface StylingRecommendation {
  cameraModel: string;
  filmStock: FilmStock;
  cameraSettings: CameraSettings;
  outfitSuggestions: OutfitSuggestion;
  props: Prop[];
  bestAngles: Angle[];
}

interface FilmStock {
  name: string;
  iso: number;
  colorProfile: string;
  characteristics: string[];
  sampleImages: string[]; // URLs to reference images
  priceRange: string; // "$", "$$", "$$$"
}

interface CameraSettings {
  aperture: string; // "f/1.8", "f/2.8"
  shutterSpeed: string; // "1/250s", "1/125s"
  iso: number;
  focusMode: 'auto' | 'manual';
  meteringMode: string;
  lightingNotes: string;
}

interface OutfitSuggestion {
  colorPalette: string[]; // Hex codes
  style: string; // "Casual vintage", "Bohemian chic"
  specificItems: string[]; // "Beige trench coat", "Wide-brim hat"
  seasonalNotes: string;
}

interface Prop {
  name: string;
  purpose: string; // "Add vintage atmosphere", "Frame composition"
  whereToFind: string; // "Local bookstores", "Flea markets"
  optional: boolean;
}

interface Angle {
  description: string;
  visualExample?: string; // URL to reference image
  technique: string; // "Rule of thirds", "Leading lines"
  bestLighting: string;
}
```

**Concept-Specific Recommendations**:

```typescript
const CONCEPT_MAPPINGS = {
  flaneur: {
    cameras: ['Leica M6', 'Contax G2', 'Olympus OM-1'],
    filmStocks: ['Kodak Portra 400', 'Ilford HP5'],
    outfitStyle: 'Minimalist urban, neutral tones',
    props: ['Vintage map', 'Pocket notebook', 'Film canister'],
    angles: ['Street-level perspective', 'Reflections in windows', 'Leading lines']
  },
  filmlog: {
    cameras: ['Canon AE-1', 'Pentax K1000', 'Nikon FM2'],
    filmStocks: ['Kodak ColorPlus 200', 'Fujifilm Superia 400'],
    outfitStyle: 'Retro casual, warm colors',
    props: ['Polaroid camera', 'Vintage postcards', 'Retro sunglasses'],
    angles: ['Rule of thirds', 'Bokeh backgrounds', 'Golden hour glow']
  },
  midnight: {
    cameras: ['Hasselblad 500C/M', 'Rolleiflex', 'Mamiya 7'],
    filmStocks: ['Kodak Tri-X 400', 'Ilford Delta 3200'],
    outfitStyle: 'Artistic, dramatic, layered textures',
    props: ['Antique book', 'Vintage glasses', 'Artist portfolio'],
    angles: ['Low-key lighting', 'Dramatic shadows', 'Artistic bokeh']
  }
};
```

**AI-Enhanced Recommendation**:
```typescript
const STYLING_PROMPT = `
Given location: {locationName} ({locationDescription})
Selected concept: {concept}
Time of visit: {timeOfDay}

Generate detailed photography and styling recommendations:

1. Film Camera: Suggest 1-2 models that match the concept and location aesthetic
2. Film Stock: Recommend specific film (brand, ISO) with color profile notes
3. Camera Settings: Aperture, shutter speed, ISO for optimal results
4. Outfit Styling: Color palette, specific items, seasonal considerations
5. Props: 2-3 small items to enhance composition
6. Best Angles: 3-5 specific composition techniques with lighting notes

Prioritize:
- Authenticity over trends
- Practical, achievable recommendations
- Concept alignment
- Location-specific considerations

Output as structured JSON.
`;
```

---

## 🔌 API Specification

### Base URL
```
Development: http://localhost:3000/api
Production: https://tripkit.vercel.app/api
```

### Authentication
**MVP**: No authentication required (public endpoints)
**Future**: JWT-based authentication

---

### Endpoints

#### 1. **POST /api/chat**
**Purpose**: Process conversation messages and advance chatbot state

**Request**:
```typescript
{
  "sessionId": "uuid-v4", // Client-generated session ID
  "message": "I'm looking for a romantic, vintage vibe trip",
  "currentStep": "mood" | "aesthetic" | "duration" | "interests",
  "preferences": {
    "mood": "romantic",
    "aesthetic": "vintage"
    // Accumulated preferences
  }
}
```

**Response**:
```typescript
{
  "reply": "Great! Romantic and vintage sounds wonderful. Are you more drawn to urban settings or natural landscapes?",
  "nextStep": "aesthetic",
  "isComplete": false,
  "recommendations": null // Only populated when isComplete=true
}
```

**Error Responses**:
```typescript
// 400 Bad Request
{
  "error": "Invalid step transition",
  "message": "Cannot transition from 'duration' to 'mood'"
}

// 500 Internal Server Error
{
  "error": "OpenAI API error",
  "message": "Rate limit exceeded"
}
```

---

#### 2. **POST /api/recommendations/destinations**
**Purpose**: Generate destination recommendations based on user preferences

**Request**:
```typescript
{
  "preferences": {
    "mood": "romantic",
    "aesthetic": "vintage",
    "duration": "medium",
    "interests": ["photography", "art"],
    "concept": "filmlog" // Selected after destinations
  }
}
```

**Response**:
```typescript
{
  "destinations": [
    {
      "id": "dest_1",
      "name": "Cinque Terre Hidden Trails",
      "city": "Cinque Terre",
      "country": "Italy",
      "description": "Lesser-known hiking paths connecting colorful cliffside villages, away from cruise ship crowds.",
      "matchReason": "Combines romantic coastal views with vintage Italian charm, perfect for film photography.",
      "bestTimeToVisit": "Late April - Early June (spring bloom, fewer tourists)",
      "photographyScore": 9,
      "transportAccessibility": "moderate",
      "safetyRating": 9
    },
    // 2 more destinations...
  ],
  "generatedAt": "2025-12-03T10:30:00Z"
}
```

---

#### 3. **POST /api/recommendations/hidden-spots**
**Purpose**: Generate hidden spot recommendations for a selected destination

**Request**:
```typescript
{
  "destinationId": "dest_1",
  "concept": "filmlog",
  "preferences": {
    "mood": "romantic",
    "interests": ["photography"]
  }
}
```

**Response**:
```typescript
{
  "hiddenSpots": [
    {
      "id": "spot_1",
      "name": "Via dell'Amore Secret Overlook",
      "address": "Path 2, Riomaggiore to Manarola",
      "coordinates": { "lat": 44.0996, "lng": 9.7368 },
      "description": "A quiet observation point above the famous lovers' path, offering panoramic views without the crowds. Local fishermen know this spot for stunning sunset backdrops.",
      "photographyTips": [
        "Golden hour: 30min before sunset",
        "Use wide aperture (f/1.8-2.8) for bokeh",
        "Frame with olive trees in foreground"
      ],
      "bestTimeToVisit": "Sunrise (6:30 AM) or Sunset (7:00 PM)",
      "estimatedDuration": "45min - 1hr",
      "nearbyAmenities": ["Trattoria dal Billy (5min walk)", "Public restroom at trail entrance"],
      "accessibilityNotes": "Requires 10min uphill hike, uneven terrain",
      "filmRecommendations": [
        {
          "filmStock": "Kodak ColorPlus 200",
          "reason": "Captures warm coastal light beautifully"
        }
      ]
    },
    // 4-9 more spots...
  ]
}
```

---

#### 4. **POST /api/generate/image**
**Purpose**: Generate AI preview image for a location

**Request**:
```typescript
{
  "locationId": "spot_1",
  "locationName": "Via dell'Amore Secret Overlook",
  "locationDescription": "Quiet observation point with panoramic coastal views...",
  "concept": "filmlog",
  "filmStock": "kodak_colorplus",
  "outfitStyle": "Vintage denim jacket, white sundress",
  "userPhoto": "data:image/jpeg;base64,..." // Optional
}
```

**Response**:
```typescript
{
  "imageUrl": "https://oaidalleapiprodscus.blob.core.windows.net/...",
  "prompt": "Create a high-quality photograph in the style of Kodak ColorPlus 200...",
  "generationTime": 12500, // ms
  "status": "success"
}
```

**Async Variation** (if generation takes >15s):
```typescript
// Initial response (202 Accepted)
{
  "taskId": "task_abc123",
  "status": "pending",
  "estimatedWait": 15000 // ms
}

// Poll endpoint: GET /api/generate/image/:taskId
{
  "taskId": "task_abc123",
  "status": "success" | "pending" | "failed",
  "imageUrl": "...", // Only when status=success
  "errorMessage": "..." // Only when status=failed
}
```

---

#### 5. **POST /api/recommendations/styling**
**Purpose**: Get film camera, outfit, and prop recommendations

**Request**:
```typescript
{
  "locationId": "spot_1",
  "concept": "filmlog",
  "timeOfDay": "sunset",
  "weather": "clear" // optional
}
```

**Response**:
```typescript
{
  "cameraModel": "Canon AE-1",
  "filmStock": {
    "name": "Kodak ColorPlus 200",
    "iso": 200,
    "colorProfile": "Warm, saturated tones with slight red-orange shift",
    "characteristics": ["Affordable", "Versatile", "Great for travel"],
    "sampleImages": [
      "https://filmsamples.com/kodak-colorplus-1.jpg",
      "https://filmsamples.com/kodak-colorplus-2.jpg"
    ],
    "priceRange": "$"
  },
  "cameraSettings": {
    "aperture": "f/2.8",
    "shutterSpeed": "1/250s",
    "iso": 200,
    "focusMode": "auto",
    "meteringMode": "Center-weighted",
    "lightingNotes": "Sunset provides soft, warm directional light. Meter for highlights to avoid overexposure."
  },
  "outfitSuggestions": {
    "colorPalette": ["#F5E6D3", "#8B7355", "#FFFFFF"],
    "style": "Casual vintage, relaxed coastal",
    "specificItems": [
      "Vintage denim jacket (light wash)",
      "White linen sundress",
      "Woven straw hat",
      "Leather sandals"
    ],
    "seasonalNotes": "Layers for evening breeze, breathable fabrics"
  },
  "props": [
    {
      "name": "Vintage Polaroid camera",
      "purpose": "Add nostalgic element to composition",
      "whereToFind": "Bring from home or local vintage shops",
      "optional": false
    },
    {
      "name": "Woven basket with flowers",
      "purpose": "Coastal aesthetic enhancement",
      "whereToFind": "Local markets in Riomaggiore",
      "optional": true
    }
  ],
  "bestAngles": [
    {
      "description": "Rule of thirds with horizon line",
      "visualExample": "https://angles.com/rule-of-thirds-sunset.jpg",
      "technique": "Position subject in right third, ocean in left two-thirds",
      "bestLighting": "15-30min before sunset (golden hour)"
    },
    {
      "description": "Bokeh with coastal background",
      "visualExample": "https://angles.com/bokeh-coast.jpg",
      "technique": "Use f/1.8-2.8, focus on subject, blur colorful village background",
      "bestLighting": "Soft diffused light (cloudy or shade)"
    },
    {
      "description": "Leading lines with path",
      "visualExample": "https://angles.com/leading-lines.jpg",
      "technique": "Use path/railings to lead eye to subject",
      "bestLighting": "Any, but avoid harsh midday sun"
    }
  ]
}
```

---

## 💾 Data Models

### TypeScript Interfaces

```typescript
// Core Entities
interface User {
  sessionId: string; // MVP: Browser sessionStorage
  preferences: UserPreferences;
  conversationState: ConversationState;
  selectedDestination?: string; // Destination ID
  selectedSpots: string[]; // Array of spot IDs
}

interface UserPreferences {
  mood: 'romantic' | 'adventurous' | 'nostalgic' | 'peaceful';
  aesthetic: 'urban' | 'nature' | 'vintage' | 'modern';
  duration: 'short' | 'medium' | 'long';
  interests: Interest[];
  concept?: Concept;
}

interface Destination {
  id: string;
  name: string;
  city: string;
  country: string;
  description: string;
  matchReason: string;
  bestTimeToVisit: string;
  photographyScore: number;
  transportAccessibility: 'easy' | 'moderate' | 'challenging';
  safetyRating: number;
  hiddenSpots: HiddenSpot[];
}

interface HiddenSpot {
  id: string;
  name: string;
  address: string;
  coordinates?: Coordinates;
  description: string;
  photographyTips: string[];
  bestTimeToVisit: string;
  estimatedDuration: string;
  nearbyAmenities: string[];
  accessibilityNotes: string;
  filmRecommendations: FilmRecommendation[];
}

interface StylingRecommendation {
  cameraModel: string;
  filmStock: FilmStock;
  cameraSettings: CameraSettings;
  outfitSuggestions: OutfitSuggestion;
  props: Prop[];
  bestAngles: Angle[];
}

// Supporting Types
type Interest = 'photography' | 'food' | 'art' | 'history' | 'nature' | 'architecture';
type Concept = 'flaneur' | 'filmlog' | 'midnight';

interface Coordinates {
  lat: number;
  lng: number;
}

interface FilmRecommendation {
  filmStock: string;
  reason: string;
}

interface FilmStock {
  name: string;
  iso: number;
  colorProfile: string;
  characteristics: string[];
  sampleImages: string[];
  priceRange: '$' | '$$' | '$$$';
}

interface CameraSettings {
  aperture: string;
  shutterSpeed: string;
  iso: number;
  focusMode: 'auto' | 'manual';
  meteringMode: string;
  lightingNotes: string;
}

interface OutfitSuggestion {
  colorPalette: string[];
  style: string;
  specificItems: string[];
  seasonalNotes: string;
}

interface Prop {
  name: string;
  purpose: string;
  whereToFind: string;
  optional: boolean;
}

interface Angle {
  description: string;
  visualExample?: string;
  technique: string;
  bestLighting: string;
}
```

---

## 🔐 Security Considerations

### API Security (MVP)
1. **Rate Limiting**:
   - 100 requests/hour per IP for chat endpoints
   - 20 image generations/hour per IP
   - Use `vercel-rate-limit` or similar

2. **Input Validation**:
   - Sanitize all user inputs
   - Validate JSON schemas with Zod
   - Prevent prompt injection attacks

3. **API Key Protection**:
   - Store OpenAI keys in environment variables
   - Never expose keys in client-side code
   - Rotate keys if compromised

4. **CORS**:
   - Restrict to production domain only
   - Development: Allow localhost:3000

### Example Rate Limiting
```typescript
import rateLimit from 'express-rate-limit';

const chatLimiter = rateLimit({
  windowMs: 60 * 60 * 1000, // 1 hour
  max: 100,
  message: 'Too many requests, please try again later.',
});

const imageLimiter = rateLimit({
  windowMs: 60 * 60 * 1000,
  max: 20,
  message: 'Image generation limit reached.',
});

// Apply to routes
app.use('/api/chat', chatLimiter);
app.use('/api/generate/image', imageLimiter);
```

---

## 🚀 Performance Optimization

### Performance Targets
| Metric | Target | Critical? |
|--------|--------|-----------|
| API Response Time (chat) | <3s | Yes |
| API Response Time (recommendations) | <5s | Yes |
| Image Generation | <15s | No (nice-to-have) |
| Page Load Time | <3s | Yes |
| Time to Interactive (TTI) | <5s | Yes |

### Optimization Strategies

#### 1. Caching
```typescript
// Client-side caching with React Query
const { data: destinations } = useQuery({
  queryKey: ['destinations', preferences],
  queryFn: () => fetchDestinations(preferences),
  staleTime: 1000 * 60 * 10, // 10 minutes
  cacheTime: 1000 * 60 * 30, // 30 minutes
});

// Server-side caching (optional Redis)
import redis from 'redis';

const cache = redis.createClient({
  url: process.env.REDIS_URL,
});

async function getCachedDestinations(preferencesHash: string) {
  const cached = await cache.get(`destinations:${preferencesHash}`);
  if (cached) return JSON.parse(cached);

  const destinations = await generateDestinations();
  await cache.setEx(`destinations:${preferencesHash}`, 3600, JSON.stringify(destinations));

  return destinations;
}
```

#### 2. Lazy Loading
```typescript
// Code splitting for heavy components
const ImageGenerator = lazy(() => import('@/components/ImageGenerator'));
const StylingRecommendations = lazy(() => import('@/components/StylingRecommendations'));

// Usage
<Suspense fallback={<LoadingSpinner />}>
  <ImageGenerator locationId={spotId} />
</Suspense>
```

#### 3. API Response Optimization
```typescript
// Stream responses for long-running operations
export async function POST(request: Request) {
  const encoder = new TextEncoder();
  const stream = new ReadableStream({
    async start(controller) {
      // Send progress updates
      controller.enqueue(encoder.encode('data: {"status": "processing", "progress": 25}\n\n'));

      const result = await generateRecommendations();

      controller.enqueue(encoder.encode(`data: ${JSON.stringify(result)}\n\n`));
      controller.close();
    },
  });

  return new Response(stream, {
    headers: {
      'Content-Type': 'text/event-stream',
      'Cache-Control': 'no-cache',
      'Connection': 'keep-alive',
    },
  });
}
```

#### 4. Image Optimization
```typescript
// Use Next.js Image component
import Image from 'next/image';

<Image
  src={imageUrl}
  alt="Location preview"
  width={1024}
  height={1024}
  loading="lazy"
  placeholder="blur"
  blurDataURL="data:image/..."
/>
```

---

## 🧪 Testing Strategy

### Testing Pyramid

```
        ┌────────────┐
        │    E2E     │  10% - Critical user flows
        │   Tests    │
        └────────────┘
       ┌──────────────┐
       │ Integration  │  30% - API endpoints, service integration
       │    Tests     │
       └──────────────┘
      ┌────────────────┐
      │  Unit Tests    │  60% - Pure functions, utilities
      └────────────────┘
```

### Unit Tests (Jest)
```typescript
// tests/services/recommendation.test.ts
import { generateDestinations } from '@/services/recommendation';

describe('Recommendation Service', () => {
  it('should generate 3 destinations for valid preferences', async () => {
    const preferences = {
      mood: 'romantic',
      aesthetic: 'vintage',
      duration: 'medium',
      interests: ['photography'],
    };

    const destinations = await generateDestinations(preferences);

    expect(destinations).toHaveLength(3);
    expect(destinations[0]).toHaveProperty('name');
    expect(destinations[0]).toHaveProperty('photographyScore');
  });

  it('should throw error for invalid preferences', async () => {
    const invalidPreferences = { mood: 'invalid' };

    await expect(generateDestinations(invalidPreferences)).rejects.toThrow();
  });
});
```

### Integration Tests (Supertest)
```typescript
// tests/api/chat.test.ts
import request from 'supertest';
import app from '@/app';

describe('POST /api/chat', () => {
  it('should process chat message and return next step', async () => {
    const response = await request(app)
      .post('/api/chat')
      .send({
        sessionId: 'test-session-1',
        message: 'I want a romantic trip',
        currentStep: 'mood',
        preferences: {},
      })
      .expect(200);

    expect(response.body).toHaveProperty('reply');
    expect(response.body).toHaveProperty('nextStep');
    expect(response.body.isComplete).toBe(false);
  });
});
```

### E2E Tests (Playwright)
```typescript
// e2e/user-flow.spec.ts
import { test, expect } from '@playwright/test';

test('complete user journey', async ({ page }) => {
  // 1. Landing page
  await page.goto('/');
  await expect(page.locator('h1')).toContainText('Trip Kit');

  // 2. Start chat
  await page.click('button:has-text("Start Planning")');
  await page.fill('input[placeholder="Type your message..."]', 'I want a romantic trip');
  await page.click('button:has-text("Send")');

  // 3. Wait for AI response
  await expect(page.locator('.ai-message')).toBeVisible({ timeout: 5000 });

  // 4. Complete conversation (simplified)
  await page.fill('input', 'vintage');
  await page.click('button:has-text("Send")');

  // 5. Select concept
  await page.click('div:has-text("Film Log")');

  // 6. View recommendations
  await expect(page.locator('.destination-card')).toHaveCount(3);

  // 7. Select destination and view spots
  await page.click('.destination-card:first-child');
  await expect(page.locator('.hidden-spot')).toHaveCount.greaterThanOrEqual(5);
});
```

---

## 📦 Deployment

### Vercel Deployment

#### Configuration (`vercel.json`)
```json
{
  "version": 2,
  "builds": [
    {
      "src": "package.json",
      "use": "@vercel/next"
    }
  ],
  "env": {
    "OPENAI_API_KEY": "@openai-api-key",
    "NODE_ENV": "production"
  },
  "functions": {
    "api/**": {
      "memory": 3008,
      "maxDuration": 60
    }
  }
}
```

#### Environment Variables
```bash
# .env.local (development)
OPENAI_API_KEY=sk-...
NEXT_PUBLIC_APP_URL=http://localhost:3000

# Production (Vercel dashboard)
OPENAI_API_KEY=sk-... (encrypted)
NEXT_PUBLIC_APP_URL=https://tripkit.vercel.app
```

#### Deployment Steps
```bash
# 1. Install Vercel CLI
npm install -g vercel

# 2. Login
vercel login

# 3. Deploy to preview
vercel

# 4. Deploy to production
vercel --prod

# 5. Set environment variables
vercel env add OPENAI_API_KEY production
```

---

## 📊 Monitoring & Observability

### Vercel Analytics
```typescript
// app/layout.tsx
import { Analytics } from '@vercel/analytics/react';

export default function RootLayout({ children }) {
  return (
    <html>
      <body>
        {children}
        <Analytics />
      </body>
    </html>
  );
}
```

### Error Tracking (Sentry - Optional)
```typescript
// sentry.config.ts
import * as Sentry from '@sentry/nextjs';

Sentry.init({
  dsn: process.env.SENTRY_DSN,
  tracesSampleRate: 0.1,
  environment: process.env.NODE_ENV,
});

// Usage in API routes
export async function POST(request: Request) {
  try {
    // ... logic
  } catch (error) {
    Sentry.captureException(error);
    return Response.json({ error: 'Internal error' }, { status: 500 });
  }
}
```

### Custom Metrics
```typescript
// lib/metrics.ts
export function trackRecommendationGenerated(destinationCount: number, duration: number) {
  if (typeof window !== 'undefined' && window.gtag) {
    window.gtag('event', 'recommendation_generated', {
      destination_count: destinationCount,
      generation_time_ms: duration,
    });
  }
}

export function trackImageGenerated(success: boolean, duration: number) {
  if (typeof window !== 'undefined' && window.gtag) {
    window.gtag('event', 'image_generated', {
      success,
      generation_time_ms: duration,
    });
  }
}
```

---

## 🔄 CI/CD Pipeline

### GitHub Actions Workflow

```yaml
# .github/workflows/ci-cd.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run linter
        run: npm run lint

      - name: Run type check
        run: npm run type-check

      - name: Run unit tests
        run: npm run test:unit

      - name: Run integration tests
        run: npm run test:integration
        env:
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}

  deploy-preview:
    needs: test
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: amondnet/vercel-action@v25
        with:
          vercel-token: ${{ secrets.VERCEL_TOKEN }}
          vercel-org-id: ${{ secrets.VERCEL_ORG_ID }}
          vercel-project-id: ${{ secrets.VERCEL_PROJECT_ID }}

  deploy-production:
    needs: test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: amondnet/vercel-action@v25
        with:
          vercel-token: ${{ secrets.VERCEL_TOKEN }}
          vercel-org-id: ${{ secrets.VERCEL_ORG_ID }}
          vercel-project-id: ${{ secrets.VERCEL_PROJECT_ID }}
          vercel-args: '--prod'
```

---

## 📝 Development Workflow

### Git Branch Strategy
```
main (production)
  ├── develop (staging)
  │   ├── feature/chat-interface
  │   ├── feature/image-generation
  │   └── feature/recommendations
  └── hotfix/critical-bug
```

### Commit Convention
```bash
# Format: <type>(<scope>): <subject>

feat(chat): implement LangGraph conversation flow
fix(api): resolve recommendation timeout issue
docs(readme): update API documentation
test(e2e): add complete user journey test
refactor(services): extract recommendation logic
```

### Development Scripts
```json
{
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "next lint",
    "type-check": "tsc --noEmit",
    "test": "jest --watch",
    "test:unit": "jest --testPathPattern=tests/unit",
    "test:integration": "jest --testPathPattern=tests/integration",
    "test:e2e": "playwright test",
    "test:coverage": "jest --coverage",
    "format": "prettier --write \"**/*.{ts,tsx,js,jsx,json,md}\"",
    "prepare": "husky install"
  }
}
```

---

## 🐛 Error Handling Strategy

### Error Types & Handling

```typescript
// lib/errors.ts
export class TripKitError extends Error {
  constructor(
    public code: string,
    public statusCode: number,
    message: string,
    public details?: unknown
  ) {
    super(message);
    this.name = 'TripKitError';
  }
}

export class OpenAIError extends TripKitError {
  constructor(message: string, details?: unknown) {
    super('OPENAI_ERROR', 503, message, details);
  }
}

export class ValidationError extends TripKitError {
  constructor(message: string, details?: unknown) {
    super('VALIDATION_ERROR', 400, message, details);
  }
}

export class RateLimitError extends TripKitError {
  constructor(message: string = 'Rate limit exceeded') {
    super('RATE_LIMIT', 429, message);
  }
}

// Global error handler middleware
export function errorHandler(error: unknown) {
  if (error instanceof TripKitError) {
    return Response.json(
      {
        error: error.code,
        message: error.message,
        details: error.details,
      },
      { status: error.statusCode }
    );
  }

  // Unexpected errors
  console.error('Unexpected error:', error);
  return Response.json(
    {
      error: 'INTERNAL_ERROR',
      message: 'An unexpected error occurred',
    },
    { status: 500 }
  );
}
```

### Retry Logic
```typescript
// lib/retry.ts
export async function retryWithBackoff<T>(
  fn: () => Promise<T>,
  maxRetries: number = 3,
  baseDelay: number = 1000
): Promise<T> {
  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error) {
      if (attempt === maxRetries) throw error;

      const delay = baseDelay * Math.pow(2, attempt);
      await new Promise(resolve => setTimeout(resolve, delay));
    }
  }
  throw new Error('Max retries exceeded');
}

// Usage
const destinations = await retryWithBackoff(
  () => openai.chat.completions.create({ ... }),
  3,
  1000
);
```

---

## 📚 External Dependencies

### Required APIs
| Service | Purpose | Pricing | Rate Limits |
|---------|---------|---------|-------------|
| **OpenAI GPT-4** | Chatbot, recommendations | ~$0.03/1K tokens | 10K requests/min |
| **OpenAI DALL-E 3** | Image generation | ~$0.04/image (1024x1024) | 50 requests/min |

### Optional Services (Post-MVP)
| Service | Purpose | Status |
|---------|---------|--------|
| **Google Maps API** | Coordinates, directions | Future |
| **Supabase** | User database, auth | Future |
| **Redis Cloud** | Session caching | Future |
| **Sentry** | Error tracking | Optional |

### NPM Packages
```json
{
  "dependencies": {
    "next": "^14.2.0",
    "react": "^18.3.0",
    "react-dom": "^18.3.0",
    "typescript": "^5.0.0",
    "tailwindcss": "^3.4.0",
    "zustand": "^4.5.0",
    "@tanstack/react-query": "^5.0.0",
    "openai": "^4.0.0",
    "langchain": "^0.1.0",
    "@langchain/langgraph": "^0.0.1",
    "zod": "^3.22.0",
    "axios": "^1.6.0"
  },
  "devDependencies": {
    "@types/node": "^20.0.0",
    "@types/react": "^18.3.0",
    "jest": "^29.7.0",
    "@testing-library/react": "^14.0.0",
    "@playwright/test": "^1.40.0",
    "eslint": "^8.0.0",
    "prettier": "^3.0.0",
    "husky": "^8.0.0"
  }
}
```

---

## 🗓️ Implementation Timeline (1 Week)

### Day 1-2: Foundation & Chatbot
- [ ] Next.js project setup with TypeScript
- [ ] Tailwind CSS configuration
- [ ] LangGraph conversation flow implementation
- [ ] `/api/chat` endpoint
- [ ] Basic UI: Landing page + Chat interface

### Day 3-4: Recommendations
- [ ] GPT-4 prompt engineering for destinations
- [ ] `/api/recommendations/destinations` endpoint
- [ ] `/api/recommendations/hidden-spots` endpoint
- [ ] Concept selection UI
- [ ] Destination cards UI
- [ ] Hidden spot gallery UI

### Day 5-6: Image Generation & Styling
- [ ] DALL-E 3 integration
- [ ] `/api/generate/image` endpoint (with async handling)
- [ ] `/api/recommendations/styling` endpoint
- [ ] Image generation UI with loading states
- [ ] Styling recommendations display
- [ ] Complete user flow integration

### Day 7: Testing & Polish
- [ ] End-to-end user flow testing
- [ ] Bug fixes
- [ ] Performance optimization
- [ ] Mobile responsiveness check
- [ ] Deploy to Vercel
- [ ] Documentation finalization

---

## 🔮 Future Enhancements (Post-MVP)

### Phase 2 (Weeks 2-4)
- User authentication (Supabase Auth)
- Save recommendations to user profile
- Share functionality (social media, export PDF)
- Admin dashboard for curated content

### Phase 3 (Months 2-3)
- O2O rental system integration
- Inventory management
- Payment processing (Stripe)
- Delivery logistics

### Phase 4 (Months 4-6)
- Machine learning recommendation improvements
- User-generated content (reviews, photos)
- Community features
- Mobile app (React Native)

---

## 📞 Technical Support Contacts

### Internal Team
- **Tech Lead**: [Contact Info]
- **Backend Engineer**: [Contact Info]
- **Frontend Engineer**: [Contact Info]
- **DevOps**: [Contact Info]

### External Services
- **OpenAI Support**: support@openai.com
- **Vercel Support**: support@vercel.com
- **Supabase Support**: support@supabase.io

---

## 📝 Appendix

### A. Glossary
- **LangGraph**: State machine framework for conversational AI
- **Film Aesthetic**: Visual style mimicking analog film cameras
- **Hidden Spots**: Non-mainstream locations favored by locals
- **O2O**: Online-to-Offline service model
- **Concept**: Predefined aesthetic/thematic travel style

### B. Reference Links
- [OpenAI API Documentation](https://platform.openai.com/docs)
- [LangGraph Documentation](https://langchain-ai.github.io/langgraph/)
- [Next.js Documentation](https://nextjs.org/docs)
- [Vercel Deployment Guide](https://vercel.com/docs)

### C. Revision History
| Version | Date | Changes | Author |
|---------|------|---------|--------|
| 1.0.0 | 2025-12-03 | Initial TRD for MVP | Engineering Team |

---

**Document Status**: ✅ Ready for Implementation
**Next Review**: End of Day 2 (Chatbot completion)
