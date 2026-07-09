# @brainpiper/composekit

ComposeKit is a page renderer SDK by BrainPiper. It renders layouts designed in the BrainPiper Studio portal, and provides individual UI components that can be used standalone.

ComposeKit uses **Tailwind CSS v4.3** internally and ships its own compiled stylesheet. If your app also uses Tailwind and customizes core utility scales (spacing, typography, colors), it may affect ComposeKit component styles.

---

## Table of Contents

- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Setup — ComposeKitClient](#setup--composekitclient)
- [Render Mode — CDN vs Build-time](#render-mode--cdn-vs-build-time)
- [Creating a Widget in BrainPiper Studio](#creating-a-widget-in-brainpiper-studio)
- [Rendering a Widget — ComposeKitRenderer](#rendering-a-widget--composekitrenderer)
  - [Data binding](#data-binding)
  - [Layout types](#layout-types)
- [Theming — setTheme](#theming--settheme)
- [Using Individual Components](#using-individual-components)
- [Server-side AI Generation — ComposeKitServer](#server-side-ai-generation--composekitserver)
- [TypeScript](#typescript)
- [Troubleshooting](#troubleshooting)
- [Changelog](#changelog)

---

## Prerequisites

- **React 18 or 19** — ComposeKit is a React library and expects React to be provided by your application. It does not bundle React internally.
- **Client ID** from the BrainPiper portal — required for fetching layouts and plugin metadata.

To get a Client ID:

1. Open [BrainPiper Studio](https://studio.brainpiper.com/portal/builder/#/credentials)
2. Create an application and copy the generated **Client ID**

---

## Installation

```bash
npm install @brainpiper/composekit lucide-react
```

`lucide-react` is an optional peer dependency. Install it only if you want to render custom icons inside `ComposeKitButton`. Only Lucide icons are supported.

Import the stylesheet once at your app root:

```ts
import '@brainpiper/composekit/dist/style.css';
```

---

## Setup — ComposeKitClient

Call `ComposeKitClient.init()` once at the entry point of your application, before rendering any ComposeKit components.

```ts
import { ComposeKitClient } from '@brainpiper/composekit';

ComposeKitClient.init({
    clientId: 'YOUR_CLIENT_ID',
});
```

### Config options

| Option | Type | Default | Description |
|---|---|---|---|
| `clientId` | `string` | required | Your application Client ID from BrainPiper Studio |
| `disableCdn` | `boolean` | `false` | See [Render Mode](#render-mode) |
| `telemetry` | `boolean` | `true` | Enables anonymous usage telemetry |
| `env` | `'dev' \| 'prod'` | `'dev'` | Environment — errors are shown in UI in `dev`, silently suppressed in `prod` |

Once initialized, any `ComposeKitRenderer` on the page will pick up these settings automatically.

---

## Render Mode

ComposeKit supports three render modes depending on how you use it.

---

### Mode 1 — CDN (default, `disableCdn: false`)

```ts
ComposeKitClient.init({
    clientId: 'YOUR_CLIENT_ID',
    // disableCdn defaults to false
});
```

Plugin bundles are downloaded from the BrainPiper CDN and injected into the browser at runtime, when the layout is first rendered. No plugin code is included in your app bundle at build time.

**Tradeoffs:**
- Your app bundle stays small
- Plugin bug fixes ship automatically — no redeploy needed
- Requires an internet connection at runtime

**Best for:** Most production apps.

> **Security note:** Plugins are served over HTTPS from BrainPiper's CDN and contain the same compiled code as the build-time bundled version. As with any externally loaded script, you are trusting BrainPiper's CDN infrastructure. If your security policy requires full control over executed code, use build-time mode (`disableCdn: true`) instead.

---

### Mode 2 — Build-time with `ComposeKitRenderer` (`disableCdn: true`)

```ts
ComposeKitClient.init({
    clientId: 'YOUR_CLIENT_ID',
    disableCdn: true,
});
```

Use this when you are using `ComposeKitRenderer` to render layouts at runtime (via `widgetId` or a `layout` array), but cannot rely on the CDN.

Because `ComposeKitRenderer` resolves which components to render at runtime from the layout data, it cannot know at build time which plugins will be needed. As a result, **all 22 plugins are bundled into your app** — tree shaking cannot help here.

Internally, all plugin imports use `webpackMode: "eager"`, which compiles them inline into the main bundle with no async chunks or network requests at runtime.

**Tradeoffs:**
- No CDN or internet required at runtime — fully offline capable
- All plugins are always bundled regardless of which ones the layout uses
- App bundle is significantly larger
- Plugin updates require a package upgrade and a full rebuild

**Best for:** Offline or intranet deployments where CDN access is not available.

---

### Mode 3 — Individual components only (no `ComposeKitRenderer`)

If you are using individual components directly (e.g. `ComposeKitInput`, `ComposeKitButton`) and not using `ComposeKitRenderer` at all, you do not need to set `disableCdn`. Simply import the components you need:

```tsx
import { ComposeKitInput } from '@brainpiper/composekit/input';
import { ComposeKitButton } from '@brainpiper/composekit/button';
```

Since each component is a separate subpath export, your bundler's tree shaking will include only the components you actually import. The other 19+ plugins will not be in your bundle.

**Tradeoffs:**
- Smallest possible bundle — only imported components are included
- No runtime CDN requests
- You manage layout and composition yourself in code — no visual page builder

**Best for:** Using ComposeKit as a standalone component library without the page renderer.

---

## Creating a Widget in BrainPiper Studio

Before you can render a widget with `ComposeKitRenderer`, you need to build and publish it in [BrainPiper Studio](https://studio.brainpiper.com/portal/builder/#/chat). There are two ways to create a widget:

### Option 1 — Chat (AI-generated)

Open the **Chat** tab in Studio and describe what you want in plain language:

> *"Create a user registration form with name, email, password, and a submit button"*

The AI generates a complete widget layout from your description. You can then refine it by sending follow-up messages or switching to the Editor tab to adjust individual components.

### Option 2 — Editor (drag-and-drop)

Open the **Editor** tab to build your widget visually:

1. Drag components from the left panel onto the canvas
2. Click any component to configure its props, data bindings, and styles in the right panel
3. Arrange components into rows and columns to define your layout

### Publishing and getting the Widget ID

Once your widget is ready:

1. Click **Publish** in the top-right corner of Studio
2. The widget ID is the UUID at the end of the browser URL, for example `https://studio.brainpiper.com/portal/builder/#/chat/4c6e51bc-59c1-4d54-afff-20bb54311dab` — copy the UUID part
3. Pass it to `ComposeKitRenderer` as the `widgetId` prop:

```tsx
<ComposeKitRenderer
    widgetId="4c6e51bc-59c1-4d54-afff-20bb54311dab"
    data={{ userId: 'u_001' }}
    onUpdate={(dataPath, value) => console.log(dataPath, value)}
/>
```

The renderer fetches the published layout at runtime and renders it using the same components as your Studio canvas.

> **Tip:** Every time you publish an updated version of the widget in Studio, `ComposeKitRenderer` picks up the changes automatically on the next page load — no code change or redeploy required.

---

## Rendering a Widget — ComposeKitRenderer

`ComposeKitRenderer` renders a widget built in BrainPiper Studio. Each widget follows this structure:

```
widget (widgetId)
  └── row
        └── column
              └── plugin  (button, input, or a nested widget)
```

Provide the widget either via a `widgetId` (fetched from the portal at runtime) or a `layout` array (when you already have the parsed layout — e.g. generated by `ComposeKitServer`).

### Using a widget ID

Pass the widget ID from BrainPiper Studio. The renderer fetches the widget's layout from the portal at runtime.

```tsx
import { ComposeKitRenderer } from '@brainpiper/composekit';

function MyPage() {
    return (
        <ComposeKitRenderer
            widgetId="your-widget-id"
            widgetDescription="User registration form"
            data={{ userId: 'u_001', name: 'Jane' }}
            onUpdate={(dataPath, value) => {
                console.log('Field updated:', dataPath, value);
            }}
            onError={(pluginName, message) => {
                console.error(`Plugin error in ${pluginName}:`, message);
            }}
        />
    );
}
```

### Using a layout array

Pass the parsed layout array directly — use this when you have fetched or generated the layout yourself (e.g. via `ComposeKitServer.createResponse()`).

```tsx
<ComposeKitRenderer
    layout={parsedLayoutArray}
    data={{ userId: 'u_001', name: 'Jane' }}
    onUpdate={(dataPath, value) => console.log('Field updated:', dataPath, value)}
/>
```

### Props

| Prop | Type | Description |
|---|---|---|
| `widgetId` | `string` | Widget ID from BrainPiper Studio — renderer fetches the layout at runtime |
| `layout` | `ILayout` | Parsed layout array — use when you already have the layout object |
| `data` | `object` | Data to inject into the rendered widget — see [Data binding](#data-binding) |
| `onUpdate` | `(dataPath: string, value: any) => void` | Called when any field value changes — see [Data binding](#data-binding) |
| `onError` | `(pluginName: string, message: string) => void` | Called when a plugin fails to render |
| `widgetDescription` | `string` | Optional description of what this widget does — for documentation purposes only, not rendered |
| `classNames.container` | `string` | CSS class for the outermost container |
| `classNames.layout` | `string` | CSS class for the layout wrapper |
| `classNames.row` | `string` | CSS class applied to every row |
| `classNames.column` | `string` | CSS class applied to every column |
| `classNames.widget` | `string` | CSS class applied to every widget |
| `styles.container` | `CSSProperties` | Inline styles for the outermost container |
| `styles.layout` | `CSSProperties` | Inline styles for the layout wrapper |
| `styles.row` | `CSSProperties` | Inline styles applied to every row |
| `styles.column` | `CSSProperties` | Inline styles applied to every column |
| `styles.widget` | `CSSProperties` | Inline styles applied to every widget |

> `clientId`, `disableCdn`, `telemetry`, and `env` are configured once via `ComposeKitClient.init()` and apply globally to all renderers on the page.

> **Beta limitations — UI orchestration (workflows, localization, design system tokens, API endpoints) not yet supported:**
> The following Studio-configured behaviours are not executed when a widget is rendered via `ComposeKitRenderer` in this beta version. The widget renders correctly and data binding works as normal.
>
> - **Workflows** — plugin event handlers (e.g. button click triggering an API call or navigation) are not executed
> - **Localization** — locale-aware field values and translations are not applied
> - **Design system tokens** — tokens configured in the Studio design system are not applied (use [`ComposeKitClient.setTheme()`](#theming--settheme) to theme components instead)
> - **API endpoints** — data source bindings configured in Studio are not fetched
>
> Full UI orchestration support is planned for a future release.

### Data binding

Each field inside a widget is configured in BrainPiper Studio with a **Data Path** — a dot-notation string like `"user.name"` or `"form.email"`. ComposeKit uses this path to:

1. **Read** the initial value from `data` — e.g. `data.user.name` pre-fills a name input.
2. **Write** back via `onUpdate` when the field changes — e.g. `onUpdate("user.name", "Jane")`.

The typical pattern is to keep `data` in React state and merge updates as they arrive:

```tsx
const [data, setDataModel] = useState({ user: { name: '', email: '' } });

<ComposeKitRenderer
    widgetId="your-widget-id"
    data={data}
    onUpdate={(dataPath, value) => {
        setDataModel(prev => ({ ...prev, [dataPath]: value }));
    }}
/>
```

`ComposeKitRenderer` never mutates `data` — all updates flow out through `onUpdate` and it is up to your code to decide whether and how to apply them.

### Layout types

The `layout` prop is typed via exported interfaces. Import them if you are constructing or validating a layout object in TypeScript:

```ts
import type { ILayout, ILayoutRow, ILayoutColumn, ILayoutWidget } from '@brainpiper/composekit';
```

| Type | Description |
|---|---|
| `ILayout` | `ILayoutRow[]` — the top-level layout array |
| `ILayoutRow` | A row containing one or more columns |
| `ILayoutColumn` | A column containing widgets and optional nested rows |
| `ILayoutWidget` | A single plugin instance inside a column |

#### `ILayoutRow`

| Property | Type | Description |
|---|---|---|
| `columns` | `ILayoutColumn[]` | Columns in this row |

#### `ILayoutColumn`

| Property | Type | Description |
|---|---|---|
| `title` | `string` | Column title |
| `description` | `string` | Column description |
| `widgets` | `ILayoutWidget[]` | Plugin instances in this column |
| `rows` | `ILayoutRow[]` | Nested rows inside this column — use when widgets within the section need to sit side by side |
| `pluginProps` | `object` | Internal rendering configuration passed to the section wrapper — set by BrainPiper Studio, not required when constructing layouts manually |

#### `ILayoutWidget`

| Property | Type | Description |
|---|---|---|
| `componentId` | `string` | Plugin component identifier (e.g. `@bplcp/input`) |
| `props` | `Record<string, any>` | Runtime props passed to the plugin |
| `pluginProps` | `object` | Internal plugin configuration — set by BrainPiper Studio, not required when constructing layouts manually |
| `visible` | `boolean` | Whether the widget is visible |

---

## Theming — setTheme

Override the default ComposeKit color tokens to match your brand. Call `setTheme` after the renderer has mounted.

```ts
import { ComposeKitClient } from '@brainpiper/composekit';

ComposeKitClient.setTheme({
    primary: '#6366f1',
    primaryForeground: '#ffffff',
    background: '#ffffff',
    foreground: '#111827',
    border: '#e5e7eb',
    radius: '0.5rem',
    fontFamily: 'Inter, sans-serif',
});
```

### Available tokens

| Token | CSS variable | Description |
|---|---|---|
| `primary` | `--primary` | Primary action color |
| `primaryForeground` | `--primary-foreground` | Text on primary background |
| `secondary` | `--secondary` | Secondary action color |
| `secondaryForeground` | `--secondary-foreground` | Text on secondary background |
| `background` | `--background` | Page background |
| `foreground` | `--foreground` | Default text color |
| `card` | `--card` | Card background |
| `cardForeground` | `--card-foreground` | Card text color |
| `muted` | `--muted` | Muted background |
| `mutedForeground` | `--muted-foreground` | Muted text |
| `accent` | `--accent` | Accent background |
| `accentForeground` | `--accent-foreground` | Accent text |
| `destructive` | `--destructive` | Destructive/error color |
| `border` | `--border` | Border color |
| `input` | `--input` | Input border color |
| `ring` | `--ring` | Focus ring color |
| `popover` | `--popover` | Popover background |
| `popoverForeground` | `--popover-foreground` | Popover text |
| `radius` | `--radius` | Border radius (e.g. `0.5rem`) |
| `fontFamily` | `--default-font-family` | Base font family |
| `spacing` | `--spacing` | Base spacing unit |

---

## Using Individual Components

Each ComposeKit component is available as a standalone import. Components can be used independently outside of a rendered page.

ComposeKit ships **22 components** in this beta release. All components are built on top of **Radix UI primitives** and **shadcn/ui**, styled with Tailwind CSS v4. The Chart component is powered by **Recharts**. Additional components will be added in future releases.

| Component | Import path | Category | Since |
|---|---|---|---|
| `ComposeKitButton` | `@brainpiper/composekit/button` | Actions | 0.0.1-beta.1 |
| `ComposeKitButtonGroup` | `@brainpiper/composekit/button-group` | Actions | 0.0.1-beta.1 |
| `ComposeKitOverlay` | `@brainpiper/composekit/overlay` | Actions | 0.0.1-beta.1 |
| `ComposeKitInput` | `@brainpiper/composekit/input` | Form | 0.0.1-beta.1 |
| `ComposeKitTextarea` | `@brainpiper/composekit/textarea` | Form | 0.0.1-beta.1 |
| `ComposeKitSelect` | `@brainpiper/composekit/select` | Form | 0.0.1-beta.1 |
| `ComposeKitRadioGroup` | `@brainpiper/composekit/radio-group` | Form | 0.0.1-beta.1 |
| `ComposeKitToggle` | `@brainpiper/composekit/toggle` | Form | 0.0.1-beta.1 |
| `ComposeKitHeading` | `@brainpiper/composekit/heading` | Display | 0.0.1-beta.1 |
| `ComposeKitParagraph` | `@brainpiper/composekit/paragraph` | Display | 0.0.1-beta.1 |
| `ComposeKitBadge` | `@brainpiper/composekit/badge` | Display | 0.0.1-beta.1 |
| `ComposeKitAlert` | `@brainpiper/composekit/alert` | Display | 0.0.1-beta.1 |
| `ComposeKitAvatar` | `@brainpiper/composekit/avatar` | Display | 0.0.1-beta.1 |
| `ComposeKitImage` | `@brainpiper/composekit/image` | Display | 0.0.1-beta.1 |
| `ComposeKitSeparator` | `@brainpiper/composekit/separator` | Display | 0.0.1-beta.1 |
| `ComposeKitCard` | `@brainpiper/composekit/card` | Structure | 0.0.1-beta.1 |
| `ComposeKitSection` | `@brainpiper/composekit/section` | Structure | 0.0.1-beta.1 |
| `ComposeKitAccordion` | `@brainpiper/composekit/accordion` | Structure | 0.0.1-beta.1 |
| `ComposeKitList` | `@brainpiper/composekit/list` | Structure | 0.0.1-beta.1 |
| `ComposeKitCarousel` | `@brainpiper/composekit/carousel` | Structure | 0.0.1-beta.1 |
| `ComposeKitGrid` | `@brainpiper/composekit/grid` | Structure | 0.0.1-beta.1 |
| `ComposeKitChart` | `@brainpiper/composekit/chart` | Charts | 0.0.1-beta.2 |

### Button

```tsx
import { ComposeKitButton } from '@brainpiper/composekit/button';

<ComposeKitButton
    label="Submit"
    variant="default"
    onClick={() => console.log('clicked')}
/>
```

### Input

```tsx
import { ComposeKitInput } from '@brainpiper/composekit/input';

<ComposeKitInput
    label="Email"
    type="email"
    placeholder="you@example.com"
    onChange={(value) => console.log(value)}
/>
```

### Textarea

```tsx
import { ComposeKitTextarea } from '@brainpiper/composekit/textarea';

<ComposeKitTextarea
    label="Notes"
    rows={4}
    onChange={(value) => console.log(value)}
/>
```

### Select

```tsx
import { ComposeKitSelect } from '@brainpiper/composekit/select';

<ComposeKitSelect
    label="Country"
    placeholder="Select a country"
    options={[
        { value: 'us', label: 'United States' },
        { value: 'uk', label: 'United Kingdom' },
    ]}
    dataKey="value"
    displayKey="label"
    onChange={(option) => console.log(option)}
/>
```

### Radio Group

```tsx
import { ComposeKitRadioGroup } from '@brainpiper/composekit/radio-group';

<ComposeKitRadioGroup
    label="Plan"
    options={[
        { value: 'free', label: 'Free' },
        { value: 'pro', label: 'Pro' },
    ]}
    dataKey="value"
    displayKey="label"
    onChange={(option) => console.log(option)}
/>
```

### Toggle

```tsx
import { ComposeKitToggle } from '@brainpiper/composekit/toggle';

<ComposeKitToggle
    label="Enable notifications"
    onChange={(checked) => console.log(checked)}
/>
```

### Grid

```tsx
import { ComposeKitGrid } from '@brainpiper/composekit/grid';

<ComposeKitGrid
    config={{
        columns: [
            { field: 'name',  label: 'Name',  type: 'text' },
            { field: 'email', label: 'Email', type: 'text' },
            { field: 'role',  label: 'Role',  type: 'text', editable: false },
        ],
        data: [
            { name: 'Alice', email: 'alice@example.com', role: 'Engineer' },
            { name: 'Bob',   email: 'bob@example.com',   role: 'Designer' },
        ],
        enablePagination: true,
        recordsPerPage: 10,
        enableSorting: true,
        enableFiltering: true,
    }}
/>
```

> Note: `ComposeKitGrid` takes a single `config` object prop rather than flat props.

### Chart

```tsx
import { ComposeKitChart } from '@brainpiper/composekit/chart';

// Line chart
<ComposeKitChart
    type="line"
    data={[
        { month: 'Jan', revenue: 4200, expenses: 2800 },
        { month: 'Feb', revenue: 5800, expenses: 3100 },
        { month: 'Mar', revenue: 7100, expenses: 4200 },
    ]}
    dataKeys={['revenue', 'expenses']}
    xAxisKey="month"
    height={300}
    showDots={true}
    valueFormatter={(v) => '$' + v.toLocaleString()}
    card={true}
    cardHeader="Monthly Revenue"
/>

// Donut chart
<ComposeKitChart
    type="pie"
    data={[
        { source: 'Organic', value: 38 },
        { source: 'Direct',  value: 27 },
        { source: 'Paid',    value: 35 },
    ]}
    dataKeys={['value']}
    xAxisKey="source"
    innerRadius={60}
    outerRadius={100}
    height={280}
/>
```

Supported types: `"line"` · `"bar"` · `"area"` · `"pie"` · `"composed"` · `"radar"`

Key props:

| Prop | Type | Default | Description |
|---|---|---|---|
| `type` | `ChartType` | `"line"` | Chart type |
| `data` | `object[]` | `[]` | Data array |
| `dataKeys` | `string[]` | `[]` | Keys to plot as series |
| `xAxisKey` | `string` | `"name"` | Key for the X/category axis |
| `height` | `number` | `300` | Chart height in px |
| `valueFormatter` | `(v: number) => string` | — | Format Y-axis ticks and tooltip values |
| `stacked` | `boolean` | `false` | Stack bar or area series |
| `horizontal` | `boolean` | `false` | Horizontal bar chart |
| `gradientFill` | `boolean` | `false` | SVG gradient fill for area charts |
| `innerRadius` | `number` | `0` | Donut hole radius — set > 0 for donut |
| `referenceLines` | `object[]` | `[]` | Horizontal reference lines with optional label and color |
| `brush` | `boolean` | `false` | Zoom/pan brush on the X-axis |
| `loading` | `boolean` | `false` | Show skeleton placeholder |
| `card` | `boolean` | `false` | Wrap in a card with optional `cardHeader` and `cardDescription` |

See the [docs](https://composekit.brainpiper.com/docs) for a full interactive reference with all chart types and feature examples.

---

## Server-side AI Generation — ComposeKitServer

`ComposeKitServer` lets you generate page layouts from a natural language prompt using BrainPiper's AI. It is intended for **server-side use only** — it requires an API key that must never be exposed in a browser.

Import from the `/server` subpath:

```ts
import { ComposeKitServer } from '@brainpiper/composekit/server';
```

### Getting an API Key

An API key is required in addition to your Client ID. Generate one from [BrainPiper Studio](https://studio.brainpiper.com/portal/builder/#/compose-kit).

### Setup

Call `ComposeKitServer.init()` once when your server starts:

```ts
import { ComposeKitServer } from '@brainpiper/composekit/server';

ComposeKitServer.init({
    clientId: 'YOUR_CLIENT_ID',
    apiKey: 'YOUR_API_KEY',
});
```

### Generate a layout (non-streaming)

Returns a complete layout response after the AI finishes generating. Send the `layout` array from the response to your client (via an API route, server props, etc.) and pass it directly to `ComposeKitRenderer`.

The response shape is:

```ts
{
    success: boolean;
    data: {
        layout:  string; // JSON-encoded ILayout — parse before use
        summary: string; // human-readable description of the generated layout
    }
}
```

```ts
// server-side (e.g. Next.js API route)
const result = await ComposeKitServer.createResponse({
    prompt: 'Create a user registration form with name, email, and password fields',
});

if (result.success) {
    return res.json({
        layout:  JSON.parse(result.data.layout),
        summary: result.data.summary,
    });
}
```

```tsx
// client-side
import type { ILayout } from '@brainpiper/composekit';

const { layout, summary } = await fetch('/api/generate').then(r => r.json());

<ComposeKitRenderer layout={layout} />
```

> **Security note:** Never call `ComposeKitServer.init()` or expose your `apiKey` in client-side code. Always use environment variables and keep the API key on the server.

---

## TypeScript

`@brainpiper/composekit` ships its own type declarations — no `@types/*` package is needed. All props, layout interfaces, and the client/server APIs are fully typed.

---

## Troubleshooting

**Widget shows "Loading widget..." and never renders**
- Check that `ComposeKitClient.init()` was called before the component mounted.
- Verify your Client ID is correct — open the browser Network tab and confirm the `/plugins.php` and `/get-widget-details` requests return `success: true`.

**No widget appears at all (blank area)**
- If `widgetId` is set but `clientId` is missing or invalid, the renderer returns nothing. Check the browser console for `"Please provide valid brainpiper clientId"`.

**Components look unstyled**
- Ensure `import '@brainpiper/composekit/dist/style.css'` is present at your app root.

**Studio-configured behaviours don't execute (workflows, localization, API endpoints, design system tokens)**
- These are known beta limitations — see the [note under ComposeKitRenderer](#rendering-a-widget--composekitrenderer) for the full list. Full UI orchestration support is planned for a future release.

**Tailwind CSS conflicts**
- ComposeKit uses Tailwind v4.3 internally with its own compiled stylesheet. If your app customizes core Tailwind scales (spacing, colors, typography) at the root level, those values may bleed into ComposeKit components. Scope your Tailwind customizations to your own layer or use CSS `@layer` to prevent conflicts.

---

## Changelog

### 0.0.1-beta.2

- **New** — `ComposeKitChart` — line, bar, area, pie/donut, composed, and radar chart types powered by [Recharts](https://recharts.org). Supports stacked series, horizontal bars, gradient fills, reference lines, brush zoom, loading/empty states, and full axis/tooltip formatting. Import from `@brainpiper/composekit/chart`.

### 0.0.1-beta.1

- Initial beta release with 21 components: Button, ButtonGroup, Overlay, Input, Textarea, Select, RadioGroup, Toggle, Heading, Paragraph, Badge, Alert, Avatar, Image, Separator, Card, Section, Accordion, List, Carousel, Grid.
- `ComposeKitRenderer` with CDN (default) and build-time (`disableCdn: true`) render modes.
- `ComposeKitClient` with `init()`, `setTheme()`, and `setLocale()`.
- `ComposeKitServer` for server-side AI layout generation.
