# 📘 Phase 3 — Lesson 1: React + TypeScript Foundation (Crystal Clear Deep Dive)

> **Goal:** React component-এ TypeScript apply করা — props typing, useState typing, event handler typing, children typing। Phase 2-এর সব knowledge-এ type safety যোগ করা।

---

# 🎯 PART 1: কেন React + TypeScript?

## 1.1 Pure JavaScript React-এর Problem

Phase 2-এ আমরা JavaScript-এ React শিখেছি। সেখানে অনেক bugs সম্ভব ছিল:

### Example 1.1.1: Props-এ Bug

```jsx
// SongCard.jsx (JavaScript)
function SongCard({ title, artist, duration }) {
    return (
        <div>
            <h3>{title}</h3>
            <p>{artist}</p>
            <p>{duration}</p>
        </div>
    );
}

// App.jsx
<SongCard tilte="Twinkle" artist="Traditional" duration="3:24" />
//        ↑ typo! "tilte" instead of "title"
```

**JavaScript কিছু বলে না!** Render হয়, কিন্তু title undefined হবে। User screen-এ blank দেখবে। Bug ধরা পড়বে production-এ।

### Example 1.1.2: Wrong Type Pass

```jsx
function SongCard({ duration }) {
    return <p>Duration: {duration.toFixed(2)}</p>;
}

// ভুলে string পাঠালাম
<SongCard duration="3:24" />
// 💥 Runtime error: "3:24".toFixed is not a function
```

JavaScript এই error compile-এ ধরবে না — runtime-এ crash।

### Example 1.1.3: Required Props ভুলে যাওয়া

```jsx
function Greeting({ name }) {
    return <h1>Hello, {name}!</h1>;
}

<Greeting />
// Output: "Hello, !" — silent bug
```

---

## 1.2 TypeScript Solution

```tsx
interface SongCardProps {
    title: string;
    artist: string;
    duration: string;
}

function SongCard({ title, artist, duration }: SongCardProps) {
    return (
        <div>
            <h3>{title}</h3>
            <p>{artist}</p>
            <p>{duration}</p>
        </div>
    );
}

// Use:
<SongCard tilte="Twinkle" artist="Traditional" duration="3:24" />
//        ↑ ❌ Error: 'tilte' does not exist. Did you mean 'title'?

<SongCard title="Twinkle" duration="3:24" />
//        ↑ ❌ Error: Property 'artist' is missing
```

**TypeScript সব ধরনের mistake আগেই ধরে!** Production-এ যাওয়ার আগে।

---

## 1.3 Setup — Vite + React + TypeScript

আগের Vite project-এ TypeScript add করতে চাইলে নতুন project বানান:

```bash
npm create vite@latest naptune-ts
```

```bash
Framework: › React
Variant:   › TypeScript     ← এবার TypeScript select করুন
```

```bash
cd naptune-ts
npm install
npm run dev
```

### File extension changes:

```
.jsx  →  .tsx     (React + TypeScript)
.js   →  .ts      (TypeScript only)
```

`.tsx` দেখলেই বুঝা যায় — TypeScript + JSX।

---

# 🎯 PART 2: Component Props Typing

## 2.1 Basic Props — Inline Type

সবচেয়ে simple way (ছোট components-এ ঠিক আছে):

```tsx
function Greeting({ name }: { name: string }) {
    return <h1>Hello, {name}!</h1>;
}

<Greeting name="Hasibuzzaman" />   // ✅
<Greeting name={123} />             // ❌ Error
<Greeting />                        // ❌ Error: missing name
```

**Inline type:** `{ name: string }` — destructuring-এর পরে type annotation।

---

## 2.2 Props with Interface (Recommended) ⭐

বড় components-এ interface use করুন — clean এবং reusable:

```tsx
interface GreetingProps {
    name: string;
}

function Greeting({ name }: GreetingProps) {
    return <h1>Hello, {name}!</h1>;
}
```

### Naming Convention:

```
ComponentName + Props
↓
GreetingProps, ButtonProps, SongCardProps, UserProfileProps
```

এটা React community-র standard। সবাই এটা use করে।

---

## 2.3 Multiple Props

```tsx
interface SongCardProps {
    title: string;
    artist: string;
    duration: number;          // seconds
    isPlaying: boolean;
}

function SongCard({ title, artist, duration, isPlaying }: SongCardProps) {
    return (
        <div>
            <h3>{isPlaying && '▶ '}{title}</h3>
            <p>{artist}</p>
            <p>{duration}s</p>
        </div>
    );
}

// Use:
<SongCard 
    title="Twinkle Twinkle"
    artist="Traditional"
    duration={185}
    isPlaying={false}
/>
```

---

## 2.4 Optional Props

`?` দিয়ে optional:

```tsx
interface SongCardProps {
    title: string;          // required
    artist: string;          // required
    duration?: number;       // optional
    isPlaying?: boolean;     // optional
    coverImage?: string;     // optional
}

function SongCard({ title, artist, duration, isPlaying = false, coverImage }: SongCardProps) {
    return (
        <div>
            {coverImage && <img src={coverImage} alt={title} />}
            <h3>{isPlaying && '▶ '}{title}</h3>
            <p>{artist}</p>
            {duration && <p>{duration}s</p>}
        </div>
    );
}

// Use — optional gula দিতে হবে না
<SongCard title="Twinkle" artist="Traditional" />              // ✅
<SongCard title="Twinkle" artist="Traditional" duration={185} /> // ✅
```

### Default Values

```tsx
function SongCard({ 
    title, 
    artist, 
    isPlaying = false,        // default value
    duration = 0               // default value
}: SongCardProps) { ... }
```

Lesson 5 (Phase 1) মনে আছে? Optional + default!

---

## 2.5 Complex Types

### Union Type Props

```tsx
interface ButtonProps {
    label: string;
    variant: "primary" | "secondary" | "danger";   // literal union
    size: "small" | "medium" | "large";
}

function Button({ label, variant, size }: ButtonProps) {
    return (
        <button className={`btn btn-${variant} btn-${size}`}>
            {label}
        </button>
    );
}

// Use:
<Button label="Save" variant="primary" size="medium" />     // ✅
<Button label="Save" variant="purple" size="medium" />      // ❌ Error!
<Button label="Save" variant="primary" size="huge" />        // ❌ Error!
```

**TypeScript autocomplete-এ valid options suggest করবে!**

### Object Props (Nested)

```tsx
interface Artist {
    name: string;
    country: string;
}

interface SongCardProps {
    title: string;
    artist: Artist;            // nested object
    tags: string[];            // array
}

function SongCard({ title, artist, tags }: SongCardProps) {
    return (
        <div>
            <h3>{title}</h3>
            <p>{artist.name} ({artist.country})</p>
            <div>
                {tags.map(tag => <span key={tag}>#{tag}</span>)}
            </div>
        </div>
    );
}

// Use:
<SongCard 
    title="Twinkle"
    artist={{ name: "Traditional", country: "Various" }}
    tags={["lullaby", "classic"]}
/>
```

---

# 🎯 PART 3: Function Props (Callbacks)

## 3.1 Simple Callback

```tsx
interface ButtonProps {
    label: string;
    onClick: () => void;       // no params, no return
}

function Button({ label, onClick }: ButtonProps) {
    return <button onClick={onClick}>{label}</button>;
}

// Use:
<Button label="Click me" onClick={() => console.log("clicked!")} />
```

### Callback Type Syntax:

```typescript
() => void                     // no params, no return
(value: string) => void        // one param
(a: number, b: number) => number  // multi params + return
```

---

## 3.2 Callback with Parameters

```tsx
interface SongCardProps {
    song: Song;
    onPlay: (song: Song) => void;          // receives song object
    onLike: (id: number) => void;          // receives id
    onDelete: (id: number) => void;
}

function SongCard({ song, onPlay, onLike, onDelete }: SongCardProps) {
    return (
        <div>
            <h3>{song.title}</h3>
            <button onClick={() => onPlay(song)}>▶</button>
            <button onClick={() => onLike(song.id)}>❤️</button>
            <button onClick={() => onDelete(song.id)}>🗑️</button>
        </div>
    );
}

// Parent component:
function App() {
    const handlePlay = (song: Song) => {
        console.log("Playing:", song.title);
    };
    
    const handleLike = (id: number) => {
        console.log("Liked song id:", id);
    };
    
    const handleDelete = (id: number) => {
        console.log("Deleted song id:", id);
    };
    
    return (
        <SongCard 
            song={mySong}
            onPlay={handlePlay}
            onLike={handleLike}
            onDelete={handleDelete}
        />
    );
}
```

**প্রতিটা callback-এ exact type annotation — কী data flow হচ্ছে clear।**

---

# 🎯 PART 4: useState Typing

## 4.1 Type Inference (যখন initial value দিচ্ছেন)

TypeScript নিজেই type infer করতে পারে:

```tsx
const [count, setCount] = useState(0);
//          ↑ inferred as number

const [name, setName] = useState("Hasib");
//          ↑ inferred as string

const [isActive, setIsActive] = useState(false);
//          ↑ inferred as boolean
```

### useState কীভাবে কাজ করে under the hood:

```typescript
// useState-এর signature (simplified)
function useState<T>(initial: T): [T, (newValue: T) => void];
```

দেখুন — generic `<T>`! আপনি initial value দিলে, TypeScript `T` infer করে নেয়।

---

## 4.2 Explicit Type — কখন দরকার?

### Case 1: Initial value `null` বা `undefined`

```tsx
// ❌ Type: null — পরে user set করলে error হবে
const [user, setUser] = useState(null);
setUser({ name: "Hasib" });   // ❌ Error: { name: string } not assignable to null

// ✅ Explicit type
const [user, setUser] = useState<User | null>(null);
setUser({ name: "Hasib", id: 1, email: "..." });  // ✅
```

### Case 2: Empty array

```tsx
// ❌ Type: never[]
const [songs, setSongs] = useState([]);
setSongs([{ id: 1 }]);   // ❌ Error

// ✅ Explicit
const [songs, setSongs] = useState<Song[]>([]);
setSongs([{ id: 1, title: "...", artist: "..." }]);  // ✅
```

### Case 3: Complex initial values

```tsx
// ✅ Explicit — clearer intent
const [filter, setFilter] = useState<"all" | "active" | "completed">("all");
```

---

## 4.3 Common Patterns

### Pattern 1: Form State

```tsx
interface FormData {
    name: string;
    email: string;
    age: number;
}

function MyForm() {
    const [formData, setFormData] = useState<FormData>({
        name: "",
        email: "",
        age: 0
    });
    
    const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
        const { name, value } = e.target;
        setFormData({ ...formData, [name]: value });
    };
    
    return ( /* JSX */ );
}
```

### Pattern 2: Loading States with Discriminated Union

Phase 1-এ শিখেছেন। React-এ এটা সবচেয়ে professional pattern:

```tsx
type FetchState<T> =
    | { status: "idle" }
    | { status: "loading" }
    | { status: "success"; data: T }
    | { status: "error"; error: string };

function UserProfile() {
    const [state, setState] = useState<FetchState<User>>({ status: "idle" });
    
    const loadUser = async () => {
        setState({ status: "loading" });
        try {
            const res = await fetch("/api/user");
            const user: User = await res.json();
            setState({ status: "success", data: user });
        } catch (e) {
            setState({ status: "error", error: (e as Error).message });
        }
    };
    
    // Type-safe rendering
    if (state.status === "idle") return <button onClick={loadUser}>Load</button>;
    if (state.status === "loading") return <p>Loading...</p>;
    if (state.status === "error") return <p>{state.error}</p>;
    
    // এখানে state.status === "success" — TypeScript জানে state.data exists
    return <p>Welcome, {state.data.name}!</p>;
}
```

**Phase 1-এর Discriminated Union এখানে real-world use!**

---

## 4.4 Functional Update Type

```tsx
const [count, setCount] = useState<number>(0);

// Direct value
setCount(5);

// Functional update
setCount(prev => prev + 1);
//        ↑ prev: number (TypeScript infer করে)
```

### useState Internal Type:

```typescript
function useState<T>(initial: T): [
    T,                              // current value
    (value: T | ((prev: T) => T)) => void   // setter — value or function
];
```

দুটোই allowed: direct value বা function।

---

# 🎯 PART 5: Event Handler Typing

React-এ events-এর জন্য specific TypeScript types আছে।

## 5.1 Click Events

```tsx
function MyButton() {
    const handleClick = (e: React.MouseEvent<HTMLButtonElement>) => {
        console.log("Clicked!", e);
        console.log("Button text:", e.currentTarget.innerText);
    };
    
    return <button onClick={handleClick}>Click</button>;
}
```

### Event Type Syntax:

```typescript
React.MouseEvent<HTMLButtonElement>
       ↑              ↑
    Event type     যেই element-এ event হচ্ছে
```

---

## 5.2 Common Event Types

### Input Change Event

```tsx
const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    console.log("New value:", e.target.value);
};

<input type="text" onChange={handleChange} />
```

### Textarea Change

```tsx
const handleChange = (e: React.ChangeEvent<HTMLTextAreaElement>) => {
    console.log(e.target.value);
};

<textarea onChange={handleChange} />
```

### Select Change

```tsx
const handleChange = (e: React.ChangeEvent<HTMLSelectElement>) => {
    console.log(e.target.value);
};

<select onChange={handleChange}>
    <option value="a">A</option>
</select>
```

### Form Submit

```tsx
const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    // ...
};

<form onSubmit={handleSubmit}>...</form>
```

### Keyboard Event

```tsx
const handleKeyDown = (e: React.KeyboardEvent<HTMLInputElement>) => {
    if (e.key === 'Enter') {
        console.log("Enter pressed!");
    }
};

<input onKeyDown={handleKeyDown} />
```

---

## 5.3 Event Type Reference Table

| Event | Type |
|-------|------|
| `onClick` | `React.MouseEvent<HTMLElement>` |
| `onChange` (input) | `React.ChangeEvent<HTMLInputElement>` |
| `onChange` (textarea) | `React.ChangeEvent<HTMLTextAreaElement>` |
| `onChange` (select) | `React.ChangeEvent<HTMLSelectElement>` |
| `onSubmit` | `React.FormEvent<HTMLFormElement>` |
| `onKeyDown` / `onKeyUp` | `React.KeyboardEvent<HTMLElement>` |
| `onFocus` / `onBlur` | `React.FocusEvent<HTMLElement>` |
| `onMouseEnter` / `onMouseLeave` | `React.MouseEvent<HTMLElement>` |

---

## 5.4 Inline Event Handlers — Type Inference

Inline handler-এ TypeScript automatic infer করে নেয় — type লিখতে হয় না:

```tsx
<button onClick={(e) => {
    // e automatically typed as React.MouseEvent<HTMLButtonElement>
    console.log(e.currentTarget.innerText);
}}>
    Click
</button>
```

কিন্তু **আলাদা function বানালে** type explicit লাগে:

```tsx
// ❌ Type implicit any
const handleClick = (e) => { ... };

// ✅ Type explicit
const handleClick = (e: React.MouseEvent<HTMLButtonElement>) => { ... };
```

---

# 🎯 PART 6: Children Prop Typing

## 6.1 `children: React.ReactNode`

`children` = component-এর opening/closing tag-এর মাঝের content।

```tsx
interface CardProps {
    title: string;
    children: React.ReactNode;     // ← এটাই standard
}

function Card({ title, children }: CardProps) {
    return (
        <div className="card">
            <h2>{title}</h2>
            <div className="content">{children}</div>
        </div>
    );
}

// Use:
<Card title="User Info">
    <p>Name: Hasibuzzaman</p>
    <p>Age: 30</p>
    <button>Edit</button>
</Card>
```

### `React.ReactNode` কী?

`React.ReactNode` — সবকিছু যা React render করতে পারে:
- JSX elements (`<div>`, `<MyComponent />`)
- Strings, numbers
- Arrays of nodes
- `null`, `undefined`, `false` (don't render)

**সবচেয়ে flexible children type।**

---

## 6.2 Optional Children

```tsx
interface CardProps {
    title: string;
    children?: React.ReactNode;    // optional!
}

function Card({ title, children }: CardProps) {
    return (
        <div>
            <h2>{title}</h2>
            {children && <div>{children}</div>}
        </div>
    );
}

// Use without children:
<Card title="Empty Card" />        // ✅

// Use with children:
<Card title="Filled">
    <p>Content here</p>
</Card>                            // ✅
```

---

## 6.3 Typed Children (Specific Type)

কখনো specific type-এর children চান। এটা advanced — সাধারণত `ReactNode` যথেষ্ট:

```tsx
interface ListProps {
    children: React.ReactElement[];   // শুধু React elements
}

interface SingleChildProps {
    children: React.ReactElement;     // exactly একটা element
}
```

কিন্তু practical-এ `React.ReactNode` use করুন।

---

# 🎯 PART 7: Real Naptune Components — TypeScript-এ

আগের Phase 2-এর components-কে TypeScript-এ convert করি।

## 7.1 Type Definitions File

প্রথমে একটা types file বানান — সব types এক জায়গায়:

### `src/types.ts`

```typescript
export type SongCategory = "Lullaby" | "Story" | "Kids" | "Educational";

export interface Song {
    readonly id: number;
    title: string;
    artist: string;
    category: SongCategory;
    duration: number;
    liked: boolean;
    isNew?: boolean;
}

export interface Artist {
    name: string;
    country: string;
}

export type Theme = "light" | "dark";
```

---

## 7.2 SongCard Component

### `src/components/SongCard.tsx`

```tsx
import type { Song } from '../types';

interface SongCardProps {
    song: Song;
    isPlaying: boolean;
    isDark: boolean;
    onPlay: (song: Song) => void;
    onLike: (id: number) => void;
    onDelete: (id: number) => void;
}

function SongCard({ song, isPlaying, isDark, onPlay, onLike, onDelete }: SongCardProps) {
    return (
        <div className={`song-card ${isPlaying ? 'playing' : (isDark ? 'dark-card' : 'light-card')}`}>
            <div className="info" onClick={() => onPlay(song)}>
                <h3>
                    {isPlaying && '🔊 '}{song.title}
                    {song.isNew && <span style={{ color: '#4CAF50' }}> 🆕</span>}
                </h3>
                <p>{song.artist} • {song.category}</p>
            </div>
            
            <div className="actions">
                <button onClick={() => onLike(song.id)}>
                    {song.liked ? '❤️' : '🤍'}
                </button>
                <button onClick={() => onDelete(song.id)}>
                    🗑️
                </button>
            </div>
        </div>
    );
}

export default SongCard;
```

### `import type { Song }` কী?

```typescript
import type { Song } from '../types';
```

`import type` — শুধু type import করছি, runtime-এ কিছু থাকবে না। এটা best practice for type imports।

---

## 7.3 SearchBar Component

### `src/components/SearchBar.tsx`

```tsx
interface SearchBarProps {
    query: string;
    onQueryChange: (newQuery: string) => void;
}

function SearchBar({ query, onQueryChange }: SearchBarProps) {
    const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
        onQueryChange(e.target.value);
    };
    
    return (
        <input 
            className="search-input"
            type="text"
            value={query}
            onChange={handleChange}
            placeholder="🔍 Search songs..."
        />
    );
}

export default SearchBar;
```

---

## 7.4 AddSongForm Component

### `src/components/AddSongForm.tsx`

```tsx
import { useState } from 'react';
import type { Song, SongCategory } from '../types';

interface AddSongFormProps {
    isDark: boolean;
    categories: SongCategory[];
    onAddSong: (song: Song) => void;
}

interface FormData {
    title: string;
    artist: string;
    category: SongCategory;
}

interface FormErrors {
    title?: string;
    artist?: string;
}

function AddSongForm({ isDark, categories, onAddSong }: AddSongFormProps) {
    const [formData, setFormData] = useState<FormData>({
        title: '',
        artist: '',
        category: 'Lullaby'
    });
    
    const [errors, setErrors] = useState<FormErrors>({});
    const [isOpen, setIsOpen] = useState<boolean>(false);
    
    const handleChange = (e: React.ChangeEvent<HTMLInputElement | HTMLSelectElement>) => {
        const { name, value } = e.target;
        setFormData({ ...formData, [name]: value });
    };
    
    const validate = (): boolean => {
        const errs: FormErrors = {};
        if (!formData.title.trim()) errs.title = 'গানের নাম লিখুন';
        if (!formData.artist.trim()) errs.artist = 'শিল্পীর নাম লিখুন';
        setErrors(errs);
        return Object.keys(errs).length === 0;
    };
    
    const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => {
        e.preventDefault();
        if (!validate()) return;
        
        const newSong: Song = {
            id: Date.now(),
            title: formData.title.trim(),
            artist: formData.artist.trim(),
            category: formData.category,
            duration: 180,
            liked: false,
            isNew: true
        };
        
        onAddSong(newSong);
        setFormData({ title: '', artist: '', category: 'Lullaby' });
        setIsOpen(false);
    };
    
    if (!isOpen) {
        return (
            <button onClick={() => setIsOpen(true)}>
                ➕ নতুন গান যোগ করুন
            </button>
        );
    }
    
    return (
        <form onSubmit={handleSubmit}>
            <input 
                name="title"
                value={formData.title}
                onChange={handleChange}
                placeholder="গানের নাম"
            />
            {errors.title && <p style={{ color: 'red' }}>{errors.title}</p>}
            
            <input 
                name="artist"
                value={formData.artist}
                onChange={handleChange}
                placeholder="শিল্পী"
            />
            {errors.artist && <p style={{ color: 'red' }}>{errors.artist}</p>}
            
            <select name="category" value={formData.category} onChange={handleChange}>
                {categories.map(cat => (
                    <option key={cat} value={cat}>{cat}</option>
                ))}
            </select>
            
            <button type="submit">Add Song</button>
            <button type="button" onClick={() => setIsOpen(false)}>Cancel</button>
        </form>
    );
}

export default AddSongForm;
```

দেখুন — সব types explicit, errors typed, events typed। **professional code!**

---

## 7.5 Card Component (with children)

### `src/components/Card.tsx`

```tsx
interface CardProps {
    title: string;
    children: React.ReactNode;
    isDark?: boolean;
}

function Card({ title, children, isDark = false }: CardProps) {
    return (
        <div className={`card ${isDark ? 'dark' : 'light'}`}>
            <h2 className="card-title">{title}</h2>
            <div className="card-content">
                {children}
            </div>
        </div>
    );
}

export default Card;
```

### Use:

```tsx
<Card title="User Info">
    <p>Name: Hasib</p>
    <p>Age: 30</p>
</Card>

<Card title="Settings" isDark={true}>
    <Toggle label="Dark mode" />
    <Toggle label="Notifications" />
</Card>
```

---

## 7.6 App Component — সব Connect

### `src/App.tsx`

```tsx
import { useState, useEffect } from 'react';
import type { Song, SongCategory } from './types';

import SongCard from './components/SongCard';
import SearchBar from './components/SearchBar';
import AddSongForm from './components/AddSongForm';

const INITIAL_SONGS: Song[] = [
    { id: 1, title: "Twinkle Twinkle", artist: "Traditional", category: "Lullaby", duration: 185, liked: false },
    { id: 2, title: "Baby Shark", artist: "Pinkfong", category: "Kids", duration: 168, liked: true },
];

const CATEGORIES: SongCategory[] = ['Lullaby', 'Story', 'Kids', 'Educational'];

function App() {
    const [songs, setSongs] = useState<Song[]>(INITIAL_SONGS);
    const [currentSong, setCurrentSong] = useState<Song | null>(null);
    const [query, setQuery] = useState<string>("");
    const [isDark, setIsDark] = useState<boolean>(false);
    
    // Handlers — fully typed
    const handlePlay = (song: Song): void => {
        setCurrentSong(currentSong?.id === song.id ? null : song);
    };
    
    const handleLike = (id: number): void => {
        setSongs(songs.map(s => 
            s.id === id ? { ...s, liked: !s.liked } : s
        ));
    };
    
    const handleDelete = (id: number): void => {
        setSongs(songs.filter(s => s.id !== id));
        if (currentSong?.id === id) setCurrentSong(null);
    };
    
    const handleAddSong = (newSong: Song): void => {
        setSongs([newSong, ...songs]);
    };
    
    // Filter
    const filteredSongs: Song[] = songs.filter(song =>
        song.title.toLowerCase().includes(query.toLowerCase())
    );
    
    return (
        <div className={`app ${isDark ? 'dark' : 'light'}`}>
            <h1>🎵 Naptune</h1>
            
            <SearchBar query={query} onQueryChange={setQuery} />
            
            <AddSongForm 
                isDark={isDark}
                categories={CATEGORIES}
                onAddSong={handleAddSong}
            />
            
            {filteredSongs.map(song => (
                <SongCard 
                    key={song.id}
                    song={song}
                    isPlaying={currentSong?.id === song.id}
                    isDark={isDark}
                    onPlay={handlePlay}
                    onLike={handleLike}
                    onDelete={handleDelete}
                />
            ))}
        </div>
    );
}

export default App;
```

প্রতিটা variable, function, state — সব **fully typed**!

---

# 🎯 PART 8: Common Mistakes

## ❌ Mistake 1: `React.FC` ব্যবহার (avoid!)

```tsx
// ❌ Old style — avoid এখন
const Button: React.FC<ButtonProps> = ({ label }) => {
    return <button>{label}</button>;
};

// ✅ Modern style
function Button({ label }: ButtonProps) {
    return <button>{label}</button>;
}
```

`React.FC` issues আছে — implicit children, generic constraints কঠিন। **Modern React community avoid করে।**

---

## ❌ Mistake 2: useState-এ wrong initial type

```tsx
// ❌ Type: never[]
const [items, setItems] = useState([]);
setItems([1, 2, 3]);  // Error!

// ✅ Explicit
const [items, setItems] = useState<number[]>([]);
setItems([1, 2, 3]);  // ✅
```

---

## ❌ Mistake 3: Event handler-এ missing type

```tsx
// ❌ implicit any
const handleClick = (e) => { ... };

// ✅ typed
const handleClick = (e: React.MouseEvent<HTMLButtonElement>) => { ... };
```

---

## ❌ Mistake 4: Children type wrong

```tsx
// ❌ Too restrictive
interface CardProps {
    children: string;
}

// ✅ Flexible
interface CardProps {
    children: React.ReactNode;
}
```

---

## ❌ Mistake 5: any everywhere

```tsx
// ❌ Defeats TypeScript purpose
function handle(data: any) { ... }

// ✅ Use proper types or unknown
function handle(data: User) { ... }
function handleUnknown(data: unknown) {
    if (isUser(data)) { ... }
}
```

---

# 🎯 PART 9: Cheat Sheet

```tsx
// ============== Component Props ==============
interface ButtonProps {
    label: string;
    variant?: "primary" | "secondary";
    disabled?: boolean;
    onClick: () => void;
    children?: React.ReactNode;
}

function Button({ label, variant = "primary", onClick }: ButtonProps) {
    return <button onClick={onClick}>{label}</button>;
}

// ============== useState ==============
const [count, setCount] = useState(0);                           // inferred
const [user, setUser] = useState<User | null>(null);             // explicit
const [items, setItems] = useState<string[]>([]);                // empty array
const [state, setState] = useState<FetchState>({ status: "idle" }); // discriminated

// ============== Event Handlers ==============
const handleClick = (e: React.MouseEvent<HTMLButtonElement>) => { };
const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => { };
const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => { };
const handleKey = (e: React.KeyboardEvent<HTMLInputElement>) => { };

// ============== Children ==============
interface Props {
    children: React.ReactNode;     // any renderable
    children?: React.ReactNode;    // optional
}

// ============== Function Props ==============
interface Props {
    onClick: () => void;
    onChange: (value: string) => void;
    onSubmit: (data: FormData) => Promise<void>;
}

// ============== Type Imports ==============
import type { Song, User } from './types';
```

---

# ✅ Lesson Summary

## Crystal Clear Concepts:

### 1. Component Props:

```tsx
interface Props { ... }
function Component({ ...props }: Props) { ... }
```

### 2. useState:

```tsx
useState(0)                    // inferred number
useState<User | null>(null)    // explicit
useState<Song[]>([])           // empty array
```

### 3. Event Handlers:

```tsx
React.MouseEvent<HTMLButtonElement>     // click
React.ChangeEvent<HTMLInputElement>     // input change
React.FormEvent<HTMLFormElement>         // form submit
React.KeyboardEvent<HTMLInputElement>   // keyboard
```

### 4. Children:

```tsx
children: React.ReactNode      // standard
```

### 5. File Extensions:

```
.tsx → React + TypeScript
.ts  → TypeScript only
```

### 6. Best Practices:

- ✅ `interface ComponentNameProps`
- ✅ `import type` for types
- ✅ explicit useState type when initial null/empty
- ❌ avoid `React.FC`
- ❌ avoid `any`

---

## 🔗 Final Comparison

| JavaScript React | TypeScript React |
|------------------|------------------|
| `function MyComp({ name })` | `function MyComp({ name }: Props)` |
| `useState(0)` | `useState<number>(0)` |
| `(e) => { }` | `(e: React.MouseEvent<...>) => { }` |
| Bug at runtime | Bug at compile time ✅ |

---

# 📚 Next Lesson Preview

**Phase 3 — Lesson 2: useEffect, Custom Hooks, Refs Typing**

`useEffect`-এ TypeScript apply, custom hooks বানানো with generics, `useRef` typing — DOM refs এবং mutable values। Phase 4 (Hooks Deep Dive) এর preparation।