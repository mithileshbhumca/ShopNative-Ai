# Future Enhancements: AI-Powered Recommendation Engine

## Overview

This document outlines the roadmap to optimize the current rule-based recommendation engine using LLMs and emerging AI technologies. The existing engine uses hardcoded scoring (category match = 2pts, price = 1pt, rating = 0–0.5pts), which provides good baseline performance but lacks adaptability, personalization, and semantic understanding.

---

## Current System Limitations

| Issue | Impact |
|-------|--------|
| **Static scoring weights** | Cannot adapt to user behavior or seasonal trends |
| **No cross-category insights** | Misses complementary products (shirt + jeans combo) |
| **No user intent understanding** | Treats "budget" and "style" queries identically |
| **No personalization** | Same recommendations for all users |
| **No temporal patterns** | Ignores time-based buying signals (season, events) |
| **Limited to local data** | Cannot leverage external trends or reviews |

---

## 🎯 Proposed AI-Powered Architecture

### 1. Embedding-Based Semantic Search (Priority: HIGH)

**Objective:** Understand semantic similarity between products beyond metadata matching.

**Technology Stack:**
- OpenAI `text-embedding-3-small` or Claude embeddings
- Vector caching with Redis/AsyncStorage
- Cosine similarity for ranking

**Implementation:**

```typescript
// src/services/embeddingService.ts
import OpenAI from "openai";

const openai = new OpenAI({
  apiKey: process.env.OPENAI_API_KEY,
});

// Cache to avoid repeated API calls
const embeddingCache = new Map<string, number[]>();

export const getProductEmbedding = async (product: Product): Promise<number[]> => {
  const cacheKey = product.id;
  
  if (embeddingCache.has(cacheKey)) {
    return embeddingCache.get(cacheKey)!;
  }

  const textRepresentation = `
    Product: ${product.name}
    Category: ${product.category}
    Description: ${product.description}
    Colors: ${product.colors.join(", ")}
    Price: Rs${product.price}
    Rating: ${product.rating}/5
    Discount: ${product.discount}%
  `.trim();

  const response = await openai.embeddings.create({
    model: "text-embedding-3-small",
    input: textRepresentation,
  });

  const embedding = response.data[0].embedding;
  embeddingCache.set(cacheKey, embedding);
  return embedding;
};

// Cosine similarity calculation
export const cosineSimilarity = (vec1: number[], vec2: number[]): number => {
  const dotProduct = vec1.reduce((sum, a, i) => sum + a * vec2[i], 0);
  const magnitude1 = Math.sqrt(vec1.reduce((sum, a) => sum + a * a, 0));
  const magnitude2 = Math.sqrt(vec2.reduce((sum, a) => sum + a * a, 0));
  
  if (magnitude1 === 0 || magnitude2 === 0) return 0;
  return dotProduct / (magnitude1 * magnitude2);
};
```

**Benefits:**
- Finds visually/stylistically similar products (not just category)
- Enables "customers who bought X also bought Y"
- Cost: $0.02 per 1M tokens (very cheap)
- Embedding cache reduces repeated API calls by 90%

**Expected Impact:** +10-15% in recommendation relevance

---

### 2. LLM-Powered Contextual Recommendations (Priority: CRITICAL)

**Objective:** Understand user intent and generate intelligent, context-aware recommendations.

**Technology Stack:**
- Claude 3.5 Sonnet or GPT-4
- Structured JSON output parsing
- Few-shot prompting for consistency

**Implementation:**

```typescript
// src/utils/llmRecommendationEngine.ts
import Anthropic from "@anthropic-ai/sdk";
import type { CartItem, Product } from "../types/productTypes";

const client = new Anthropic({
  apiKey: process.env.ANTHROPIC_API_KEY,
});

interface LLMRecommendationResult {
  reasoning: string;
  recommendedProducts: Product[];
  explanation: string;
}

export const getContextualRecommendations = async (
  userQuery: string,
  cartItems: CartItem[],
  wishlistItems: Product[],
  allProducts: Product[]
): Promise<LLMRecommendationResult> => {
  const cartSummary = cartItems.length > 0
    ? cartItems
        .map((item) => `${item.name} (${item.category}) - ${item.quantity}x Rs${item.price}`)
        .join(", ")
    : "empty";

  const wishlistSummary = wishlistItems.length > 0
    ? wishlistItems.map((item) => `${item.name} (${item.category})`).join(", ")
    : "none";

  const productCatalog = allProducts
    .map(
      (p) =>
        `${p.id}|${p.name}|${p.category}|Rs${p.price}|${p.discount}% off|${p.rating}⭐|${p.description}`
    )
    .join("\n");

  const systemPrompt = `You are an expert e-commerce shopping assistant for ShopNative - a fashion and lifestyle store.
Your role is to provide intelligent, personalized product recommendations based on user queries, cart contents, and wishlist.

IMPORTANT GUIDELINES:
1. Understand user intent: budget constraints, style preferences, occasions, complementary items
2. Never recommend products already in cart or wishlist
3. Prioritize value: high rating + good discount = best recommendation
4. For style queries, suggest coordinated combinations (e.g., shirt + jeans as a complete look)
5. Consider seasonal relevance and delivery time
6. Always explain why you're recommending these items
7. Provide exactly 3 product recommendations unless user asks otherwise`;

  const userPrompt = `USER QUERY: "${userQuery}"

CURRENT CART: ${cartSummary}
SAVED WISHLIST: ${wishlistSummary}

AVAILABLE PRODUCTS:
${productCatalog}

Please analyze this query and provide exactly 3 personalized product recommendations.
Format your response as valid JSON with this structure:
{
  "reasoning": "Brief explanation of your recommendation strategy",
  "productIds": ["id1", "id2", "id3"],
  "explanation": "Friendly 1-2 sentence explanation of the recommendations for the user"
}`;

  const message = await client.messages.create({
    model: "claude-3-5-sonnet-20241022",
    max_tokens: 500,
    system: systemPrompt,
    messages: [
      {
        role: "user",
        content: userPrompt,
      },
    ],
  });

  const responseText =
    message.content[0].type === "text" ? message.content[0].text : "{}";
  
  let parsed;
  try {
    // Extract JSON from response (in case of extra text)
    const jsonMatch = responseText.match(/\{[\s\S]*\}/);
    parsed = JSON.parse(jsonMatch ? jsonMatch[0] : responseText);
  } catch (error) {
    console.error("Failed to parse LLM response:", responseText);
    throw new Error("Invalid recommendation response from AI");
  }

  const recommendedProducts = parsed.productIds
    .map((id: string) => allProducts.find((p) => p.id === id))
    .filter((p: Product | undefined) => p !== undefined) as Product[];

  return {
    reasoning: parsed.reasoning || "AI-powered recommendation",
    recommendedProducts,
    explanation: parsed.explanation || "These products match your interests",
  };
};
```

**Integration with existing AiAssistant:**

```typescript
// Update src/components/AiAssistant.tsx
const sendMessage = async (text: string) => {
  const message = text.trim();

  if (!message || loading) {
    return;
  }

  const userMessage: Message = {
    id: `user-${Date.now()}`,
    role: "user",
    text: message,
  };

  setMessages((current) => [...current, userMessage]);
  setInput("");
  setLoading(true);

  try {
    // Try LLM-powered recommendations first
    const result = await getContextualRecommendations(
      message,
      cartItems,
      wishlistItems,
      products
    );
    
    setMessages((current) => [
      ...current,
      {
        id: `assistant-${Date.now()}`,
        role: "assistant",
        text: result.explanation,
      },
    ]);
  } catch (error) {
    // Fallback to rule-based engine on API failure
    console.warn("LLM recommendation failed, using fallback", error);
    setMessages((current) => [
      ...current,
      {
        id: `assistant-${Date.now()}`,
        role: "assistant",
        text: createLocalReply(message, cartItems, wishlistItems),
      },
    ]);
  } finally {
    setLoading(false);
  }
};
```

**Benefits:**
- Natural language understanding (not just keyword matching)
- Understands context: "under 2000" or "for gifting"
- Generates personalized explanations
- Adaptable to seasonal promotions and trends

**Expected Impact:** +20-30% improvement in user satisfaction and conversion

---

### 3. Hybrid Recommendation Engine (Priority: HIGH)

**Objective:** Combine LLM reasoning, semantic embeddings, and collaborative filtering for best-in-class recommendations.

**Architecture:**

```typescript
// src/utils/hybridRecommendationEngine.ts
import { products } from "../data/products";
import type { CartItem, Product } from "../types/productTypes";
import { getContextualRecommendations } from "./llmRecommendationEngine";
import {
  getProductEmbedding,
  cosineSimilarity,
} from "../services/embeddingService";

interface HybridScoringInput {
  llmRelevance: number; // 0-1 (does LLM think it's relevant?)
  semanticSimilarity: number; // 0-1 (how similar to cart items?)
  rating: number; // 0-1 (user ratings)
  discount: number; // 0-1 (normalized discount)
  priceAlignment: number; // 0-1 (within user's price range?)
}

const calculateHybridScore = (input: HybridScoringInput): number => {
  const weights = {
    llmRelevance: 0.5, // LLM intent understanding (most important)
    semanticSimilarity: 0.25, // Embedding-based similarity
    rating: 0.15, // Product quality
    discount: 0.05, // Value signaling
    priceAlignment: 0.05, // Budget matching
  };

  return (
    input.llmRelevance * weights.llmRelevance +
    input.semanticSimilarity * weights.semanticSimilarity +
    input.rating * weights.rating +
    input.discount * weights.discount +
    input.priceAlignment * weights.priceAlignment
  );
};

export const getHybridRecommendations = async (
  cartItems: CartItem[],
  wishlistItems: Product[],
  userQuery: string = "recommend complementary items",
  limit = 3
): Promise<Product[]> => {
  try {
    // Step 1: Get LLM-ranked candidates
    const llmResult = await getContextualRecommendations(
      userQuery,
      cartItems,
      wishlistItems,
      products
    );

    if (llmResult.recommendedProducts.length === 0) {
      return [];
    }

    // Step 2: Calculate average embedding of cart items
    const cartEmbeddings = await Promise.all(
      cartItems.map((item) => getProductEmbedding(item))
    );

    const avgCartEmbedding =
      cartEmbeddings.length > 0
        ? cartEmbeddings[0].map((_, i) =>
            cartEmbeddings.reduce((sum, emb) => sum + emb[i], 0) /
            cartEmbeddings.length
          )
        : new Array(1536).fill(0); // Fallback for empty cart

    // Step 3: Calculate average price in cart
    const avgPrice =
      cartItems.length > 0
        ? cartItems.reduce((sum, item) => sum + item.price, 0) /
          cartItems.length
        : 1800;

    // Step 4: Hybrid scoring
    const hybridScores = await Promise.all(
      llmResult.recommendedProducts.map(async (product) => {
        const embedding = await getProductEmbedding(product);
        const semanticScore = cosineSimilarity(avgCartEmbedding, embedding);
        const ratingScore = product.rating / 5;
        const normalizedDiscount = Math.min(product.discount / 50, 1);
        const priceAlignment =
          Math.abs(product.price - avgPrice) < 1000
            ? 1
            : Math.max(0, 1 - Math.abs(product.price - avgPrice) / 5000);

        return {
          product,
          score: calculateHybridScore({
            llmRelevance: 1.0, // LLM already ranked these
            semanticSimilarity: semanticScore,
            rating: ratingScore,
            discount: normalizedDiscount,
            priceAlignment,
          }),
        };
      })
    );

    return hybridScores
      .sort((a, b) => b.score - a.score)
      .slice(0, limit)
      .map((item) => item.product);
  } catch (error) {
    console.error("Hybrid recommendation failed:", error);
    return [];
  }
};
```

**Benefits:**
- Combines strengths of three approaches
- Resilient: falls back gracefully if LLM fails
- Continuously improves as embeddings learn
- Balances multiple ranking signals

**Expected Impact:** +15-20% uplift when combined with LLM

---

### 4. Retrieval-Augmented Generation (RAG) - Advanced (Priority: MEDIUM)

**Objective:** Enable Claude to retrieve relevant product context from a vector database, allowing it to make more informed recommendations.

**Technology Stack:**
- Pinecone, Weaviate, or Supabase (pgvector)
- Langchain for RAG orchestration
- Product embeddings + metadata indexing

**Implementation Sketch:**

```typescript
// src/services/ragService.ts
import { Pinecone } from "@pinecone-database/pinecone";
import { getProductEmbedding } from "./embeddingService";

const pinecone = new Pinecone({
  apiKey: process.env.PINECONE_API_KEY,
});

const indexName = "shopnative-products";

// One-time: Index all products (run on backend)
export const indexProductsForRAG = async (allProducts: Product[]) => {
  const index = pinecone.Index(indexName);

  for (const product of allProducts) {
    const embedding = await getProductEmbedding(product);

    await index.upsert([
      {
        id: product.id,
        values: embedding,
        metadata: {
          name: product.name,
          category: product.category,
          price: product.price,
          rating: product.rating,
          discount: product.discount,
          description: product.description,
        },
      },
    ]);
  }

  console.log(`Indexed ${allProducts.length} products in Pinecone`);
};

// Query-time: Retrieve relevant products
export const queryRelevantProducts = async (
  userQuery: string,
  topK = 5
): Promise<Product[]> => {
  const index = pinecone.Index(indexName);
  const queryEmbedding = await getProductEmbedding({
    name: userQuery,
    description: userQuery,
  } as any);

  const results = await index.query({
    vector: queryEmbedding,
    topK,
    includeMetadata: true,
  });

  return results.matches.map((match) => ({
    id: match.id,
    name: match.metadata?.name || "",
    price: match.metadata?.price || 0,
    rating: match.metadata?.rating || 0,
    // ... other fields
  })) as Product[];
};
```

**Benefits:**
- Scales to 100K+ products
- Claude has context about entire catalog
- Faster than client-side similarity search
- Enables cross-product insights

**Cost:** Pinecone free tier supports up to 1M vectors (~12K products)

---

### 5. Multimodal Vision-Based Recommendations (Priority: MEDIUM)

**Objective:** Allow users to upload images and get recommendations for similar styles.

**Technology Stack:**
- Claude 3.5 Sonnet with vision (supports base64 images)
- Image processing libraries

**Implementation:**

```typescript
// src/services/visionRecommendationService.ts
import Anthropic from "@anthropic-ai/sdk";
import { readFileAsBase64 } from "../utils/imageUtils";

const client = new Anthropic({
  apiKey: process.env.ANTHROPIC_API_KEY,
});

export const recommendByStyle = async (
  imageUri: string,
  allProducts: Product[]
): Promise<{
  styleDescription: string;
  recommendations: Product[];
}> => {
  // Convert image to base64
  const base64Image = await readFileAsBase64(imageUri);

  const productList = allProducts
    .map((p) => `${p.id}: ${p.name} (${p.category})`)
    .join("\n");

  const response = await client.messages.create({
    model: "claude-3-5-sonnet-20241022",
    max_tokens: 1024,
    messages: [
      {
        role: "user",
        content: [
          {
            type: "image",
            source: {
              type: "base64",
              media_type: "image/jpeg",
              data: base64Image,
            },
          },
          {
            type: "text",
            text: `Analyze the style in this image and recommend 3 similar products from our catalog.

AVAILABLE PRODUCTS:
${productList}

Respond as JSON:
{
  "styleDescription": "description of the style",
  "productIds": ["id1", "id2", "id3"],
  "reason": "why these match the style"
}`,
          },
        ],
      },
    ],
  });

  const responseText =
    response.content[0].type === "text" ? response.content[0].text : "{}";
  const parsed = JSON.parse(responseText);

  return {
    styleDescription: parsed.styleDescription,
    recommendations: parsed.productIds
      .map((id: string) => allProducts.find((p) => p.id === id))
      .filter((p: Product | undefined) => p !== undefined) as Product[],
  };
};
```

**UI Integration:**

```typescript
// New screen: StyleMatchScreen.tsx
import { launchImageLibrary } from "react-native-image-picker";
import { recommendByStyle } from "../services/visionRecommendationService";

const handleUploadStyle = async () => {
  const result = await launchImageLibrary({ mediaType: "photo" });
  
  if (result.assets?.[0]?.uri) {
    setLoading(true);
    try {
      const { styleDescription, recommendations } = await recommendByStyle(
        result.assets[0].uri,
        products
      );
      navigation.navigate("StyleRecommendations", {
        style: styleDescription,
        products: recommendations,
      });
    } catch (error) {
      Alert.alert("Error", "Could not analyze style. Please try another image.");
    } finally {
      setLoading(false);
    }
  }
};
```

**Benefits:**
- Unique differentiator in fashion e-commerce
- Bridges online-to-offline shopping
- Enables "wear-this-look" type searches

**Expected Impact:** High engagement on social sharing

---

### 6. Trend Detection & Seasonal Analytics (Priority: LOW)

**Objective:** Detect trending products and seasonal recommendations automatically.

**Implementation:**

```typescript
// src/services/trendAnalyticsService.ts
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic({
  apiKey: process.env.ANTHROPIC_API_KEY,
});

export const detectTrendingPatterns = async (
  searchHistory: string[],
  cartHistory: CartItem[][]
): Promise<{
  trending: string[];
  seasonal: string[];
  insights: string;
}> => {
  const currentDate = new Date();
  const currentMonth = currentDate.toLocaleString("default", { month: "long" });
  const currentSeason =
    [11, 0, 1].includes(currentDate.getMonth())
      ? "winter"
      : [2, 3, 4].includes(currentDate.getMonth())
        ? "spring"
        : [5, 6, 7].includes(currentDate.getMonth())
          ? "summer"
          : "autumn";

  const response = await client.messages.create({
    model: "claude-3-5-sonnet-20241022",
    max_tokens: 500,
    messages: [
      {
        role: "user",
        content: `Analyze shopping patterns and detect trends.

Recent searches: ${searchHistory.slice(-50).join(", ")}
Current season: ${currentSeason} (${currentMonth})

Identify:
1. Currently trending product categories
2. Seasonal recommendations for ${currentSeason}
3. Brief market insight

Respond as JSON:
{
  "trending": ["category1", "category2"],
  "seasonal": ["category1", "category2"],
  "insights": "brief analysis"
}`,
      },
    ],
  });

  const responseText =
    response.content[0].type === "text" ? response.content[0].text : "{}";
  return JSON.parse(responseText);
};
```

---

## 📊 Implementation Roadmap

| Phase | Feature | Effort | Timeline | ROI | Dependencies |
|-------|---------|--------|----------|-----|--------------|
| **1** | Embedding-based similarity | 2-3 days | Week 1 | High | OpenAI API key |
| **2** | LLM contextual recommendations | 3-4 days | Week 1-2 | Very High | Claude/GPT API key |
| **3** | Hybrid scoring system | 2 days | Week 2 | Very High | Phase 1 + 2 |
| **4** | RAG with vector DB | 3-5 days | Week 2-3 | Medium | Pinecone/Weaviate |
| **5** | Vision-based recommendations | 4-5 days | Week 3-4 | High | Claude Vision API |
| **6** | Trend detection | 2-3 days | Week 4 | Medium | Phase 2 |

---

## 💰 Cost Breakdown

| Service | Cost | Usage | Monthly Cost |
|---------|------|-------|--------------|
| **Claude API (input)** | $3/1M tokens | ~100K tokens/day | ~$9 |
| **Claude API (cache)** | $0.30/1M tokens | ~500K tokens cached/day | ~$4.50 |
| **Text Embeddings** | $0.02/1M tokens | ~50K tokens/day (cached) | ~$0.03 |
| **Pinecone** | Free | Up to 1M vectors | $0 (free tier) |
| **TOTAL** | - | - | **~$13.50/month** |

*For 10K daily active users with 50K API calls/day*

---

## 🚀 Quick Start Guide

### Setup (Backend/Environment)

```bash
# Install dependencies
npm install @anthropic-ai/sdk openai pinecone-database

# Create .env file
ANTHROPIC_API_KEY=sk-ant-xxxxx
OPENAI_API_KEY=sk-xxxxx
PINECONE_API_KEY=xxxxx
NODE_ENV=production
```

### Phase 1: Add Embeddings (First 2 Days)

1. Create `src/services/embeddingService.ts`
2. Add embedding cache to Redux store
3. Update `getRecommendedProducts()` to use semantic similarity
4. Test with existing product data

### Phase 2: Add LLM (Days 3-4)

1. Create `src/utils/llmRecommendationEngine.ts`
2. Update `AiAssistant.tsx` to use Claude for responses
3. Add fallback to rule-based engine
4. Test with various user queries

### Phase 3: Hybrid System (Day 5)

1. Create `src/utils/hybridRecommendationEngine.ts`
2. Integrate with both previous approaches
3. Test end-to-end flow
4. Monitor API costs

---

## 📈 Expected Impact Metrics

| Metric | Before | After | Uplift |
|--------|--------|-------|--------|
| **Recommendation CTR** | 12% | 18-22% | +50-85% |
| **Average Order Value** | Rs 2,000 | Rs 2,500-2,800 | +15-25% |
| **Cart Abandonment** | 45% | 35-38% | -7-10 pts |
| **Session Duration** | 5 mins | 6-7 mins | +20-40% |
| **Repeat Purchase Rate** | 20% | 28-35% | +40-75% |
| **User Satisfaction (NPS)** | 35 | 50-55 | +15-20 pts |

---

## 🔒 Privacy & Security Considerations

- **User Data:** Never send PII (email, phone, address) to LLMs; only send product preferences
- **Cart Privacy:** Don't store user cart history externally; process locally or use encrypted queries
- **API Keys:** Use environment variables, rotate regularly, implement rate limiting
- **Compliance:** Ensure GDPR/local data protection compliance when storing search history

---

## ⚠️ Potential Challenges & Mitigations

| Challenge | Mitigation |
|-----------|-----------|
| **API latency** (LLM responses slow down browsing) | Cache responses, use background workers, show skeleton loaders |
| **Cost explosion** (high API usage) | Implement aggressive caching, batch requests, use local models for fallback |
| **LLM hallucinations** (recommending non-existent products) | Strict JSON validation, product ID verification |
| **Cold start** (new users, no data) | Use trending products + category defaults |
| **Mobile constraints** (memory, bandwidth) | Offload embeddings to backend, lightweight local fallback |

---

## 🎯 Success Criteria

- ✅ Recommendations reflect user intent (not just metadata)
- ✅ 85%+ user approval rating on suggested products
- ✅ 15-25% uplift in AOV from personalized recommendations
- ✅ API costs under Rs 500/month ($6 USD)
- ✅ <2 second recommendation latency (p95)
- ✅ Zero PII leakage in API calls

---

## 📚 References & Resources

- [Claude API Documentation](https://docs.anthropic.com)
- [OpenAI Embeddings Guide](https://platform.openai.com/docs/guides/embeddings)
- [Pinecone Vector Database](https://docs.pinecone.io)
- [Recommendation Systems Best Practices](https://arxiv.org/pdf/1803.02465.pdf)
- [RAG Patterns in Production](https://www.anthropic.com/research)

---

## 📝 Next Steps

1. **Week 1:** Implement Phase 1 (Embeddings) and Phase 2 (LLM) in parallel
2. **Week 2:** Deploy Phase 3 (Hybrid) to production with A/B testing
3. **Week 3:** Monitor metrics and optimize weights
4. **Week 4:** Plan Phase 4-6 based on initial results

---

*Last Updated: September 16, 2026*
*Document Owner: ShopNative AI Team*
