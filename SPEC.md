# QuizGenius — System Specification & Architecture Document (SPEC.md)

**Role**: Senior Systems Architect  
**Project**: QuizGenius (AI-Powered Flashcard & Study SaaS)  
**Tech Stack**: React (Vite), Tailwind CSS, Supabase (Auth, Postgres, RLS), Google Gemini API  

---

## 1. Database Schema (Supabase / PostgreSQL)

### 1.1 Overview & Entity-Relationship Model
The QuizGenius data model consists of user profiles (managed by Supabase Auth), study **Decks**, and **Flashcards**. Row Level Security (RLS) is strictly enforced to ensure multi-tenant data isolation. Each flashcard explicitly tracks its individual difficulty rating (`easy`, `medium`, `hard`).

```
       +--------------------+
       |   auth.users       | (Supabase Managed)
       +--------------------+
                 | 1
                 |
                 | N
       +--------------------+
       |       decks        |
       +--------------------+
                 | 1
                 |
                 | N
       +--------------------+
       |     flashcards     |
       +--------------------+
```

---

### 1.2 SQL Schema & DDL Scripts

```sql
-- Enable UUID extension
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- Create difficulty ENUM type
CREATE TYPE public.card_difficulty AS ENUM ('easy', 'medium', 'hard');

-- ==========================================
-- TABLE: decks
-- ==========================================
CREATE TABLE public.decks (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
    title VARCHAR(255) NOT NULL,
    description TEXT,
    source_text TEXT,
    card_count INT NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Index for user lookups and sorting by date
CREATE INDEX idx_decks_user_id ON public.decks(user_id);
CREATE INDEX idx_decks_created_at ON public.decks(created_at DESC);

-- ==========================================
-- TABLE: flashcards
-- ==========================================
CREATE TABLE public.flashcards (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    deck_id UUID NOT NULL REFERENCES public.decks(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
    front TEXT NOT NULL,
    back TEXT NOT NULL,
    hint TEXT,
    difficulty public.card_difficulty NOT NULL DEFAULT 'medium',
    position INT NOT NULL DEFAULT 0,
    mastered BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Indexes for performance
CREATE INDEX idx_flashcards_deck_id ON public.flashcards(deck_id);
CREATE INDEX idx_flashcards_user_id ON public.flashcards(user_id);
CREATE INDEX idx_flashcards_difficulty ON public.flashcards(difficulty);

-- ==========================================
-- TRIGGER: Auto-update updated_at timestamp
-- ==========================================
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER update_decks_updated_at
    BEFORE UPDATE ON public.decks
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_flashcards_updated_at
    BEFORE UPDATE ON public.flashcards
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

-- ==========================================
-- TRIGGER: Maintain deck card_count automatically
-- ==========================================
CREATE OR REPLACE FUNCTION update_deck_card_count()
RETURNS TRIGGER AS $$
BEGIN
    IF (TG_OP = 'INSERT') THEN
        UPDATE public.decks SET card_count = card_count + 1 WHERE id = NEW.deck_id;
    ELSIF (TG_OP = 'DELETE') THEN
        UPDATE public.decks SET card_count = GREATEST(card_count - 1, 0) WHERE id = OLD.deck_id;
    END IF;
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_update_deck_card_count
    AFTER INSERT OR DELETE ON public.flashcards
    FOR EACH ROW EXECUTE FUNCTION update_deck_card_count();
```

---

### 1.3 Row Level Security (RLS) Policies

```sql
-- Enable RLS on all tables
ALTER TABLE public.decks ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.flashcards ENABLE ROW LEVEL SECURITY;

-- ------------------------------------------
-- Decks Policies
-- ------------------------------------------
CREATE POLICY "Users can view their own decks"
    ON public.decks FOR SELECT
    USING (auth.uid() = user_id);

CREATE POLICY "Users can create their own decks"
    ON public.decks FOR INSERT
    WITH CHECK (auth.uid() = user_id);

CREATE POLICY "Users can update their own decks"
    ON public.decks FOR UPDATE
    USING (auth.uid() = user_id)
    WITH CHECK (auth.uid() = user_id);

CREATE POLICY "Users can delete their own decks"
    ON public.decks FOR DELETE
    USING (auth.uid() = user_id);

-- ------------------------------------------
-- Flashcards Policies
-- ------------------------------------------
CREATE POLICY "Users can view flashcards in their decks"
    ON public.flashcards FOR SELECT
    USING (auth.uid() = user_id);

CREATE POLICY "Users can create flashcards in their decks"
    ON public.flashcards FOR INSERT
    WITH CHECK (auth.uid() = user_id);

CREATE POLICY "Users can update flashcards in their decks"
    ON public.flashcards FOR UPDATE
    USING (auth.uid() = user_id)
    WITH CHECK (auth.uid() = user_id);

CREATE POLICY "Users can delete flashcards in their decks"
    ON public.flashcards FOR DELETE
    USING (auth.uid() = user_id);
```

---

## 2. JSON Interface for AI Generation (Google Gemini API)

To guarantee consistent output parsing, QuizGenius enforces **Structured Outputs** via Gemini's `responseSchema` configuration (`responseMimeType: "application/json"`). Every generated flashcard MUST include a required `difficulty` field (`easy`, `medium`, or `hard`).

### 2.1 Gemini API Response JSON Schema Specification

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "QuizGeniusFlashcardGenerationResponse",
  "description": "Structured JSON output generated by Google Gemini for flashcard generation.",
  "type": "object",
  "required": ["deckTitle", "deckDescription", "cards"],
  "properties": {
    "deckTitle": {
      "type": "string",
      "description": "A concise, descriptive title for the flashcard deck derived from the source text (max 60 chars)."
    },
    "deckDescription": {
      "type": "string",
      "description": "A 1-2 sentence summary of the concepts covered in this deck."
    },
    "cards": {
      "type": "array",
      "description": "Array of generated flashcards.",
      "minItems": 1,
      "items": {
        "type": "object",
        "required": ["front", "back", "difficulty"],
        "properties": {
          "front": {
            "type": "string",
            "description": "The question, concept term, or prompt on the front of the card."
          },
          "back": {
            "type": "string",
            "description": "The clear, accurate explanation, definition, or answer on the back."
          },
          "hint": {
            "type": "string",
            "description": "Optional mnemonic, clue, or context tip to aid recall."
          },
          "difficulty": {
            "type": "string",
            "enum": ["easy", "medium", "hard"],
            "description": "The cognitive complexity level required to answer this card ('easy' = basic recall/definition, 'medium' = conceptual understanding/application, 'hard' = complex synthesis/deep analysis)."
          }
        },
        "additionalProperties": false
      }
    }
  },
  "additionalProperties": false
}
```

---

### 2.2 System Prompt & Prompt Engineering Specification

```typescript
export const GEMINI_SYSTEM_INSTRUCTION = `
You are QuizGenius AI, an expert educational content designer and memory science specialist.
Your task is to analyze user-provided study materials or notes and transform them into high-quality, atomized flashcards for effective spaced repetition learning.

Rules for Card Generation:
1. Atomicity: Each card must test ONE single concept or fact. Break down complex ideas into multiple focused cards.
2. Clarity: Questions on the front must be unambiguous. Answers on the back must be concise yet thorough.
3. Hints: Include a subtle hint (clue or mnemonic) whenever helpful, but keep it brief.
4. Difficulty Assessment: You MUST explicitly categorize EVERY card's difficulty as one of:
   - "easy": Direct definitions, simple terminology, or direct factual recall.
   - "medium": Multi-step concepts, applications of rules, or relationships between terms.
   - "hard": Complex problem solving, synthesis of multiple ideas, or edge cases.
5. Language & Tone: Match the language of the source text. Keep tone encouraging and strictly factual.
6. Formatting: Do not use Markdown inside card fields unless rendering mathematical/scientific expressions.

Return your response strictly according to the provided JSON schema.
`;
```

---

### 2.3 TypeScript Interfaces for Frontend & AI Engine

```typescript
// Shared Types across React frontend and AI integration layer

export type CardDifficulty = 'easy' | 'medium' | 'hard';

export interface FlashcardAIItem {
  front: string;
  back: string;
  hint?: string;
  difficulty: CardDifficulty;
}

export interface AIGeneratedDeckResponse {
  deckTitle: string;
  deckDescription: string;
  cards: FlashcardAIItem[];
}

export interface Deck {
  id: string;
  user_id: string;
  title: string;
  description?: string;
  source_text?: string;
  card_count: number;
  created_at: string;
  updated_at: string;
}

export interface Flashcard {
  id: string;
  deck_id: string;
  user_id: string;
  front: string;
  back: string;
  hint?: string;
  difficulty: CardDifficulty;
  position: number;
  mastered: boolean;
  created_at: string;
  updated_at: string;
}

export interface GenerateCardsOptions {
  cardCountTarget?: number;
  difficultyFilter?: CardDifficulty | 'all';
}
```

---

## 3. Component Hierarchy & Architecture

QuizGenius adopts a clean, modular component design built on top of React, Vite, and Tailwind CSS. State management flows top-down with focused views for generation, grid editing, and focused single-card study.

### 3.1 Visual Tree Diagram

```
App (Root Provider / Auth Guard / Router)
 ├── Navbar (Logo, Navigation Links, User Avatar & Menu, Theme Toggle)
 ├── Sidebar / Drawer (Deck History, Quick Filters, Stats Summary)
 │
 ├── Main Content Area
 │    │
 │    ├── View: Dashboard / Deck List
 │    │    ├── DeckHeader (Search, Filter by difficulty, Create Deck Trigger)
 │    │    └── DeckGrid
 │    │         └── DeckCard (Title, Card Count, Progress Bar, Actions Menu)
 │    │
 │    ├── View: Flashcard Generator (Input Section & Grid Preview)
 │    │    ├── GeneratorHeader (Title & Instructions)
 │    │    ├── InputSection
 │    │    │    ├── SourceTextInput (Textarea with character count, file dropzone)
 │    │    │    ├── GenerationControls (Target Card Count Slider, Difficulty Distribution)
 │    │    │    └── ActionToolbar (Generate Button with loading state, Clear Button)
 │    │    │
 │    │    └── GenerationPreview (Active after AI completion)
 │    │         ├── PreviewHeader (Deck Title Edit, Save Button, Switch to Study View)
 │    │         └── FlashcardGrid
 │    │              ├── GridControls (Filter by Difficulty badge, Select All)
 │    │              ├── FlashcardTile (Flippable Grid Card Item)
 │    │              │    ├── CardFront (Prompt Text, Difficulty Badge)
 │    │              │    ├── CardBack (Answer Text, Hint Badge)
 │    │              │    └── CardActions (Edit Button, Delete Button)
 │    │              └── AddCustomCardButton (Inline card creator)
 │    │
 │    └── View: Single Card Study View (Focus Study Mode)
 │         ├── StudyHeader (Deck Title, Exit Study View Button)
 │         ├── StudyProgress (Progress Bar, Current Index / Total, Mastery Tracker)
 │         ├── SingleCardStudyView (Focused Single Card Viewer)
 │         │    ├── DifficultyBadgeIndicator (Displays card's easy/medium/hard rating)
 │         │    ├── SingleFlashcardCard (Large 3D Flip Card: Front vs Back view)
 │         │    │    ├── FrontView (Question, Hint Trigger)
 │         │    │    └── BackView (Answer, Explanation, Key Concepts)
 │         │    └── NavigationControls (Previous Card, Next Card, Shuffle, Flip Button)
 │         └── StudyRatingActions (Mark Mastered / Retry, Keyboard Shortcuts Guide)
 │
 ├── Modals & Toasts
 │    ├── EditCardModal
 │    ├── DeleteConfirmationModal
 │    └── ToastContainer (Notifications for Save, AI errors, Copy success)
 └── Footer (Copyright, API Status, Help Links)
```

---

### 3.2 Key Component Specifications & Data Flow

| Component Name | Type | Props / State | Responsibilities |
|---|---|---|---|
| `App` | Controller | `user`, `activeDeck`, `currentView` | Handles authentication status via Supabase `onAuthStateChange`, routes between Dashboard, Generator, and Single Card Study View. |
| `InputSection` | Feature Container | `onGenerate(text, options)`, `isGenerating` | Captures source text, configures target card counts, and triggers Gemini API generation requests. |
| `FlashcardGrid` | Layout / Presentational | `cards`, `onCardUpdate`, `onCardDelete` | Renders cards in a multi-column responsive grid layout with difficulty badges (`easy`, `medium`, `hard`). |
| `SingleCardStudyView` | Focus Study Component | `cards`, `currentIndex`, `onNext`, `onPrev`, `onToggleMastered` | **Displays exactly one flashcard at a time**. Manages single-card 3D flip animation, keyboard shortcuts (`Space` to flip, `ArrowLeft`/`ArrowRight` to navigate), and difficulty color coding. |
| `SingleFlashcardCard` | Presentational Widget | `card`, `isFlipped`, `onFlip` | High-impact 3D card flipping container holding `FrontView` and `BackView` of the active card. |
| `StudyProgress` | Presentational Widget | `currentIndex`, `totalCards`, `masteredCount` | Displays visual progress bar and mastery ratio during single-card study sessions. |

---

### 3.3 State Management & Data Flow Architecture

1. **Generation Flow**:
   - User inputs source text into `SourceTextInput` inside `InputSection`.
   - `InputSection` calls Gemini API with structured schema mandating a `difficulty` (`easy` | `medium` | `hard`) on every item in `cards[]`.
   - AI response is parsed and rendered in `FlashcardGrid`.
2. **Single Card Study View Flow**:
   - User initiates study session for a deck.
   - `SingleCardStudyView` maintains `currentIndex` state (0 to N-1).
   - Only the active flashcard `cards[currentIndex]` is rendered to DOM inside `SingleFlashcardCard`.
   - Pressing Next (`->`) or Previous (`<-`) updates `currentIndex` and resets card flip state to `isFlipped = false`.
3. **Persistence Flow**:
   - User saves deck to Supabase. Flashcards are inserted into `public.flashcards` along with their corresponding `difficulty` column value.
