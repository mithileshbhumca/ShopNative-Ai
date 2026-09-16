# ShopNative AI 🛒

AI-powered e-commerce app built with React Native that provides a modern shopping experience with AI-assisted search and in-app assistance, Firebase authentication, persisted cart & wishlist, checkout, payment integration, push notifications, and order history.

## Project info (short)
- Mobile app (React Native + TypeScript) for shopping with AI-assisted product search and in-app assistance.
- Key capabilities: intelligent search/suggestions, user auth (Firebase + Google), persisted cart & wishlist (redux-persist + AsyncStorage), checkout + Razorpay payments, push notifications, and order history.
- Intended users: mobile shoppers and demo/test app audiences; also a reference/sample implementation for integrating AI features into RN e-commerce flows.

## Stack
- **Language(s):** TypeScript (primary), Kotlin/Java (native Android bits), JavaScript
- **Framework / runtime:** React Native (0.85.x)
- **Notable libraries:**
  - @react-navigation (navigation)
  - @reduxjs/toolkit + react-redux + redux-persist (state management & persistence)
  - @react-native-firebase/app + @react-native-firebase/auth (Firebase)
  - react-native-webrtc (real-time/voice/video capabilities if used)
  - razorpay / react-native-razorpay (payment integration)
  - react-native-paper (UI components)

## High-level project flow / design

### Overview (user-facing flow)
1. App bootstrap (App.tsx)
   - Provider + PersistGate wrap the app to provide Redux store and persisted state.
   - SafeArea + NavigationContainer initialize screens and navigation.
   - Splash screen shown while auth & bootstrap tasks complete.
2. Authentication
   - Firebase email/password + Google Sign-In handled by auth services.
   - On successful auth, user state is stored in Redux and persisted.
3. Product discovery & AI features
   - Product listing screens present catalog from local/mock services or API.
   - AI-powered search/assistant augments product search (suggestions, semantic search, filtering).
   - Search results feed into product details.
4. Cart & wishlist
   - Add/remove items update Redux state.
   - Redux state persisted to AsyncStorage via redux-persist so cart/wishlist survive restarts.
5. Checkout & payment
   - Checkout screen collects address & order details.
   - Payment handled using Razorpay integration; backend or client-side flow validates and completes payment.
6. Post-purchase
   - Orders saved to history (Firebase/Backend).
   - Push notifications inform users of order updates.
7. Optional real-time features
   - WebRTC included for any live assistance (if implemented).

### Design (layered)
- UI layer: src/screens and src/components — screen-level composition and reusable UI.
- Navigation: src/navigation — stacks, tab navigators and route definitions.
- State: src/store and src/redux — RTK slices, actions, selectors, and persistence configuration.
- Services: src/services — auth, payments, push notifications, AI/search adapter, API clients.
- Data/persistence: AsyncStorage + redux-persist for local state; Firebase for user & order data.
- Theme & utils: src/theme + src/utils for consistent styling and helpers.
- Types: src/types for shared TypeScript interfaces.

### Simple ASCII architecture diagram
```
App (App.tsx)
  ├─ Provider (Redux + redux-persist)
  ├─ NavigationContainer (src/navigation)
  │    ├─ Auth stack (Login / Signup)
  │    └─ App stack (Home, Search, Product, Cart, Checkout, Orders)
  ├─ Screens -> Components (UI)
  └─ Services (src/services)
        ├─ AuthService (Firebase)
        ├─ Search/AI Adapter (semantic search / suggestions)
        ├─ PaymentService (Razorpay)
        └─ PushService (notifications)
```

## 🔮 Future Enhancements: AI-Powered Recommendations

We're planning major AI enhancements to optimize the recommendation engine using LLMs and emerging technologies.

### What's Planned?

| Feature | Impact | Timeline |
|---------|--------|----------|
| **Embedding-Based Semantic Search** | Find products by meaning, not just keywords | Week 1-2 |
| **LLM Contextual Recommendations** | AI understands intent ("trendy summer outfit") | Week 1-2 |
| **Hybrid Scoring System** | Combine LLM + embeddings + user ratings | Week 2 |
| **Vision-Based Matching** | Upload style photos → get similar products | Week 3-4 |
| **Trend Detection** | Automatic seasonal & trending recommendations | Week 3-4 |
| **RAG Integration** | Vector database for intelligent retrieval | Week 2-3 |

### Expected Impact 📈
- **15-25% uplift** in Average Order Value
- **50-85% improvement** in Recommendation Click-Through Rate
- **85%+ user satisfaction** with AI suggestions
- **35% reduction** in cart abandonment

### Cost Analysis 💰
- Claude API: ~$9/month
- Embeddings: ~$0.03/month
- Vector DB: Free tier (Pinecone)
- **Total: ~$13.50/month** for 10K active users

### Get Details 📖
👉 **[Read the Complete Roadmap →](./FUTURE_ENHANCEMENTS.md)**

For technical specs, implementation details, and code examples, see:
- **[FUTURE_ENHANCEMENTS.md](./FUTURE_ENHANCEMENTS.md)** - Quick navigation & overview
- **[futureEnhancements.md](./futureEnhancements.md)** - Complete technical guide with TypeScript implementations

---

## How it's organized (top-level)
- App.tsx — app bootstrap (Provider, PersistGate, Navigation)
- src/
  - components/   — reusable UI components
  - navigation/   — navigation stacks and navigators
  - redux/        — RTK slices and reducers
  - store/        — store creation and persistence setup
  - services/     — auth, payments, AI-search adapter, notifications
  - screens/      — screen components (Home, Search, Product, Cart, Checkout, Orders, Auth)
  - theme/        — colors, spacing, typography
  - types/        — shared TypeScript types/interfaces
  - utils/        — helpers and utilities (including recommendationEngine.ts)
- android/ ios/   — native platform projects
- scripts/        — helper scripts

## How to run (fast path)
1. Clone:
   ```bash
   git clone https://github.com/mithileshbhumca/ShopNative-Ai.git
   ```
2. Install:
   ```bash
   npm install
   ```
   or
   ```bash
   yarn
   ```
3. Configure environment:
   - Copy `.env.example` -> `.env` and set required keys (Firebase config, Razorpay key, Google sign-in config, etc.)
   - Set up Firebase project, enable Auth and Firestore/Realtime DB if used.
   - Configure Android/iOS Google sign-in and payment keys as per libraries' docs.
4. Start Metro:
   ```bash
   npm run start
   ```
5. Run on device/emulator:
   ```bash
   npm run android
   # or
   npm run ios
   ```

**Notes:**
- Node engine in package.json requires Node >= 22.11.0 (check compatibility with your environment).
- Native SDKs require platform-specific setup (Android SDK, Xcode for iOS).
- Check the `src/services` files for required backend endpoints and keys.

## Security & env vars
- Do not commit real API keys or Firebase service JSON files.
- Keep Razorpay keys, Google OAuth client IDs, and Firebase configs in `.env` or platform-specific secure storage.

## Where to look next (useful places in the repo)
- **App.tsx** — app bootstrap and top-level provider setup
- **src/navigation** — navigation graphs
- **src/store / src/redux** — store + slices and persistence
- **src/services/authService** — Firebase auth integration
- **src/services/pushNotifications** — push setup
- **src/utils/recommendationEngine.ts** — Current product recommendation logic
- **.env.example** — variables to configure
- **[FUTURE_ENHANCEMENTS.md](./FUTURE_ENHANCEMENTS.md)** — AI enhancement roadmap

## Suggested follow-ups
- Add an ARCHITECTURE.md explaining detailed data flows and where AI processing occurs (client vs backend).
- Document platform-specific setup steps for Google Sign-In and Razorpay for Android/iOS.
- Add a short diagram image (assets/docs/architecture.png) for visuals.
- Implement Phase 1 of AI enhancements (see [Future Enhancements](./FUTURE_ENHANCEMENTS.md))
