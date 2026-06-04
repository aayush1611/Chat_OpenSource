# @fynix/chat-assistant — Integration Guide

This guide covers everything you need to integrate the chat-assistant widget into any standalone repository. For an overview of the package, architecture, and project structure, see [README.md](./README.md).

---

## Table of Contents

- [Integration Mode 1: Script Tag (IIFE)](#integration-mode-1-script-tag-iife)
  - [How render() works](#how-render-works)
  - [Shared state across components](#shared-state-across-components)
  - [Unmounting](#unmounting)
- [Integration Mode 2: React Import](#integration-mode-2-react-import)
  - [ChatProvider](#chatprovider)
  - [Using individual components](#using-individual-components)
- [Configuration Reference](#configuration-reference)
  - [ChatCommonConfig (required)](#chatcommonconfig-required)
  - [ChatThemeConfig (optional)](#chatthemeconfig-optional)
  - [ChatTextConfig (optional)](#chattextconfig-optional)
  - [Component-Specific Configs](#component-specific-configs)
- [Component Details](#component-details)
  - [ChatContainer](#chatcontainer)
  - [ChatInput](#chatinput)
  - [HistorySidebar (History)](#historysidebar-history)
  - [AssistantMessage](#assistantmessage)
  - [UserMessage](#usermessage)
  - [SidePanel](#sidepanel)
  - [Tool Components](#tool-components)
- [Backend Integration: ChatAdapter](#backend-integration-chatadapter)
  - [What is the ChatAdapter?](#what-is-the-chatadapter)
  - [ChatAdapter interface](#chatadapter-interface)
  - [Adapter method details](#adapter-method-details)
  - [Default adapter endpoints](#default-adapter-endpoints)
  - [Writing a custom adapter](#writing-a-custom-adapter)
- [Streaming Protocol](#streaming-protocol)
  - [How streaming works](#how-streaming-works)
  - [Chunk delimiter](#chunk-delimiter)
  - [Chunk types reference](#chunk-types-reference)
  - [Stream lifecycle](#stream-lifecycle)
  - [Auto-resume behavior](#auto-resume-behavior)
- [Theming & Styling](#theming--styling)
  - [How styles are isolated](#how-styles-are-isolated)
  - [CSS custom properties](#css-custom-properties)
  - [Overriding via ChatThemeConfig](#overriding-via-chatthemeconfig)
  - [Overriding via CSS](#overriding-via-css)
  - [Dark mode](#dark-mode)
- [Custom Tool Renderers](#custom-tool-renderers)
- [State Management](#state-management)
  - [Store architecture](#store-architecture)
  - [IChatStore](#ichatstore)
  - [IHistoryStore](#ihistorystore)
  - [IPreviewStore](#ipreviewstore)
  - [Accessing stores](#accessing-stores)
- [Exported Hooks](#exported-hooks)
- [TypeScript Types](#typescript-types)
- [Full Working Examples](#full-working-examples)
  - [Example 1: Minimal HTML page](#example-1-minimal-html-page)
  - [Example 2: React app with sidebar](#example-2-react-app-with-sidebar)
  - [Example 3: Custom adapter](#example-3-custom-adapter)
  - [Example 4: Custom theme (dark mode)](#example-4-custom-theme-dark-mode)
  - [Example 5: Chat-only (no sidebar)](#example-5-chat-only-no-sidebar)

---

## Integration Mode 1: Script Tag (IIFE)

This is the simplest way to add the chat widget to any page. No build tools, no npm, no React setup required.

### Step 1: Add the script

```html
<script src="path/to/chat.min.js"></script>
```

This creates a global `FynixChat` object with the following API:
- `FynixChat.render(componentName, selector, config, componentConfig?, adapter?)` — mount a component
- `FynixChat.unmount(selector)` — unmount a component
- `FynixChat.version` — package version string

### Step 2: Create mount targets

```html
<div id="sidebar"></div>
<div id="chat"></div>
```

### Step 3: Configure and render

```html
<script>
  // This config object is shared by all components
  const config = {
    apiBaseUrl: 'https://your-api.example.com/v1.0',
    getAuthHeaders: () => ({
      'Authorization': 'Bearer YOUR_TOKEN_HERE',
    }),
    productMode: 'devas',
    enableToolPreview: true,
    enableFileUpload: true,
  };

  // Mount the chat container
  FynixChat.render('ChatContainer', '#chat', config);

  // Mount the history sidebar
  FynixChat.render('History', '#sidebar', config, {
    showDeleteButton: true,
  });
</script>
```

### How render() works

```ts
FynixChat.render(
  componentName: string,              // Which component to mount (see Component Details)
  selector: string,                   // CSS selector for the DOM element to mount into
  commonConfig: ChatCommonConfig,     // Shared config (API URL, auth, features, theme)
  componentConfig?: ComponentConfig,  // Optional per-component config overrides
  adapter?: ChatAdapter               // Optional custom backend adapter
): { unmount: () => void }
```

**What happens internally:**
1. Finds the DOM element matching `selector`
2. Looks up the component in the registry by `componentName`
3. Creates (or reuses) shared Zustand stores based on the `commonConfig` object reference
4. Creates a React root via `createRoot()` and renders the component wrapped in `<ChatProvider>`
5. Returns an object with an `unmount()` method

### Shared state across components

When you call `render()` multiple times with the **same config object reference**, the components share the same Zustand stores. This means:

```js
// CORRECT — same object reference = shared state
const config = { apiBaseUrl: '...', getAuthHeaders: () => ({}) };
FynixChat.render('ChatContainer', '#chat', config);   // Uses store A
FynixChat.render('History', '#sidebar', config);       // Also uses store A — they stay in sync!

// WRONG — different object references = separate state
FynixChat.render('ChatContainer', '#chat', { apiBaseUrl: '...' });  // Uses store A
FynixChat.render('History', '#sidebar', { apiBaseUrl: '...' });     // Uses store B — NOT synced!
```

### Unmounting

```js
// Option 1: Use the returned handle
const handle = FynixChat.render('ChatContainer', '#chat', config);
handle.unmount();

// Option 2: Unmount by selector
FynixChat.unmount('#chat');
```

---

## Integration Mode 2: React Import

For React applications, you can import components directly and have full control over composition and layout.

### ChatProvider

All chat components must be wrapped in a `<ChatProvider>`. This provides:
- The config (API URL, auth, features, theme)
- The adapter (backend communication)
- The Zustand stores (chat state, history, preview)
- A `.fynix-chat-root` wrapper div with CSS variables applied

```tsx
import { ChatProvider } from '@fynix/chat-assistant';

<ChatProvider
  config={config}                // Required: ChatCommonConfig
  componentConfig={compConfig}   // Optional: component-specific overrides
  adapter={customAdapter}        // Optional: custom ChatAdapter (uses default if omitted)
  stores={externalStores}        // Optional: pre-created Zustand stores
>
  {children}
</ChatProvider>
```

| Prop | Type | Required | Description |
|------|------|----------|-------------|
| `config` | `ChatCommonConfig` | Yes | API configuration, feature flags, theming |
| `componentConfig` | `ComponentConfig` | No | Per-component overrides (merged into component config context) |
| `adapter` | `ChatAdapter` | No | Custom backend adapter. If omitted, a default adapter is created from `config` |
| `stores` | `{ chatStore, historyStore, previewStore }` | No | Pre-created Zustand stores. Use this to share state between multiple `ChatProvider` instances |

### Using individual components

```tsx
import {
  ChatProvider,
  ChatContainer,    // Full chat UI (messages + input + side panel)
  HistorySidebar,   // Thread list sidebar
  ChatInput,        // Standalone input (useful for custom layouts)
  AssistantMessage, // Single AI message (for custom message rendering)
  UserMessage,      // Single user message
  SidePanel,        // Tool preview panel
} from '@fynix/chat-assistant';

function App() {
  return (
    <ChatProvider config={config}>
      {/* Compose however you want */}
      <div className="flex h-screen">
        <HistorySidebar />
        <div className="flex-1">
          <ChatContainer />
        </div>
      </div>
    </ChatProvider>
  );
}
```

---

## Configuration Reference

### ChatCommonConfig (required)

This is the main configuration object. It must be passed to `render()` (script tag) or `<ChatProvider>` (React).

```ts
interface ChatCommonConfig {
  // ─── Required ──────────────────────────────────────────

  apiBaseUrl: string;
  // Base URL for all API calls. The default adapter appends
  // endpoint paths to this URL.
  // Example: 'https://api.example.com/v1.0'

  getAuthHeaders: () => Record<string, string>;
  // Function that returns authorization headers. Called before
  // every API request, so it can return fresh tokens.
  // Example: () => ({ 'Authorization': `Bearer ${getToken()}` })

  // ─── Feature Flags (all optional) ─────────────────────

  productMode?: 'devas' | 'dofle';
  // Product mode identifier sent to the backend.
  // Default: 'devas'

  enableToolPreview?: boolean;
  // When true, clicking tool results opens a side panel
  // with detailed previews (browser screenshots, code, etc.)
  // Default: undefined (falsy)

  enableFileUpload?: boolean;
  // When true, shows a file upload button in the chat input
  // and enables paste-to-upload for images.
  // Default: undefined (falsy)

  enableSuggestions?: boolean;
  // When true, fetches autocomplete suggestions as the user types.
  // Requires the adapter to implement fetchSuggestions().
  // Default: undefined (falsy)

  enableReactions?: boolean;
  // When true, shows like/dislike buttons on assistant messages.
  // Default: undefined (falsy)

  enableMessageEdit?: boolean;
  // When true, shows an edit button on user messages (on hover).
  // Editing a message re-sends it and truncates the thread.
  // Default: undefined (falsy)

  maxRetryCount?: number;
  // Maximum number of times the client will attempt to resume
  // a stream when the server indicates the task is still active.
  // Default: 10

  streamChunkDelimiter?: string;
  // Custom delimiter for splitting stream chunks.
  // Default: 'x-token-9f8a7c2bfa4e49bd83c6aef78b29c1d3: '

  // ─── Callbacks ─────────────────────────────────────────

  onNavigate?: (path: string) => void;
  // Called when the widget triggers a navigation event
  // (e.g., creating a new chat clears the path).
  // Useful for syncing URL state with the widget.

  onThreadChange?: (threadId: string | null) => void;
  // Called whenever the active thread changes. Use this to:
  //   - Update the URL (e.g., ?thread=abc123)
  //   - Track analytics
  //   - Sync with external state

  // ─── Custom Tool Renderers ─────────────────────────────

  toolRenderers?: Record<string, React.ComponentType<any>>;
  // Register custom React components to render specific tool
  // types in the side panel. Keys are tool type strings,
  // values are React components.
  // See "Custom Tool Renderers" section for details.

  // ─── Theming ───────────────────────────────────────────

  theme?: 'light' | 'dark' | 'auto';
  // Color scheme preference.
  //   'light' — always use light colors
  //   'dark'  — always use dark colors
  //   'auto'  — follow the user's OS preference
  // Default: 'light' (when omitted)

  themeConfig?: ChatThemeConfig;
  // Override specific colors, spacing, border radius, font.
  // See ChatThemeConfig section below.

  textConfig?: ChatTextConfig;
  // Override UI text strings (placeholders, labels, etc.)
  // See ChatTextConfig section below.
}
```

### ChatThemeConfig (optional)

Every field is optional. Each one maps to a CSS custom property that controls the widget's appearance. If you don't set a value, the default is used.

| Property | CSS Variable | Default | What it controls |
|----------|-------------|---------|------------------|
| `primaryColor` | `--fc-primary` | `#7c3aed` | Primary buttons, accents, active states |
| `primaryHoverColor` | `--fc-primary-hover` | `#6d28d9` | Primary button hover state |
| `userBubbleColor` | `--fc-user-bubble` | `#f3f4f6` | User message bubble background |
| `userBubbleBorder` | `--fc-user-bubble-border` | `#e5e7eb` | User message bubble border |
| `assistantBorderColor` | `--fc-assistant-border` | `#e5e7eb` | Assistant message border, input border, dividers |
| `backgroundColor` | `--fc-bg` | `#ffffff` | Main background color |
| `textColor` | `--fc-text` | `#111827` | Primary text color |
| `secondaryTextColor` | `--fc-text-secondary` | `#6b7280` | Secondary/muted text color |
| `borderColor` | `--fc-border` | `#e5e7eb` | General borders |
| `surfaceColor` | `--fc-surface` | `#f9fafb` | Surface/card backgrounds |
| `surfaceHoverColor` | `--fc-surface-hover` | `#f3f4f6` | Surface hover state |
| `successColor` | `--fc-success` | `#22c55e` | Success indicators ("Answer completed") |
| `errorColor` | `--fc-error` | `#ef4444` | Error messages |
| `warningColor` | `--fc-warning` | `#eab308` | Warning indicators |
| `linkColor` | `--fc-link` | `#3b82f6` | Links in markdown content |
| `codeBackground` | `--fc-code-bg` | `#f9fafb` | Code block backgrounds |
| `fontFamily` | `--fc-font-family` | System font stack | Font for all widget text |
| `messagePaddingX` | `--fc-spacing-msg-x` | `1rem` | Horizontal padding inside messages |
| `messagePaddingY` | `--fc-spacing-msg-y` | `0.75rem` | Vertical padding inside messages |
| `messageGap` | `--fc-spacing-msg-gap` | `0.5rem` | Gap between message elements |
| `containerPadding` | `--fc-spacing-container` | `1rem` | Outer padding of the chat container |
| `borderRadius` | `--fc-radius` | `0.75rem` | Border radius for cards, messages |
| `inputBorderRadius` | `--fc-radius-input` | `1rem` | Border radius for the chat input |

### ChatTextConfig (optional)

Every user-facing string in the widget can be customized. This is useful for localization or branding.

| Property | Default | Where it appears |
|----------|---------|------------------|
| `inputPlaceholder` | `'How can I help you today?'` | Chat input placeholder |
| `newChatButtonText` | `'New Chat'` | New chat button in sidebar |
| `emptyStateTitle` | (none) | Title shown when chat has no messages |
| `emptyStateSubtitle` | (none) | Subtitle below empty state title |
| `loadingText` | `'Loading chat history...'` | Shown while loading a thread |
| `errorText` | `'Failed to load chat'` | Shown on load error |
| `thinkingText` | `'Thinking'` | Animated status while AI is processing |
| `answerCompletedText` | `'Answer completed'` | Shown below completed AI responses |
| `helpfulText` | `'Is this answer helpful?'` | Label next to like/dislike buttons |
| `followUpTitle` | `'Suggested Follow-ups:'` | Header for follow-up question chips |
| `noConversationsText` | `'No conversations yet'` | Shown in empty sidebar |
| `saveAndSendText` | `'Save & Send'` | Button text when editing a message |
| `cancelText` | `'Cancel'` | Cancel button when editing a message |
| `feedbackText` | (none) | Feedback prompt text |
| `stepsCompletedText` | (none) | Tool steps completed label |
| `stepsErrorText` | (none) | Tool steps error label |
| `processingStepsText` | (none) | Tool steps processing label |
| `showDetailsText` | (none) | "Show details" toggle label |
| `hideDetailsText` | (none) | "Hide details" toggle label |
| `previewText` | (none) | Preview button label |
| `liveText` | (none) | Live indicator label |
| `copyCodeText` | (none) | Copy code button tooltip |
| `downloadCodeText` | (none) | Download code button tooltip |
| `somethingWentWrongText` | (none) | Generic error fallback |
| `noThoughtsText` | (none) | Shown when no thoughts are available |

### Component-Specific Configs

These are passed as the 4th argument to `render()` or via the `componentConfig` prop on `<ChatProvider>`. They override the corresponding `textConfig` values for that specific component.

#### ChatContainerConfig

Used when rendering `'ChatContainer'`.

```ts
interface ChatContainerConfig {
  enableComputerTool?: boolean;
  // Show a mini preview button that opens the computer tool
  // side panel when tool results are available.
  // Default: true

  emptyStateTitle?: string;
  // Overrides config.textConfig.emptyStateTitle for this component

  emptyStateSubtitle?: string;
  // Overrides config.textConfig.emptyStateSubtitle for this component
}
```

#### ChatInputComponentConfig

Used when rendering `'ChatInput'`.

```ts
interface ChatInputComponentConfig {
  placeholder?: string;
  // Overrides config.textConfig.inputPlaceholder
  // Default: 'How can I help you today?'

  maxRows?: number;
  // Maximum number of rows the textarea can grow to
  // Default: 7

  minRows?: number;
  // Minimum number of rows the textarea starts with
  // Default: 2

  maxFileUploads?: number;
  // Maximum number of files that can be attached at once
  // Default: 5

  allowedFileTypes?: string[];
  // Array of allowed MIME types for file uploads
  // Default: ['image/jpeg', 'image/png', 'image/jpg', 'image/webp']
}
```

#### HistoryComponentConfig

Used when rendering `'History'`.

```ts
interface HistoryComponentConfig {
  sidebarWidth?: string;
  // CSS width value for the sidebar
  // Default: '16rem' (256px)

  showDeleteButton?: boolean;
  // Whether to show the delete button on thread items
  // Default: true

  showSearch?: boolean;
  // Whether to show a search input in the sidebar

  newChatButtonText?: string;
  // Overrides config.textConfig.newChatButtonText

  noConversationsText?: string;
  // Overrides config.textConfig.noConversationsText
}
```

#### MessagesComponentConfig

```ts
interface MessagesComponentConfig {
  showCopyButton?: boolean;
  // Whether to show copy-to-clipboard on messages

  maxUserMessageWidth?: string;
  // CSS max-width for user message bubbles
}
```

---

## Component Details

### ChatContainer

**Registry name:** `'ChatContainer'`

The main all-in-one chat component. It includes:
- Auto-scrolling message list (user and assistant messages)
- Chat input with file upload and send/stop button
- Side panel for tool previews (when `enableToolPreview` is true)
- Empty state with customizable title and subtitle
- Loading and error states
- Computer tool preview mini-button

**Props (when used as React component):**

| Prop | Type | Default | Description |
|------|------|---------|-------------|
| `loading` | `boolean` | `false` | Show loading spinner |
| `error` | `string \| null` | `null` | Show error message |

### ChatInput

**Registry name:** `'ChatInput'`

A standalone chat input component. Features:
- Auto-resizing textarea (grows with content up to `maxRows`)
- Send button (enabled when there's text or uploaded files)
- Stop button (when streaming is active, sends cancel request)
- File upload button (when `enableFileUpload` is true in config)
- File preview thumbnails with remove/retry actions
- Paste-to-upload for images
- Enter to send, Shift+Enter for new line

**Props:**

| Prop | Type | Default | Description |
|------|------|---------|-------------|
| `initialQuery` | `string` | (none) | Pre-fill the input with a query |

### HistorySidebar (History)

**Registry name:** `'History'`

Thread list sidebar. Features:
- "New Chat" button at the top
- Scrollable list of thread items
- Active thread highlighting
- Thread status icons (running, completed)
- Delete button on hover (configurable)
- Empty state message when no threads exist

Clicking a thread item sets it as the active thread in the chat store, and calls `config.onThreadChange()`.

### AssistantMessage

**Registry name:** `'AssistantMessage'`

Renders a single AI response. Features:
- Markdown rendering with GitHub Flavored Markdown (tables, code blocks, etc.)
- Syntax-highlighted code blocks with copy and download buttons
- Tool execution steps list (expandable)
- Streaming thoughts display (clickable to open in side panel)
- "Answer completed" footer with success indicator
- Like/dislike reaction buttons
- Follow-up question suggestions
- Animated "Thinking..." progress indicator during streaming

**Props:**

| Prop | Type | Default | Description |
|------|------|---------|-------------|
| `message` | `IAssistantMessage` | (required) | The assistant message object |
| `isStreaming` | `boolean` | `false` | Whether this message is currently streaming |
| `progressMessage` | `string` | (none) | Text for the progress indicator |
| `showActions` | `boolean` | `true` | Show thoughts button, reactions, follow-ups |

### UserMessage

**Registry name:** `'UserMessage'`

Renders a single user message bubble. Features:
- Right-aligned bubble with custom styling
- Image attachments with click-to-preview
- Copy-to-clipboard button (on hover)
- Edit button (on hover, when `enableMessageEdit` is true)
- Inline edit mode with textarea, save/cancel buttons
- Editing truncates the thread and re-sends the message

**Props:**

| Prop | Type | Default | Description |
|------|------|---------|-------------|
| `message` | `IUserMessage` | (required) | The user message object |
| `images` | `UploadedImage[]` | (none) | Attached images |
| `isEditable` | `boolean` | `false` | Show edit button on hover |
| `onEditSubmit` | `(messageId: string, newContent: string) => void` | (none) | Callback when edit is saved |

### SidePanel

**Registry name:** `'SidePanel'`

An animated side panel that shows tool previews. Features:
- Slides in from the right with Framer Motion animation
- Full-screen on mobile with backdrop overlay
- Desktop: positioned alongside the chat container
- Routes to different preview components based on `type`:
  - `'computer'` → Computer tool preview with progress scrubbing
  - `'streaming_toughts_preview'` → Full thoughts viewer
  - Custom types → Your registered `toolRenderers`

### Tool Components

These components are used internally by the SidePanel but can also be imported directly for custom layouts:

```ts
import {
  BrowserTool,       // Renders browser screenshots with URL bar
  CodeViewerTool,    // Syntax-highlighted code viewer
  RunCodeTool,       // Code execution output viewer
  SearchEngineTool,  // Search results list (title, URL, description)
  IframeTool,        // Iframe/VNC preview for remote environments
  MarkdownTool,      // Markdown content viewer
} from '@fynix/chat-assistant';
```

---

## Backend Integration: ChatAdapter

### What is the ChatAdapter?

The `ChatAdapter` is an interface that abstracts all communication between the widget and your backend. Think of it as a data access layer — the widget calls adapter methods, and the adapter translates those calls into whatever HTTP requests your backend expects.

**You have two options:**
1. **Use the default adapter** — it calls specific REST endpoints that your backend must implement (see [Default Adapter Endpoints](#default-adapter-endpoints))
2. **Write a custom adapter** — implement the `ChatAdapter` interface to connect to any backend

### ChatAdapter interface

```ts
interface ChatAdapter {
  // Required methods
  startStream(params: StartStreamParams): Promise<ReadableStream>;
  cancelStream(threadId: string): Promise<void>;
  resumeStream(threadId: string): Promise<ReadableStream>;
  fetchThreads(params: FetchThreadsParams): Promise<FetchThreadsResponse>;
  deleteThread(threadId: string): Promise<void>;
  fetchThreadMessages(threadId: string, signal?: AbortSignal): Promise<ThreadMessagesResponse>;

  // Optional methods
  fetchSuggestions?(query: string, limit: number, signal?: AbortSignal): Promise<SuggestionResponse>;
  uploadFile?(file: File): Promise<UploadedImage>;
  submitReaction?(threadId: string, messageId: string, reaction: MessageReaction): Promise<void>;
}
```

### Adapter method details

#### `startStream(params)`

Starts a new chat interaction. Must return a `ReadableStream` of text chunks following the [streaming protocol](#streaming-protocol).

**Parameters (`StartStreamParams`):**

```ts
{
  message: string;                  // The user's message text
  threadId: string | null;          // null for new chats, thread ID for continuing
  isNewChat: boolean;               // true if this is a brand new conversation
  productMode: 'devas' | 'dofle';  // Product mode from config
  parentMessageId: string | null;   // ID of the last message in the thread
  images?: UploadedImage[];         // Attached images (from file upload)
  selectedKbs?: SelectedKB[];       // Selected knowledge bases
  outputResponseFormat?: string;    // Requested response format
  extraHeaders?: Record<string, string>; // Additional headers
}
```

#### `cancelStream(threadId)`

Cancels an in-progress stream. Called when the user clicks the stop button.

#### `resumeStream(threadId)`

Resumes a stream that was interrupted. Used for long-running tasks — when the initial stream ends but the server reports the task is still active, the widget calls this method to continue receiving chunks.

#### `fetchThreads(params)`

Fetches the list of conversation threads for the history sidebar.

```ts
// Parameters
{
  page: number;      // Page number (1-based)
  email: string;     // User's email for filtering
  query?: string;    // Optional search query
  product?: string;  // Optional product filter
}

// Expected response
{
  threads: IThread[];
  pagination: {
    has_next: boolean;
    page: number;
    total: number;
  };
}
```

#### `fetchThreadMessages(threadId, signal?)`

Loads all messages for a specific thread. Called when the user clicks a thread in the sidebar.

```ts
// Expected response
{
  success: boolean;
  data: {
    messages: any[];       // Array of message objects
    active_task: boolean;  // If true, widget will auto-resume streaming
  };
}
```

#### `uploadFile(file)` (optional)

Uploads a file (typically an image). If not implemented, file upload buttons won't work even if `enableFileUpload` is true.

```ts
// Expected response
{
  type: string;       // MIME type
  filename: string;   // Server-assigned filename
  url: string;        // Accessible URL for the uploaded file
}
```

#### `submitReaction(threadId, messageId, reaction)` (optional)

Submits a like or dislike reaction on an assistant message.

```ts
// Reaction object
{
  type: 'liked' | 'disliked';
  dislike_reason_id: number | null;
  details: string | null;
}
```

#### Supporting types

```ts
interface UploadedImage {
  type: string;       // MIME type
  filename: string;
  url: string;
}

interface SelectedKB {
  kb_id: string;
  name: string;
}

interface MessageReaction {
  type: 'liked' | 'disliked';
  dislike_reason_id: number | null;
  details: string | null;
}

interface IThread {
  id: string;
  title: string;
  last_message_id?: string;
  created_at?: string;
  updated_at?: string;
  user_id?: string;
  product?: string;
  is_running?: boolean;
}
```

### Default adapter endpoints

When you don't provide a custom adapter, the widget creates one using `createDefaultAdapter(config)`. This adapter calls the following REST endpoints against `config.apiBaseUrl`:

| Adapter Method | HTTP Method | Endpoint Path | Request Body |
|---------------|------------|---------------|-------------|
| `startStream` | POST | `/streaming/chat/start` | `{ message: { content }, product, parent_message_id, stream: true, thread_id, metadata: { response_length, selected_kbs, response_format }, images }` |
| `cancelStream` | POST | `/streaming/cancel/{threadId}/task` | (none) |
| `resumeStream` | GET | `/streaming/chunks/{threadId}` | (none) |
| `fetchThreads` | GET | `/threads/?page=&user_email=&product=&page_size=25&query=&agent_type=conversational` | (none) |
| `deleteThread` | DELETE | `/threads/{threadId}` | (none) |
| `fetchThreadMessages` | GET | `/threads/thread/{threadId}/messages` | (none) |
| `fetchSuggestions` | GET | `/prompts/suggestions?query=&limit=` | (none) |
| `uploadFile` | POST | `/upload` | `FormData` with `file` field |
| `submitReaction` | POST | `/threads/{threadId}/messages/{messageId}/reaction` | `{ type, dislike_reason_id, details }` |

All requests include:
- Headers from `getAuthHeaders()`
- `Content-Type: application/json` (except `uploadFile` which uses multipart)

### Writing a custom adapter

Here's a complete example of a custom adapter:

```ts
import { ChatAdapter, StartStreamParams } from '@fynix/chat-assistant';

function getAuth() {
  return { 'Authorization': `Bearer ${localStorage.getItem('token')}` };
}

export const myAdapter: ChatAdapter = {
  async startStream(params: StartStreamParams): Promise<ReadableStream> {
    const res = await fetch('https://my-api.com/chat/stream', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json', ...getAuth() },
      body: JSON.stringify({
        message: params.message,
        thread_id: params.threadId,
        is_new: params.isNewChat,
        product: params.productMode,
        parent_id: params.parentMessageId,
        images: params.images || [],
      }),
    });
    if (!res.ok) throw new Error(`Stream failed: ${res.status}`);
    return res.body!;
  },

  async cancelStream(threadId: string): Promise<void> {
    await fetch(`https://my-api.com/chat/${threadId}/cancel`, {
      method: 'POST',
      headers: getAuth(),
    });
  },

  async resumeStream(threadId: string): Promise<ReadableStream> {
    const res = await fetch(`https://my-api.com/chat/${threadId}/stream`, {
      headers: getAuth(),
    });
    if (!res.ok) throw new Error(`Resume failed: ${res.status}`);
    return res.body!;
  },

  async fetchThreads(params) {
    const res = await fetch(
      `https://my-api.com/threads?page=${params.page}&email=${params.email}`,
      { headers: { ...getAuth(), 'Content-Type': 'application/json' } }
    );
    const data = await res.json();
    return {
      threads: data.threads,
      pagination: data.pagination,
    };
  },

  async deleteThread(threadId: string): Promise<void> {
    await fetch(`https://my-api.com/threads/${threadId}`, {
      method: 'DELETE',
      headers: getAuth(),
    });
  },

  async fetchThreadMessages(threadId: string, signal?: AbortSignal) {
    const res = await fetch(`https://my-api.com/threads/${threadId}/messages`, {
      headers: { ...getAuth(), 'Content-Type': 'application/json' },
      signal,
    });
    return res.json();
  },

  // Optional methods
  async uploadFile(file: File) {
    const form = new FormData();
    form.append('file', file);
    const res = await fetch('https://my-api.com/upload', {
      method: 'POST',
      headers: getAuth(),
      body: form,
    });
    const data = await res.json();
    return data; // { type, filename, url }
  },

  async submitReaction(threadId, messageId, reaction) {
    await fetch(`https://my-api.com/threads/${threadId}/messages/${messageId}/reaction`, {
      method: 'POST',
      headers: { ...getAuth(), 'Content-Type': 'application/json' },
      body: JSON.stringify(reaction),
    });
  },
};
```

---

## Streaming Protocol

### How streaming works

The chat widget uses server-sent streaming to receive AI responses in real-time. When the user sends a message:

1. The widget calls `adapter.startStream()` which returns a `ReadableStream`
2. The stream is read chunk by chunk using a `TextDecoder`
3. Each chunk is split by the delimiter and parsed as JSON
4. Each parsed chunk is routed to the appropriate handler based on its `type` field
5. The UI updates reactively via Zustand stores

### Chunk delimiter

Chunks in the stream are separated by this delimiter string:

```
x-token-9f8a7c2bfa4e49bd83c6aef78b29c1d3:
```

Note the trailing space after the colon. When the client reads a chunk of text from the stream, it splits by this delimiter and processes each piece independently.

### Chunk types reference

Each chunk is a JSON object with a `type` field. Here's what each type does:

| Type | When it appears | What `content` contains | What the widget does |
|------|----------------|------------------------|---------------------|
| `thread_created` | Start of a new conversation | The server-assigned thread ID (string) | Creates a new thread in the store, updates the active thread ID, syncs the sidebar |
| `thread_title` | After thread creation | The generated conversation title (string) | Sets the thread title, adds it to the sidebar |
| `assistant_message_created` | Before response content | The server-assigned message ID (string) | Stores the message ID for future reference (reactions, etc.) |
| `streaming` | During thinking/reasoning | Partial text (string, appended to previous) | Appends to the "thoughts" section of the assistant message. Multiple `streaming` chunks are concatenated |
| `supervisor_streaming` | During final answer generation | Partial markdown text (string, appended) | Renders as markdown in the main response area. Chunks are concatenated for continuous display |
| `security_block` | When content is filtered | Block/warning message (string) | Displayed as the response content (same rendering as `supervisor_streaming`) |
| `toolStart` | When a tool invocation begins | Tool metadata | Added to the response steps list |
| `toolUsed` | When a tool completes | Tool result with `detail.content` (JSON string) | Added to response steps list AND to the computer tool progress tracker (for side panel previews) |
| `consolidated_data` | Aggregated data | Data payload (charts, tables, etc.) | Appended as a chunk in the assistant message |
| `next_questions` | End of response | JSON array of suggested questions (string) | Parsed and displayed as clickable follow-up question chips |

### Stream lifecycle

```
User sends message
        |
        v
  startStream() --> ReadableStream
        |
        v (for new chats)
  { type: "thread_created", content: "thread_abc123" }
        |
        v
  { type: "thread_title", content: "How to deploy a React app" }
        |
        v
  { type: "assistant_message_created", content: "msg_xyz789" }
        |
        v (thinking phase - may have multiple chunks)
  { type: "streaming", content: "Let me think about this..." }
  { type: "streaming", content: " I need to consider..." }
        |
        v (tool usage - optional, may repeat)
  { type: "toolStart", name: "search_web_tool", ... }
  { type: "toolUsed", name: "search_web_tool", detail: { content: "..." }, ... }
        |
        v (final answer - multiple chunks)
  { type: "supervisor_streaming", content: "Here's how to " }
  { type: "supervisor_streaming", content: "deploy your React app:\n\n1. " }
  { type: "supervisor_streaming", content: "Build the project..." }
        |
        v (optional follow-ups)
  { type: "next_questions", content: "[\"How to add CI/CD?\", \"How to add SSL?\"]" }
        |
        v
  Stream ends (ReadableStream done)
        |
        v
  fetchThreadMessages() --> Check active_task
        |
        |-- active_task: true  --> resumeStream() (repeat)
        '-- active_task: false --> Done, mark thread as completed
```

### Auto-resume behavior

Some tasks take longer than a single stream connection. When the stream ends:

1. The widget calls `fetchThreadMessages(threadId)` to check the thread status
2. If `response.data.active_task` is `true`, the task is still running on the server
3. The widget calls `resumeStream(threadId)` to get a new stream and continues processing
4. This repeats up to `maxRetryCount` times (default: 10)
5. If the retry limit is reached, the thread is marked as completed regardless

---

## Theming & Styling

### How styles are isolated

The widget uses two mechanisms to prevent style conflicts with your host application:

1. **Tailwind prefix**: All Tailwind utility classes use the `fc-` prefix.
   Example: `fc-flex`, `fc-text-sm`, `fc-bg-white` instead of `flex`, `text-sm`, `bg-white`

2. **Important selector**: Tailwind's `important` option is set to `.fynix-chat-root`.
   This means all generated styles are scoped under `.fynix-chat-root`, giving them higher specificity than most host app styles, but only within the widget's DOM subtree.

3. **CSS variable namespace**: All CSS custom properties use the `--fc-` prefix to avoid conflicts.

### CSS custom properties

The widget's visual appearance is controlled entirely by CSS custom properties. These are set on the `.fynix-chat-root` element (which wraps every widget component):

```css
.fynix-chat-root {
  /* Colors */
  --fc-primary: #7c3aed;           /* Primary brand color */
  --fc-primary-hover: #6d28d9;     /* Primary hover state */
  --fc-user-bubble: #f3f4f6;       /* User message background */
  --fc-user-bubble-border: #e5e7eb;/* User message border */
  --fc-assistant-border: #e5e7eb;  /* Assistant message & input borders */
  --fc-bg: #ffffff;                /* Main background */
  --fc-text: #111827;              /* Primary text */
  --fc-text-secondary: #6b7280;   /* Secondary text */
  --fc-border: #e5e7eb;           /* General borders */
  --fc-surface: #f9fafb;          /* Surface/card backgrounds */
  --fc-surface-hover: #f3f4f6;    /* Surface hover */
  --fc-success: #22c55e;          /* Success states */
  --fc-error: #ef4444;            /* Error states */
  --fc-warning: #eab308;          /* Warning states */
  --fc-link: #3b82f6;             /* Links */
  --fc-code-bg: #f9fafb;          /* Code block backgrounds */

  /* Typography */
  --fc-font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;

  /* Spacing */
  --fc-spacing-msg-x: 1rem;       /* Message horizontal padding */
  --fc-spacing-msg-y: 0.75rem;    /* Message vertical padding */
  --fc-spacing-msg-gap: 0.5rem;   /* Gap between message elements */
  --fc-spacing-container: 1rem;   /* Container padding */

  /* Border radius */
  --fc-radius: 0.75rem;           /* Default border radius */
  --fc-radius-input: 1rem;        /* Input field border radius */
}
```

### Overriding via ChatThemeConfig

The easiest way to customize the theme — pass a `themeConfig` object in your config:

```js
const config = {
  apiBaseUrl: '...',
  getAuthHeaders: () => ({}),
  themeConfig: {
    primaryColor: '#2563eb',      // Blue instead of purple
    backgroundColor: '#0f172a',   // Dark background
    textColor: '#f1f5f9',         // Light text
    borderRadius: '0.5rem',       // Smaller corners
  },
};
```

### Overriding via CSS

You can also override CSS variables directly in your stylesheet:

```css
/* Override in your host app's CSS */
.fynix-chat-root {
  --fc-primary: #2563eb;
  --fc-bg: #0f172a;
  --fc-text: #f1f5f9;
}
```

### Dark mode

Set `theme: 'dark'` or `theme: 'auto'` in your config, and provide dark-appropriate colors:

```js
const config = {
  theme: 'dark',
  themeConfig: {
    primaryColor: '#818cf8',
    backgroundColor: '#0f172a',
    textColor: '#e2e8f0',
    secondaryTextColor: '#94a3b8',
    borderColor: '#334155',
    surfaceColor: '#1e293b',
    surfaceHoverColor: '#334155',
    userBubbleColor: '#1e293b',
    userBubbleBorder: '#334155',
    assistantBorderColor: '#334155',
    codeBackground: '#1e293b',
  },
};
```

With `theme: 'auto'`, the `useTheme()` hook listens to the OS `prefers-color-scheme` media query and returns the current scheme. You can use this to dynamically switch theme configs.

---

## Custom Tool Renderers

The side panel can display previews for different tool types. Two types are built-in (`computer` and `streaming_toughts_preview`), but you can register your own.

### Registering a custom renderer

```ts
const config = {
  apiBaseUrl: '...',
  getAuthHeaders: () => ({}),
  toolRenderers: {
    'data_visualization': DataVizComponent,
    'code_sandbox': CodeSandboxComponent,
  },
};
```

### Creating a renderer component

Your component receives a `previewObject` prop with the preview data:

```tsx
function DataVizComponent({ previewObject }) {
  // previewObject shape:
  // {
  //   title: string;
  //   description?: string;
  //   image?: string;
  //   textContent?: string;
  //   streamingThoughts?: string[];
  //   type?: string;           // matches the key in toolRenderers
  //   images?: UploadedImage[];
  //   pptKey?: string;
  //   pptSerialNo?: number;
  // }

  return (
    <div>
      <h3>{previewObject.title}</h3>
      <p>{previewObject.description}</p>
      {/* Your custom rendering logic */}
    </div>
  );
}
```

### How it works

1. When a `toolUsed` chunk is processed, it's added to the progress tracker
2. Clicking on a tool step (or the mini preview button) sets the `AgentPreviewObject` in the preview store
3. The `SidePanel` component checks `AgentPreviewObject.type`:
   - `'computer'` → renders `ComputerToolPreview`
   - `'streaming_toughts_preview'` → renders `StreamingThoughtsPreview`
   - Anything else → looks up `config.toolRenderers[type]` and renders your component

---

## State Management

### Store architecture

The widget uses three independent Zustand stores. This separation keeps concerns clean and allows fine-grained subscriptions:

```
+-------------------------------------------+
|               ChatStore                    |
|  - threadsById: Record<id, ThreadState>   |
|  - activeThreadId                          |
|  - productMode                             |
|  - Per-thread: messages, streaming,       |
|    abortController, progressTracker       |
+-------------------------------------------+

+-------------------------------------------+
|             HistoryStore                   |
|  - agentThreads: IThread[]                |
|  - concurrentThreads: Record<id, status>  |
|  - currentActiveThread                     |
|  - historyLoaded                           |
+-------------------------------------------+

+-------------------------------------------+
|             PreviewStore                   |
|  - AgentPreviewObject                      |
|  - threadChartObject                       |
|  - chatAttachmentPreview                   |
|  - currentChatAttachmentIndex              |
+-------------------------------------------+
```

### IChatStore

The main store for chat state. Manages all thread data, messages, and streaming.

**Key state:**
- `threadsById` — Map of thread ID to thread state (messages, streaming status, abort controller, progress tracker)
- `activeThreadId` — Currently active thread
- `productMode` — Current product mode (`'devas'` or `'dofle'`)

**Key actions:**
- `setActiveThreadId(id)` — Switch active thread
- `addNewThread(id)` — Create a new thread entry
- `appendUserMessage(threadId, message)` — Add a user message
- `appendAssistantMessage(threadId, chunk)` — Add/append to assistant message
- `setThreadStreaming(threadId, bool)` — Set streaming state
- `editAndResendMessage(threadId, messageId, newContent)` — Edit a sent message (truncates thread)

### IHistoryStore

Manages the thread list shown in the sidebar.

**Key state:**
- `agentThreads` — Array of `IThread` objects
- `concurentThreads` — Map of thread ID to `ConcurrentThreadStatus` (running, completed, etc.)
- `currentActiveThread` — Currently selected thread info

**Key actions:**
- `setAgentThreads(threads)` — Replace the thread list
- `appendThread(thread)` — Add a new thread to the top of the list
- `deleteThread(threadId)` — Remove a thread
- `setConcurrentThreadStatus(threadId, status)` — Update thread running status

### IPreviewStore

Manages side panel preview state.

**Key state:**
- `AgentPreviewObject` — The current preview data (or null when side panel is closed)
- `chatAttachmentPreview` — Images being previewed
- `currentChatAttachmentIndex` — Currently viewed image index

**Key actions:**
- `setAgentPreviewObject(obj)` — Open side panel with preview data (set to null to close)
- `setChatAttachmentPreview(images)` — Set images for the image viewer
- `goToPreviousAttachment()` / `goToNextAttachment()` — Navigate images

### Accessing stores

From within `<ChatProvider>`:

```tsx
import { useChatStore, useHistoryStore, usePreviewStore } from '@fynix/chat-assistant';

function MyComponent() {
  // Get the store API (for imperative access)
  const chatStore = useChatStore();
  const activeId = chatStore.getState().activeThreadId;

  // Or use the selector hooks for reactive updates
  const activeThreadId = useChatStoreState((s) => s.activeThreadId);
  const isStreaming = useChatStoreState((s) =>
    activeThreadId ? s.threadsById[activeThreadId]?.streaming : false
  );
}
```

Creating stores externally (for sharing between providers):

```tsx
import { createChatStore, createHistoryStore, createPreviewStore, ChatProvider } from '@fynix/chat-assistant';

const stores = {
  chatStore: createChatStore(),
  historyStore: createHistoryStore(),
  previewStore: createPreviewStore(),
};

// Both providers share the same stores
<ChatProvider config={config1} stores={stores}>
  <ChatContainer />
</ChatProvider>
<ChatProvider config={config2} stores={stores}>
  <HistorySidebar />
</ChatProvider>
```

---

## Exported Hooks

All hooks must be used inside a `<ChatProvider>`.

### Context hooks

```ts
import {
  useChatStore,        // () => StoreApi<IChatStore>
  useHistoryStore,     // () => StoreApi<IHistoryStore>
  usePreviewStore,     // () => StoreApi<IPreviewStore>
  useAdapter,          // () => ChatAdapter
  useConfig,           // () => ChatCommonConfig
  useComponentConfig,  // <T>() => T
} from '@fynix/chat-assistant';
```

### Utility hooks

#### `useFileUpload(options?)`

Manages file upload state with automatic upload to the backend.

```ts
const upload = useFileUpload({
  maxFiles: 5,                // Max files allowed (default: 5)
  allowedTypes: ['image/png'], // Allowed MIME types (default: jpeg, png, jpg, webp)
});

// upload.files           -> FileUploadItem[] (current files with status)
// upload.uploadFiles(files) -> Upload new files
// upload.removeFile(id)    -> Remove a file
// upload.retryFile(id)     -> Retry a failed upload
// upload.handleFileInput(e) -> onChange handler for <input type="file">
// upload.handlePaste(e)    -> onPaste handler (supports image paste)
// upload.isUploading       -> boolean (any file currently uploading?)
// upload.hasErrors         -> boolean (any file has an error?)
// upload.getUploadedImages() -> UploadedImage[] (successfully uploaded)
// upload.clearAll()        -> Remove all files
```

#### `useTheme()`

Returns the current color scheme based on config and OS preference.

```ts
const { colorScheme } = useTheme();
// colorScheme: 'light' | 'dark'
```

#### `useThemeVars(themeConfig?)`

Converts a `ChatThemeConfig` object into a `React.CSSProperties` object with CSS custom properties.

```ts
const style = useThemeVars({ primaryColor: '#2563eb' });
// style: { '--fc-primary': '#2563eb', '--fc-primary-hover': '#6d28d9', ... }
```

### Store factory functions

These create new Zustand store instances. Use them when you need to share stores across multiple providers or manage stores externally.

```ts
import {
  createChatStore,      // () => StoreApi<IChatStore>
  createHistoryStore,   // () => StoreApi<IHistoryStore>
  createPreviewStore,   // () => StoreApi<IPreviewStore>
  createDefaultAdapter, // (config: ChatCommonConfig) => ChatAdapter
} from '@fynix/chat-assistant';
```

---

## TypeScript Types

All types are exported and can be imported directly:

```ts
import type {
  // Config types
  ChatConfig,            // Alias for ChatCommonConfig
  ChatCommonConfig,
  ChatThemeConfig,
  ChatTextConfig,
  ChatContainerConfig,
  ChatInputComponentConfig,
  HistoryComponentConfig,
  MessagesComponentConfig,
  ComponentConfig,

  // Adapter types
  ChatAdapter,

  // Message types
  IAssistantMessage,
  IUserMessage,
  MessageReaction,
  UploadedImage,
  SelectedKB,

  // Thread types
  IThread,
  IThreadState,
  ProductMode,
  ConcurrentThreadStatus,

  // Tool types
  IToolType,
  IToolHeaderMessage,
  ProgressTrackerItem,
  BrowserToolProps,
  IframeToolProps,
  PlanToolProps,
  SearchResult,

  // Preview types
  AgentPreviewObjectWithParams,

  // Hook types
  FileUploadItem,
  UseFileUploadOptions,
  UseFileUploadReturn,
} from '@fynix/chat-assistant';
```

---

## Full Working Examples

### Example 1: Minimal HTML page

The simplest possible integration — just a script tag and a few lines of JavaScript.

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>Chat Widget</title>
  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; }
    body { font-family: sans-serif; }
    .layout { display: flex; height: 100vh; }
    #sidebar { width: 260px; border-right: 1px solid #e5e7eb; }
    #chat { flex: 1; }
  </style>
</head>
<body>
  <div class="layout">
    <div id="sidebar"></div>
    <div id="chat"></div>
  </div>

  <script src="path/to/chat.min.js"></script>
  <script>
    // 1. Create the shared config object
    const commonConfig = {
      apiBaseUrl: 'https://your-api.example.com/v1.0',
      getAuthHeaders: () => ({
        'Authorization': 'Bearer YOUR_TOKEN_HERE',
      }),
      productMode: 'devas',
      enableToolPreview: true,
      enableFileUpload: true,
      enableMessageEdit: true,

      // Customize the look
      themeConfig: {
        primaryColor: '#7c3aed',
      },

      // Customize the text
      textConfig: {
        inputPlaceholder: 'How can I help you today?',
        emptyStateTitle: 'Welcome to Chat',
        emptyStateSubtitle: 'Ask me anything to get started',
      },

      // Sync thread ID with URL
      onThreadChange: (threadId) => {
        if (threadId) {
          window.history.replaceState(null, '', `?thread=${threadId}`);
        }
      },
    };

    // 2. Mount the components (same config = shared state)
    FynixChat.render('ChatContainer', '#chat', commonConfig, {
      enableComputerTool: true,
    });

    FynixChat.render('History', '#sidebar', commonConfig, {
      showDeleteButton: true,
      newChatButtonText: 'New Chat',
    });
  </script>
</body>
</html>
```

### Example 2: React app with sidebar

```tsx
import React from 'react';
import {
  ChatProvider,
  ChatContainer,
  HistorySidebar,
  ChatCommonConfig,
} from '@fynix/chat-assistant';

const config: ChatCommonConfig = {
  apiBaseUrl: 'https://your-api.example.com/v1.0',
  getAuthHeaders: () => ({
    'Authorization': `Bearer ${localStorage.getItem('token')}`,
  }),
  productMode: 'devas',
  enableToolPreview: true,
  enableFileUpload: true,
  enableMessageEdit: true,
  enableReactions: true,
  themeConfig: {
    primaryColor: '#2563eb',
    borderRadius: '0.5rem',
  },
  textConfig: {
    emptyStateTitle: 'Welcome',
    emptyStateSubtitle: 'How can I help you today?',
  },
  onThreadChange: (threadId) => {
    console.log('Active thread:', threadId);
  },
};

export default function App() {
  return (
    <ChatProvider config={config}>
      <div style={{ display: 'flex', height: '100vh' }}>
        <HistorySidebar />
        <ChatContainer />
      </div>
    </ChatProvider>
  );
}
```

### Example 3: Custom adapter

For when your backend doesn't match the default adapter's expected endpoints.

```tsx
import React from 'react';
import {
  ChatProvider,
  ChatContainer,
  HistorySidebar,
  ChatAdapter,
  ChatCommonConfig,
} from '@fynix/chat-assistant';

// Implement the adapter to match YOUR backend
const customAdapter: ChatAdapter = {
  async startStream(params) {
    const res = await fetch('https://my-backend.com/api/chat', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${getToken()}`,
      },
      body: JSON.stringify({
        query: params.message,
        conversation_id: params.threadId,
        new_conversation: params.isNewChat,
      }),
    });
    if (!res.ok) throw new Error(`Chat failed: ${res.status}`);
    return res.body!;
    // The stream must follow the streaming protocol
    // (chunks separated by the delimiter, with type fields)
  },

  async cancelStream(threadId) {
    await fetch(`https://my-backend.com/api/chat/${threadId}/stop`, {
      method: 'POST',
      headers: { 'Authorization': `Bearer ${getToken()}` },
    });
  },

  async resumeStream(threadId) {
    const res = await fetch(`https://my-backend.com/api/chat/${threadId}/continue`, {
      headers: { 'Authorization': `Bearer ${getToken()}` },
    });
    return res.body!;
  },

  async fetchThreads({ page, email }) {
    const res = await fetch(
      `https://my-backend.com/api/conversations?page=${page}&user=${email}`,
      { headers: { 'Authorization': `Bearer ${getToken()}` } }
    );
    const data = await res.json();
    return {
      threads: data.conversations.map((c: any) => ({
        id: c.id,
        title: c.name,
        created_at: c.created,
      })),
      pagination: { has_next: data.has_more, page, total: data.total },
    };
  },

  async deleteThread(threadId) {
    await fetch(`https://my-backend.com/api/conversations/${threadId}`, {
      method: 'DELETE',
      headers: { 'Authorization': `Bearer ${getToken()}` },
    });
  },

  async fetchThreadMessages(threadId, signal) {
    const res = await fetch(
      `https://my-backend.com/api/conversations/${threadId}/messages`,
      { headers: { 'Authorization': `Bearer ${getToken()}` }, signal }
    );
    const data = await res.json();
    return {
      success: true,
      data: { messages: data.messages, active_task: data.is_running },
    };
  },
};

const config: ChatCommonConfig = {
  apiBaseUrl: '', // Not used when custom adapter is provided
  getAuthHeaders: () => ({}), // Not used when custom adapter is provided
  enableToolPreview: true,
};

export default function App() {
  return (
    <ChatProvider config={config} adapter={customAdapter}>
      <div style={{ display: 'flex', height: '100vh' }}>
        <HistorySidebar />
        <ChatContainer />
      </div>
    </ChatProvider>
  );
}
```

### Example 4: Custom theme (dark mode)

```html
<div id="chat" style="height: 100vh;"></div>

<script src="path/to/chat.min.js"></script>
<script>
  const darkConfig = {
    apiBaseUrl: 'https://your-api.example.com/v1.0',
    getAuthHeaders: () => ({ 'Authorization': 'Bearer TOKEN' }),
    theme: 'dark',
    themeConfig: {
      primaryColor: '#818cf8',
      primaryHoverColor: '#6366f1',
      backgroundColor: '#0f172a',
      textColor: '#e2e8f0',
      secondaryTextColor: '#94a3b8',
      borderColor: '#334155',
      surfaceColor: '#1e293b',
      surfaceHoverColor: '#334155',
      userBubbleColor: '#1e293b',
      userBubbleBorder: '#334155',
      assistantBorderColor: '#334155',
      codeBackground: '#1e293b',
      successColor: '#4ade80',
      errorColor: '#f87171',
      linkColor: '#93c5fd',
    },
    textConfig: {
      emptyStateTitle: 'Welcome to Dark Mode Chat',
      emptyStateSubtitle: 'Ask anything...',
    },
  };

  FynixChat.render('ChatContainer', '#chat', darkConfig);
</script>
```

### Example 5: Chat-only (no sidebar)

Sometimes you just want the chat container without the history sidebar.

```html
<div id="chat" style="height: 100vh;"></div>

<script src="path/to/chat.min.js"></script>
<script>
  FynixChat.render('ChatContainer', '#chat', {
    apiBaseUrl: 'https://your-api.example.com/v1.0',
    getAuthHeaders: () => ({ 'Authorization': 'Bearer TOKEN' }),
    textConfig: {
      emptyStateTitle: 'Hi there!',
      emptyStateSubtitle: 'Type a message to get started',
    },
  });
</script>
```
