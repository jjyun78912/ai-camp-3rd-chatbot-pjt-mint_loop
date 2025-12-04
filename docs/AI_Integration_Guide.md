# Trip Kit - AI Integration Guide
## LangGraph, OpenAI GPT-4, and DALL-E 3 Integration Patterns

---

## 📋 Document Information

- **Document Version**: 1.0.0
- **Last Updated**: 2025-12-03
- **Target Framework**: LangGraph + OpenAI SDK
- **Related Documents**: [TRD](./TRD_TripKit_MVP.md), [API Docs](./API_Documentation.md)

---

## 🎯 Overview

This guide provides implementation patterns for integrating Trip Kit with:
1. **LangGraph**: Stateful conversation management
2. **OpenAI GPT-4**: Intelligent recommendations and content generation
3. **OpenAI DALL-E 3**: AI image generation

---

## 🗣️ LangGraph Conversation System

### Architecture

LangGraph manages conversation state through a directed state graph where each node represents a conversation step.

```typescript
// State Definition
interface ConversationState {
  messages: Message[];
  currentStep: ConversationStep;
  preferences: UserPreferences;
  recommendations: Destination[] | null;
  sessionId: string;
  metadata: {
    startTime: number;
    totalMessages: number;
  };
}

type ConversationStep =
  | 'init'
  | 'mood'
  | 'aesthetic'
  | 'duration'
  | 'interests'
  | 'complete';

interface Message {
  role: 'user' | 'assistant' | 'system';
  content: string;
  timestamp: number;
}

interface UserPreferences {
  mood?: 'romantic' | 'adventurous' | 'nostalgic' | 'peaceful';
  aesthetic?: 'urban' | 'nature' | 'vintage' | 'modern';
  duration?: 'short' | 'medium' | 'long';
  interests?: Interest[];
  concept?: 'flaneur' | 'filmlog' | 'midnight';
}
```

### Implementation

#### 1. State Graph Setup

```typescript
// lib/langgraph/conversation-graph.ts
import { StateGraph, END } from '@langchain/langgraph';
import { ChatOpenAI } from 'langchain/chat_models/openai';
import { HumanMessage, AIMessage, SystemMessage } from 'langchain/schema';

// Initialize GPT-4 model
const model = new ChatOpenAI({
  modelName: 'gpt-4-turbo-preview',
  temperature: 0.7,
  openAIApiKey: process.env.OPENAI_API_KEY,
});

// Define state channels
const conversationChannels = {
  messages: {
    value: (x: Message[], y: Message[]) => x.concat(y),
    default: () => [],
  },
  currentStep: {
    value: (x: ConversationStep, y: ConversationStep) => y,
    default: () => 'init' as ConversationStep,
  },
  preferences: {
    value: (x: UserPreferences, y: UserPreferences) => ({ ...x, ...y }),
    default: () => ({}),
  },
  recommendations: {
    value: (x: Destination[] | null, y: Destination[] | null) => y,
    default: () => null,
  },
};

// Create state graph
const conversationGraph = new StateGraph({
  channels: conversationChannels,
});
```

#### 2. Define Conversation Nodes

```typescript
// lib/langgraph/nodes.ts

// System prompt for all nodes
const SYSTEM_PROMPT = `
You are a knowledgeable travel curator specializing in aesthetic, film photography-focused travel experiences.
Your goal is to understand the user's preferences through natural conversation.

Guidelines:
- Be warm, enthusiastic, and conversational
- Ask one question at a time
- Provide context for why you're asking
- Use natural language, not robotic
- Show excitement about travel possibilities
`;

// Init Node: Welcome user
async function initNode(state: ConversationState): Promise<Partial<ConversationState>> {
  const welcomeMessage = `
Hello! 👋 I'm your Trip Kit travel curator.

I'm here to help you discover hidden gems and create the perfect aesthetic travel experience.
Think film cameras, vintage vibes, and those Instagram-worthy moments that feel authentic.

To get started, tell me: **What kind of vibe are you feeling for your next trip?**

For example:
- "Something romantic and nostalgic"
- "Adventurous but artsy"
- "Peaceful and introspective"

What speaks to you?
  `.trim();

  return {
    messages: [
      {
        role: 'assistant',
        content: welcomeMessage,
        timestamp: Date.now(),
      },
    ],
    currentStep: 'mood',
  };
}

// Mood Node: Extract mood preference
async function moodNode(state: ConversationState): Promise<Partial<ConversationState>> {
  const userMessage = state.messages[state.messages.length - 1].content;

  // Use GPT-4 to extract mood and generate response
  const extractionPrompt = `
${SYSTEM_PROMPT}

User said: "${userMessage}"

Task:
1. Extract the user's mood/vibe from their message
2. Map to one of: romantic, adventurous, nostalgic, peaceful
3. Generate a warm, conversational response acknowledging their mood
4. Ask about their aesthetic preference (urban vs nature, vintage vs modern)

Respond in JSON format:
{
  "mood": "romantic|adventurous|nostalgic|peaceful",
  "reply": "Your conversational response with next question"
}
  `;

  const response = await model.invoke([
    new SystemMessage(extractionPrompt),
  ]);

  const parsed = JSON.parse(response.content);

  return {
    messages: [
      {
        role: 'assistant',
        content: parsed.reply,
        timestamp: Date.now(),
      },
    ],
    preferences: {
      mood: parsed.mood,
    },
    currentStep: 'aesthetic',
  };
}

// Aesthetic Node: Extract aesthetic preference
async function aestheticNode(state: ConversationState): Promise<Partial<ConversationState>> {
  const userMessage = state.messages[state.messages.length - 1].content;

  const extractionPrompt = `
${SYSTEM_PROMPT}

User's mood: ${state.preferences.mood}
User said: "${userMessage}"

Task:
1. Extract aesthetic preference (urban/nature, vintage/modern)
2. Map to one of: urban, nature, vintage, modern
3. Generate response acknowledging aesthetic
4. Ask about trip duration (quick weekend, week-long, extended)

Respond in JSON:
{
  "aesthetic": "urban|nature|vintage|modern",
  "reply": "Your response with duration question"
}
  `;

  const response = await model.invoke([new SystemMessage(extractionPrompt)]);
  const parsed = JSON.parse(response.content);

  return {
    messages: [{ role: 'assistant', content: parsed.reply, timestamp: Date.now() }],
    preferences: { aesthetic: parsed.aesthetic },
    currentStep: 'duration',
  };
}

// Duration Node: Extract trip duration
async function durationNode(state: ConversationState): Promise<Partial<ConversationState>> {
  const userMessage = state.messages[state.messages.length - 1].content;

  const extractionPrompt = `
${SYSTEM_PROMPT}

User's preferences so far:
- Mood: ${state.preferences.mood}
- Aesthetic: ${state.preferences.aesthetic}

User said: "${userMessage}"

Task:
1. Extract trip duration
2. Map to: short (1-3 days), medium (4-7 days), long (8+ days)
3. Generate response
4. Ask about specific interests (photography, food, art, history, nature, architecture)

Respond in JSON:
{
  "duration": "short|medium|long",
  "reply": "Your response with interests question"
}
  `;

  const response = await model.invoke([new SystemMessage(extractionPrompt)]);
  const parsed = JSON.parse(response.content);

  return {
    messages: [{ role: 'assistant', content: parsed.reply, timestamp: Date.now() }],
    preferences: { duration: parsed.duration },
    currentStep: 'interests',
  };
}

// Interests Node: Extract interests and complete conversation
async function interestsNode(state: ConversationState): Promise<Partial<ConversationState>> {
  const userMessage = state.messages[state.messages.length - 1].content;

  const extractionPrompt = `
${SYSTEM_PROMPT}

User's preferences so far:
- Mood: ${state.preferences.mood}
- Aesthetic: ${state.preferences.aesthetic}
- Duration: ${state.preferences.duration}

User said: "${userMessage}"

Task:
1. Extract interests (can be multiple)
2. Map to: photography, food, art, history, nature, architecture
3. Generate enthusiastic closing message summarizing their preferences
4. Let them know you're generating personalized recommendations

Respond in JSON:
{
  "interests": ["photography", "food", ...],
  "reply": "Your enthusiastic summary and transition to recommendations"
}
  `;

  const response = await model.invoke([new SystemMessage(extractionPrompt)]);
  const parsed = JSON.parse(response.content);

  // Generate recommendations
  const recommendations = await generateDestinationRecommendations({
    ...state.preferences,
    interests: parsed.interests,
  });

  return {
    messages: [{ role: 'assistant', content: parsed.reply, timestamp: Date.now() }],
    preferences: { interests: parsed.interests },
    recommendations,
    currentStep: 'complete',
  };
}

// Add nodes to graph
conversationGraph.addNode('init', initNode);
conversationGraph.addNode('mood', moodNode);
conversationGraph.addNode('aesthetic', aestheticNode);
conversationGraph.addNode('duration', durationNode);
conversationGraph.addNode('interests', interestsNode);

// Define edges (transitions)
conversationGraph.addEdge('init', 'mood');
conversationGraph.addEdge('mood', 'aesthetic');
conversationGraph.addEdge('aesthetic', 'duration');
conversationGraph.addEdge('duration', 'interests');
conversationGraph.addEdge('interests', END);

// Set entry point
conversationGraph.setEntryPoint('init');

// Compile graph
export const chatbot = conversationGraph.compile();
```

#### 3. API Integration

```typescript
// app/api/chat/route.ts
import { chatbot } from '@/lib/langgraph/conversation-graph';
import { ConversationState } from '@/lib/langgraph/types';

export async function POST(request: Request) {
  try {
    const { sessionId, message, currentStep, preferences } = await request.json();

    // Initialize or restore state
    let state: ConversationState = {
      messages: [],
      currentStep: currentStep || 'init',
      preferences: preferences || {},
      recommendations: null,
      sessionId,
      metadata: {
        startTime: Date.now(),
        totalMessages: 0,
      },
    };

    // If user sent a message, add it to state
    if (message) {
      state.messages.push({
        role: 'user',
        content: message,
        timestamp: Date.now(),
      });
    }

    // Run chatbot graph
    const result = await chatbot.invoke(state);

    // Extract latest assistant message
    const latestMessage = result.messages[result.messages.length - 1];

    return Response.json({
      reply: latestMessage.content,
      nextStep: result.currentStep,
      isComplete: result.currentStep === 'complete',
      recommendations: result.recommendations,
      sessionId,
    });
  } catch (error) {
    console.error('Chat error:', error);
    return Response.json(
      { error: 'CHAT_ERROR', message: 'Failed to process conversation' },
      { status: 500 }
    );
  }
}
```

### Advanced: State Persistence

For production, persist conversation state across requests:

```typescript
// lib/storage/conversation-store.ts
import { createClient } from 'redis';

const redis = createClient({
  url: process.env.REDIS_URL,
});

await redis.connect();

export async function saveConversationState(
  sessionId: string,
  state: ConversationState
): Promise<void> {
  await redis.set(
    `conversation:${sessionId}`,
    JSON.stringify(state),
    { EX: 3600 } // Expire after 1 hour
  );
}

export async function loadConversationState(
  sessionId: string
): Promise<ConversationState | null> {
  const data = await redis.get(`conversation:${sessionId}`);
  return data ? JSON.parse(data) : null;
}

// Use in API route
const savedState = await loadConversationState(sessionId);
const state = savedState || initializeNewState(sessionId);

// After processing
await saveConversationState(sessionId, result);
```

---

## 🧠 OpenAI GPT-4 Integration

### Destination Recommendation System

#### Prompt Engineering Strategy

```typescript
// lib/ai/prompts/destination-recommendations.ts

export function buildDestinationPrompt(preferences: UserPreferences): string {
  const { mood, aesthetic, duration, interests, concept } = preferences;

  return `
You are an expert travel curator with deep knowledge of hidden, non-touristy destinations worldwide.

User Preferences:
- Mood/Vibe: ${mood} (${getMoodDescription(mood)})
- Aesthetic: ${aesthetic} (${getAestheticDescription(aesthetic)})
- Trip Duration: ${duration} (${getDurationDescription(duration)})
- Interests: ${interests.join(', ')}
${concept ? `- Selected Concept: ${concept} (${getConceptDescription(concept)})` : ''}

Task: Generate exactly 3 unique destination recommendations that:

1. **Are NOT mainstream tourist destinations** (avoid top-10 lists, cruise ship ports, overly commercialized areas)
2. **Highly photogenic** with strong aesthetic appeal for film photography
3. **Match the user's mood and interests** closely
4. **Accessible by public transportation** (no remote islands requiring private boats)
5. **Safe for solo travelers** (safety rating ≥7/10)
6. **Align with the selected concept** aesthetic (if provided)

For each destination, provide:

- **name**: Specific area/neighborhood, not just city name (e.g., "Alfama District" not "Lisbon")
- **city**: City name
- **country**: Country name
- **description**: 2-3 sentences describing the area, atmosphere, and unique characteristics (100-150 words)
- **matchReason**: Explain specifically why this matches the user's preferences (50-100 words)
- **bestTimeToVisit**: Specific months or season with reasoning
- **photographyScore**: 1-10 rating for photography potential
- **transportAccessibility**: "easy" (metro/bus readily available), "moderate" (regional trains), or "challenging"
- **safetyRating**: 1-10 rating
- **estimatedBudget**: "$" (budget-friendly), "$$" (moderate), "$$$" (expensive)
- **tags**: Array of 3-5 descriptive keywords

Output as valid JSON array with 3 objects. No additional text.

Example output structure:
\`\`\`json
[
  {
    "name": "Porto Ribeira District",
    "city": "Porto",
    "country": "Portugal",
    "description": "...",
    "matchReason": "...",
    "bestTimeToVisit": "May - June",
    "photographyScore": 8,
    "transportAccessibility": "easy",
    "safetyRating": 9,
    "estimatedBudget": "$",
    "tags": ["riverside", "wine", "architecture"]
  }
]
\`\`\`
`.trim();
}

// Helper functions
function getMoodDescription(mood: string): string {
  const descriptions = {
    romantic: 'Intimate, dreamy atmosphere with charm',
    adventurous: 'Exciting exploration and discovery',
    nostalgic: 'Timeless, classic, evokes memories',
    peaceful: 'Calm, serene, meditative spaces',
  };
  return descriptions[mood] || mood;
}

function getAestheticDescription(aesthetic: string): string {
  const descriptions = {
    urban: 'City streets, architecture, cultural hubs',
    nature: 'Landscapes, natural beauty, outdoor settings',
    vintage: 'Retro, historic, timeless character',
    modern: 'Contemporary, sleek, innovative design',
  };
  return descriptions[aesthetic] || aesthetic;
}

function getDurationDescription(duration: string): string {
  const descriptions = {
    short: '1-3 days, quick getaway',
    medium: '4-7 days, balanced exploration',
    long: '8+ days, deep immersion',
  };
  return descriptions[duration] || duration;
}

function getConceptDescription(concept: string): string {
  const descriptions = {
    flaneur: 'Urban wanderer, literary aesthetic, slow exploration',
    filmlog: 'Vintage photography, retro analog feel, nostalgic documentation',
    midnight: 'Artistic time travel, bohemian spirit, cultural depth',
  };
  return descriptions[concept] || concept;
}
```

#### Implementation

```typescript
// lib/ai/services/recommendation-service.ts
import OpenAI from 'openai';
import { buildDestinationPrompt } from '../prompts/destination-recommendations';

const openai = new OpenAI({
  apiKey: process.env.OPENAI_API_KEY,
});

export async function generateDestinationRecommendations(
  preferences: UserPreferences
): Promise<Destination[]> {
  try {
    const prompt = buildDestinationPrompt(preferences);

    const response = await openai.chat.completions.create({
      model: 'gpt-4-turbo-preview',
      messages: [
        {
          role: 'system',
          content: 'You are a travel curation expert specializing in hidden destinations.',
        },
        {
          role: 'user',
          content: prompt,
        },
      ],
      temperature: 0.8, // Higher creativity for diverse recommendations
      max_tokens: 2000,
      response_format: { type: 'json_object' }, // Ensure JSON output
    });

    const content = response.choices[0].message.content;
    const destinations = JSON.parse(content);

    // Validate and add IDs
    return destinations.map((dest: any, index: number) => ({
      id: `dest_${Date.now()}_${index}`,
      ...dest,
    }));
  } catch (error) {
    console.error('Recommendation generation error:', error);
    throw new Error('Failed to generate recommendations');
  }
}

// API Route
export async function POST(request: Request) {
  try {
    const { preferences } = await request.json();

    const destinations = await generateDestinationRecommendations(preferences);

    return Response.json({
      destinations,
      generatedAt: new Date().toISOString(),
      preferences,
    });
  } catch (error) {
    return Response.json(
      { error: 'RECOMMENDATION_ERROR', message: error.message },
      { status: 500 }
    );
  }
}
```

### Hidden Spots Generation

```typescript
// lib/ai/prompts/hidden-spots.ts

export function buildHiddenSpotsPrompt(
  destination: Destination,
  concept: string,
  preferences: Partial<UserPreferences>
): string {
  return `
You are a local insider with deep knowledge of ${destination.name} in ${destination.city}, ${destination.country}.

Context:
- Destination: ${destination.name}
- Selected Concept: ${concept}
- User Interests: ${preferences.interests?.join(', ')}
- Photography Focus: High priority

Task: Generate 7-10 **hidden, local-favorite locations** within ${destination.name} that:

1. **Are NOT major tourist attractions** (avoid guidebook clichés, TripAdvisor top lists)
2. **Highly photogenic** for film aesthetic photography
3. **Match the "${concept}" concept** aesthetic and vibe
4. **Known by locals** but relatively unknown to tourists
5. **Accessible** (no private property or dangerous areas)
6. **Diverse** (mix of viewpoints, streets, cafes, cultural spots)

For each location:

- **name**: Specific location name (street corner, café, viewpoint, etc.)
- **address**: Full address or detailed location description
- **coordinates**: {lat, lng} (if known, otherwise null)
- **description**: 50-100 words about the location, atmosphere, and why locals love it
- **photographyTips**: Array of 3-5 specific tips (angles, lighting, composition)
- **bestTimeToVisit**: Specific time of day or season
- **estimatedDuration**: How long to spend there ("30min", "1-2hr")
- **nearbyAmenities**: Cafes, restrooms, shops within 5min walk
- **accessibilityNotes**: How to get there, any physical challenges
- **crowdLevel**: "low", "medium", "high"
- **localTip**: Insider advice that only locals would know
- **filmRecommendations**: Array of 1-2 film stocks that work well here with reasoning
- **safetyNotes**: Any safety considerations

Output as valid JSON array. No additional text.

Example structure:
\`\`\`json
[
  {
    "name": "Secret overlook above Via dell'Amore",
    "address": "Path 2, Riomaggiore to Manarola",
    "coordinates": {"lat": 44.0996, "lng": 9.7368},
    "description": "...",
    "photographyTips": ["Golden hour: 30min before sunset", "Use f/2.8 for bokeh"],
    "bestTimeToVisit": "Sunset (7:00 PM)",
    "estimatedDuration": "45min - 1hr",
    "nearbyAmenities": ["Trattoria dal Billy (5min walk)"],
    "accessibilityNotes": "10min uphill hike, uneven terrain",
    "crowdLevel": "low",
    "localTip": "Visit on weekdays to avoid tourists",
    "filmRecommendations": [
      {"filmStock": "Kodak ColorPlus 200", "reason": "Captures warm coastal light"}
    ],
    "safetyNotes": "Stay on marked paths. Cliff edges not fenced."
  }
]
\`\`\`
`.trim();
}

// Service implementation
export async function generateHiddenSpots(
  destination: Destination,
  concept: string,
  preferences: Partial<UserPreferences>
): Promise<HiddenSpot[]> {
  const prompt = buildHiddenSpotsPrompt(destination, concept, preferences);

  const response = await openai.chat.completions.create({
    model: 'gpt-4-turbo-preview',
    messages: [
      { role: 'system', content: 'You are a local travel insider and photographer.' },
      { role: 'user', content: prompt },
    ],
    temperature: 0.9, // High creativity for unique spots
    max_tokens: 3000,
    response_format: { type: 'json_object' },
  });

  const spots = JSON.parse(response.choices[0].message.content);

  return spots.map((spot: any, index: number) => ({
    id: `spot_${Date.now()}_${index}`,
    ...spot,
  }));
}
```

---

## 🎨 DALL-E 3 Image Generation

### Prompt Engineering for Film Aesthetic

```typescript
// lib/ai/prompts/image-generation.ts

interface ImageGenerationRequest {
  locationName: string;
  locationDescription: string;
  filmStock: string;
  outfitStyle: string;
  timeOfDay: string;
  composition: string;
  expression: string;
}

const FILM_STOCK_PROFILES = {
  kodak_colorplus: {
    name: 'Kodak ColorPlus 200',
    characteristics: 'Warm, saturated tones with slight red-orange shift, fine grain',
    colorProfile: 'Warm golden tones, enhanced reds and oranges, slight vignetting',
    referencePhotographers: 'William Eggleston color palette, Stephen Shore saturation',
  },
  kodak_portra: {
    name: 'Kodak Portra 400',
    characteristics: 'Natural skin tones, subtle colors, smooth grain structure',
    colorProfile: 'Neutral-warm tones, excellent skin rendering, subtle pastels',
    referencePhotographers: 'Rinko Kawauchi softness, Ryan Muirhead natural light',
  },
  fuji_superia: {
    name: 'Fujifilm Superia 400',
    characteristics: 'Vibrant colors, enhanced greens and blues, moderate grain',
    colorProfile: 'Cool-neutral tones, punchy colors, slight cyan shift in shadows',
    referencePhotographers: 'Alex Webb color intensity, Saul Leiter mood',
  },
  ilford_hp5: {
    name: 'Ilford HP5 Plus 400',
    characteristics: 'Classic black and white, high contrast, visible grain texture',
    colorProfile: 'Monochrome with rich blacks, moderate contrast, organic grain',
    referencePhotographers: 'Henri Cartier-Bresson composition, Fan Ho contrast',
  },
};

export function buildImageGenerationPrompt(request: ImageGenerationRequest): string {
  const filmProfile = FILM_STOCK_PROFILES[request.filmStock] || FILM_STOCK_PROFILES.kodak_colorplus;

  return `
Create a hyper-realistic photograph that authentically replicates the look of ${filmProfile.name} film photography.

SCENE DESCRIPTION:
${request.locationDescription}
Location: ${request.locationName}

SUBJECT:
- A young person (gender-neutral, 20s-30s) wearing ${request.outfitStyle}
- Holding a vintage 35mm film camera (Canon AE-1 or similar)
- Natural, candid pose (NOT staged or overly posed)
- Expression: ${request.expression}
- Looking ${getGazeDirection(request.composition)} with authentic emotion

FILM AESTHETIC - ${filmProfile.name}:
- Color Profile: ${filmProfile.colorProfile}
- Grain Texture: ${filmProfile.characteristics}
- Reference Style: ${filmProfile.referencePhotographers}
- Slight vignetting (darker corners)
- Natural film imperfections (subtle dust specs, slight color shifts)
- Authentic analog depth and dimension

COMPOSITION:
- Subject positioned in ${getCompositionDescription(request.composition)}
- Background elements: ${getBackgroundGuidance(request.locationDescription)}
- Depth of field: f/1.8-2.8 (sharp subject, beautifully blurred background)
- Focal length feel: 35mm or 50mm lens perspective

LIGHTING - ${request.timeOfDay}:
${getLightingDescription(request.timeOfDay)}

TECHNICAL SPECIFICATIONS:
- Style: Professional film photography, cinematic, analog warmth
- Quality: High detail, authentic grain texture
- Atmosphere: Nostalgic, timeless, emotionally resonant
- NO digital sharpness, NO HDR look, NO oversaturation
- YES organic film grain, YES natural color shifts, YES authentic analog feel

MOOD: Intimate, genuine moment captured on film. Should feel like a frame from a personal travel documentary shot by a skilled film photographer in the ${getDecadeReference(request.filmStock)}.

Important: This should look like it was actually shot on ${filmProfile.name} film, scanned at high resolution. Not a digital photo with a filter applied.
`.trim();
}

function getGazeDirection(composition: string): string {
  const directions = {
    'left-third': 'towards the right (out of frame)',
    'right-third': 'towards the left (out of frame)',
    center: 'directly at camera with soft expression',
  };
  return directions[composition] || 'naturally into the scene';
}

function getCompositionDescription(composition: string): string {
  const descriptions = {
    'left-third': 'left third of frame (rule of thirds)',
    'right-third': 'right third of frame (rule of thirds)',
    center: 'center of frame with balanced negative space',
  };
  return descriptions[composition] || 'right third of frame';
}

function getBackgroundGuidance(locationDescription: string): string {
  return `${locationDescription.substring(0, 100)}... (softly blurred with beautiful bokeh)`;
}

function getLightingDescription(timeOfDay: string): string {
  const lighting = {
    morning: `
- Soft, cool morning light (6:30-8:00 AM feel)
- Gentle directional light from low sun angle
- Slight haze in atmosphere
- Cool-neutral color temperature
- Long, soft shadows
    `.trim(),
    noon: `
- Bright, even overhead light
- Find shade or diffused light to avoid harsh shadows
- Warm-neutral color temperature
- Slightly higher contrast
    `.trim(),
    sunset: `
- Warm, golden light (golden hour: 30min before sunset)
- Directional warm glow from low sun
- Rich golden-orange tones
- Long dramatic shadows
- Slight backlight creating rim light on subject
    `.trim(),
    night: `
- Low ambient light with practical light sources
- Warm tungsten or cool fluorescent light mix
- Slightly underexposed for mood
- Natural grain more visible
- High ISO film aesthetic (pushed film look)
    `.trim(),
  };
  return lighting[timeOfDay] || lighting.sunset;
}

function getDecadeReference(filmStock: string): string {
  const decades = {
    kodak_colorplus: '1980s-90s (classic American color film era)',
    kodak_portra: '2000s-2010s (modern fine art film photography)',
    fuji_superia: '1990s-2000s (Japanese travel photography era)',
    ilford_hp5: '1960s-70s (classic photojournalism era)',
  };
  return decades[filmStock] || '1980s-90s';
}
```

### DALL-E 3 API Integration

```typescript
// lib/ai/services/image-generation-service.ts
import OpenAI from 'openai';
import { buildImageGenerationPrompt } from '../prompts/image-generation';

const openai = new OpenAI({
  apiKey: process.env.OPENAI_API_KEY,
});

export async function generateLocationImage(
  request: ImageGenerationRequest
): Promise<ImageGenerationResponse> {
  const startTime = Date.now();

  try {
    const prompt = buildImageGenerationPrompt(request);

    const response = await openai.images.generate({
      model: 'dall-e-3',
      prompt,
      n: 1,
      size: '1024x1024',
      quality: 'hd', // Use HD quality for better detail
      style: 'natural', // Natural (photorealistic) vs vivid (artistic)
    });

    const imageUrl = response.data[0].url;
    const revisedPrompt = response.data[0].revised_prompt; // DALL-E may revise prompt

    return {
      imageUrl,
      prompt,
      generationTime: Date.now() - startTime,
      status: 'success',
      metadata: {
        model: 'dall-e-3',
        size: '1024x1024',
        quality: 'hd',
        revisedPrompt,
      },
    };
  } catch (error) {
    console.error('Image generation error:', error);

    return {
      imageUrl: '',
      prompt: buildImageGenerationPrompt(request),
      generationTime: Date.now() - startTime,
      status: 'failed',
      errorMessage: error.message,
      errorCode: getErrorCode(error),
    };
  }
}

function getErrorCode(error: any): string {
  if (error.message?.includes('content_policy')) return 'CONTENT_POLICY_VIOLATION';
  if (error.message?.includes('rate_limit')) return 'RATE_LIMIT_EXCEEDED';
  if (error.message?.includes('billing')) return 'BILLING_ERROR';
  return 'GENERATION_ERROR';
}

// API Route with async handling
export async function POST(request: Request) {
  try {
    const body = await request.json();

    // Validate request
    if (!body.locationId || !body.filmStock) {
      return Response.json(
        { error: 'VALIDATION_ERROR', message: 'Missing required fields' },
        { status: 400 }
      );
    }

    // Check if generation should be async (based on load)
    const shouldRunAsync = await shouldUseAsyncGeneration();

    if (shouldRunAsync) {
      // Queue for async processing
      const taskId = await queueImageGeneration(body);

      return Response.json(
        {
          taskId,
          status: 'pending',
          estimatedWait: 15000,
          pollUrl: `/api/generate/image/${taskId}`,
        },
        { status: 202 }
      );
    }

    // Synchronous generation
    const result = await generateLocationImage(body);

    return Response.json(result);
  } catch (error) {
    return Response.json(
      { error: 'IMAGE_GENERATION_ERROR', message: error.message },
      { status: 500 }
    );
  }
}
```

### Async Task Queue (Optional)

```typescript
// lib/queue/image-generation-queue.ts
import { Queue, Worker } from 'bullmq';

const imageQueue = new Queue('image-generation', {
  connection: {
    host: process.env.REDIS_HOST,
    port: parseInt(process.env.REDIS_PORT || '6379'),
  },
});

export async function queueImageGeneration(
  request: ImageGenerationRequest
): Promise<string> {
  const job = await imageQueue.add('generate-image', request, {
    attempts: 3,
    backoff: {
      type: 'exponential',
      delay: 2000,
    },
  });

  return job.id;
}

// Worker to process queue
const worker = new Worker(
  'image-generation',
  async job => {
    const result = await generateLocationImage(job.data);
    return result;
  },
  {
    connection: {
      host: process.env.REDIS_HOST,
      port: parseInt(process.env.REDIS_PORT || '6379'),
    },
  }
);

// Poll endpoint
export async function GET(request: Request, { params }: { params: { taskId: string } }) {
  const job = await imageQueue.getJob(params.taskId);

  if (!job) {
    return Response.json(
      { error: 'TASK_NOT_FOUND', message: 'Invalid task ID' },
      { status: 404 }
    );
  }

  const state = await job.getState();

  if (state === 'completed') {
    const result = job.returnvalue;
    return Response.json({
      taskId: params.taskId,
      status: 'success',
      ...result,
    });
  }

  if (state === 'failed') {
    return Response.json({
      taskId: params.taskId,
      status: 'failed',
      errorMessage: job.failedReason,
    });
  }

  return Response.json({
    taskId: params.taskId,
    status: 'pending',
    progress: job.progress || 0,
  });
}
```

---

## 🛠️ Best Practices

### 1. Error Handling

```typescript
// lib/ai/utils/error-handler.ts

export class AIServiceError extends Error {
  constructor(
    public service: 'openai' | 'langgraph',
    public code: string,
    message: string,
    public details?: any
  ) {
    super(message);
    this.name = 'AIServiceError';
  }
}

export async function retryWithBackoff<T>(
  fn: () => Promise<T>,
  maxRetries: number = 3,
  baseDelay: number = 1000
): Promise<T> {
  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error) {
      if (attempt === maxRetries) {
        throw new AIServiceError(
          'openai',
          'MAX_RETRIES_EXCEEDED',
          `Failed after ${maxRetries} attempts`,
          error
        );
      }

      // Exponential backoff
      const delay = baseDelay * Math.pow(2, attempt);
      await new Promise(resolve => setTimeout(resolve, delay));
    }
  }

  throw new Error('Unexpected error in retry logic');
}

// Usage
const destinations = await retryWithBackoff(
  () => generateDestinationRecommendations(preferences),
  3,
  1000
);
```

### 2. Rate Limiting & Cost Management

```typescript
// lib/ai/utils/rate-limiter.ts
import { Ratelimit } from '@upstash/ratelimit';
import { Redis } from '@upstash/redis';

const redis = new Redis({
  url: process.env.UPSTASH_REDIS_REST_URL!,
  token: process.env.UPSTASH_REDIS_REST_TOKEN!,
});

// Chat endpoint: 100 requests per hour per IP
export const chatRateLimiter = new Ratelimit({
  redis,
  limiter: Ratelimit.slidingWindow(100, '1 h'),
  analytics: true,
});

// Image generation: 20 requests per hour per IP
export const imageRateLimiter = new Ratelimit({
  redis,
  limiter: Ratelimit.slidingWindow(20, '1 h'),
  analytics: true,
});

// Usage in API route
const identifier = getIpAddress(request);
const { success, limit, remaining, reset } = await imageRateLimiter.limit(identifier);

if (!success) {
  return Response.json(
    {
      error: 'RATE_LIMIT_EXCEEDED',
      message: 'Too many requests. Please try again later.',
      retryAfter: reset - Date.now(),
    },
    {
      status: 429,
      headers: {
        'X-RateLimit-Limit': limit.toString(),
        'X-RateLimit-Remaining': remaining.toString(),
        'X-RateLimit-Reset': reset.toString(),
      },
    }
  );
}
```

### 3. Caching Strategy

```typescript
// lib/ai/utils/cache.ts

// Cache recommendations to reduce API calls
export async function getCachedRecommendations(
  preferences: UserPreferences
): Promise<Destination[] | null> {
  const cacheKey = `recommendations:${hashPreferences(preferences)}`;

  const cached = await redis.get(cacheKey);
  if (cached) {
    console.log('Cache hit for recommendations');
    return JSON.parse(cached);
  }

  return null;
}

export async function setCachedRecommendations(
  preferences: UserPreferences,
  recommendations: Destination[]
): Promise<void> {
  const cacheKey = `recommendations:${hashPreferences(preferences)}`;

  await redis.set(cacheKey, JSON.stringify(recommendations), {
    ex: 3600, // Expire after 1 hour
  });
}

function hashPreferences(preferences: UserPreferences): string {
  return createHash('md5')
    .update(JSON.stringify(preferences))
    .digest('hex');
}

// Usage
const cached = await getCachedRecommendations(preferences);
if (cached) return cached;

const fresh = await generateDestinationRecommendations(preferences);
await setCachedRecommendations(preferences, fresh);
return fresh;
```

### 4. Prompt Versioning

```typescript
// lib/ai/prompts/versions.ts

export const PROMPT_VERSIONS = {
  destinationRecommendation: {
    v1: buildDestinationPromptV1,
    v2: buildDestinationPromptV2,
    current: 'v2',
  },
  imageGeneration: {
    v1: buildImagePromptV1,
    v2: buildImagePromptV2,
    current: 'v2',
  },
};

export function getPromptVersion(
  category: keyof typeof PROMPT_VERSIONS,
  version?: string
) {
  const versions = PROMPT_VERSIONS[category];
  const selectedVersion = version || versions.current;
  return versions[selectedVersion];
}

// Usage
const promptBuilder = getPromptVersion('destinationRecommendation', 'v2');
const prompt = promptBuilder(preferences);
```

### 5. Monitoring & Logging

```typescript
// lib/ai/utils/logger.ts

interface AIMetric {
  service: 'openai' | 'langgraph';
  operation: string;
  duration: number;
  success: boolean;
  tokenUsage?: {
    prompt: number;
    completion: number;
    total: number;
  };
  cost?: number;
  error?: string;
}

export async function logAIMetric(metric: AIMetric): Promise<void> {
  console.log('[AI Metric]', JSON.stringify(metric));

  // Send to analytics service (e.g., PostHog, Mixpanel)
  if (typeof window !== 'undefined' && window.analytics) {
    window.analytics.track('ai_operation', metric);
  }

  // Store in database for analysis
  await db.aiMetrics.create({ data: metric });
}

// Usage wrapper
export async function withMetrics<T>(
  service: 'openai' | 'langgraph',
  operation: string,
  fn: () => Promise<T>
): Promise<T> {
  const startTime = Date.now();

  try {
    const result = await fn();

    await logAIMetric({
      service,
      operation,
      duration: Date.now() - startTime,
      success: true,
    });

    return result;
  } catch (error) {
    await logAIMetric({
      service,
      operation,
      duration: Date.now() - startTime,
      success: false,
      error: error.message,
    });

    throw error;
  }
}

// Usage
const recommendations = await withMetrics(
  'openai',
  'generate_destinations',
  () => generateDestinationRecommendations(preferences)
);
```

---

## 📊 Cost Estimation

### OpenAI Pricing (as of Dec 2025)

| Model | Input | Output | Use Case |
|-------|-------|--------|----------|
| GPT-4 Turbo | $0.01/1K tokens | $0.03/1K tokens | Recommendations, conversation |
| DALL-E 3 (1024x1024, HD) | N/A | $0.080/image | Image generation |

### Estimated Costs per User Journey

**Complete User Flow**:
- Chat conversation (5-7 exchanges): ~3K tokens input, ~2K output = $0.09
- Destination recommendations: ~1K input, ~1K output = $0.04
- Hidden spots (1 destination): ~1K input, ~2K output = $0.07
- Image generation (1 image): $0.08
- Styling recommendations: ~1K input, ~1.5K output = $0.055

**Total per user**: ~$0.335

**Monthly cost for 1,000 users**: ~$335

### Cost Optimization Strategies

1. **Caching**: Reduce repeated recommendations by 60-70%
2. **Prompt optimization**: Reduce token usage by 20-30%
3. **Lazy image generation**: Only generate when user explicitly requests
4. **Rate limiting**: Prevent abuse and excessive costs

---

## 🧪 Testing AI Integrations

### Unit Tests

```typescript
// tests/ai/services/recommendation-service.test.ts
import { generateDestinationRecommendations } from '@/lib/ai/services/recommendation-service';

describe('Recommendation Service', () => {
  it('should generate 3 destinations', async () => {
    const preferences = {
      mood: 'romantic',
      aesthetic: 'vintage',
      duration: 'medium',
      interests: ['photography'],
    };

    const destinations = await generateDestinationRecommendations(preferences);

    expect(destinations).toHaveLength(3);
    expect(destinations[0]).toHaveProperty('name');
    expect(destinations[0]).toHaveProperty('photographyScore');
    expect(destinations[0].photographyScore).toBeGreaterThanOrEqual(1);
    expect(destinations[0].photographyScore).toBeLessThanOrEqual(10);
  });

  it('should include concept in recommendations', async () => {
    const preferences = {
      mood: 'nostalgic',
      aesthetic: 'vintage',
      duration: 'short',
      interests: ['art'],
      concept: 'filmlog',
    };

    const destinations = await generateDestinationRecommendations(preferences);

    expect(destinations[0].matchReason).toContain('film');
  });
});
```

### Mock AI Responses

```typescript
// tests/mocks/ai-mocks.ts

export function mockOpenAI() {
  jest.mock('openai', () => {
    return {
      default: jest.fn().mockImplementation(() => ({
        chat: {
          completions: {
            create: jest.fn().mockResolvedValue({
              choices: [
                {
                  message: {
                    content: JSON.stringify([
                      {
                        name: 'Test Destination',
                        city: 'Test City',
                        country: 'Test Country',
                        description: 'Test description',
                        matchReason: 'Test reason',
                        photographyScore: 8,
                      },
                    ]),
                  },
                },
              ],
            }),
          },
        },
        images: {
          generate: jest.fn().mockResolvedValue({
            data: [
              {
                url: 'https://example.com/test-image.png',
                revised_prompt: 'Test revised prompt',
              },
            ],
          }),
        },
      })),
    };
  });
}
```

---

## 📚 Additional Resources

### Official Documentation
- [OpenAI API Reference](https://platform.openai.com/docs/api-reference)
- [LangGraph Documentation](https://langchain-ai.github.io/langgraph/)
- [LangChain Documentation](https://python.langchain.com/docs/get_started/introduction)

### Prompt Engineering
- [OpenAI Prompt Engineering Guide](https://platform.openai.com/docs/guides/prompt-engineering)
- [DALL-E Best Practices](https://platform.openai.com/docs/guides/images)

### Community Resources
- [LangChain Discord](https://discord.gg/langchain)
- [OpenAI Developer Forum](https://community.openai.com/)

---

**Document Version**: 1.0.0
**Last Updated**: 2025-12-03
**Status**: ✅ Ready for Implementation
