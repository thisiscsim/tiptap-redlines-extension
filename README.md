# tiptap-redlines-extension

Track changes (redlines) extension for **TipTap v3**. Adds Microsoft Word-style revision tracking to any TipTap editor — insertions are highlighted with color and underline, deletions are preserved with strikethrough.

Built for TipTap v3. Based on [chenyuncai/tiptap-track-change-extension](https://github.com/chenyuncai/tiptap-track-change-extension), rewritten to use TipTap v3's `addStorage()` API instead of the v2 extension-lookup pattern.

## Features

- **Insertion tracking** — New text is wrapped in `<insert>` tags
- **Deletion tracking** — Deleted text is preserved and wrapped in `<delete>` tags (strikethrough)
- **Accept/Reject** — Accept or reject individual changes or all changes at once
- **User attribution** — Attach user ID and nickname to each change
- **CJK input support** — Handles IME composition for Chinese/Japanese/Korean input
- **Y.js compatible** — Ignores changes from collaborative sync

## Installation

```bash
npm install tiptap-redlines-extension
```

Peer dependencies:

```bash
npm install @tiptap/core @tiptap/pm
```

## Quick Start

```typescript
import { Editor } from '@tiptap/core'
import StarterKit from '@tiptap/starter-kit'
import { RedlinesExtension } from 'tiptap-redlines-extension'

const editor = new Editor({
  extensions: [
    StarterKit,
    RedlinesExtension.configure({
      enabled: false, // start with tracking disabled
      onStatusChange: (enabled) => {
        console.log('Track changes:', enabled ? 'ON' : 'OFF')
      },
    }),
  ],
  content: '<p>Hello World</p>',
})
```

## Usage

### Enable/Disable Track Changes

```typescript
// Enable
editor.commands.setTrackChangeStatus(true)

// Disable
editor.commands.setTrackChangeStatus(false)

// Toggle
editor.commands.toggleTrackChangeStatus()
```

### Accept and Reject Changes

```typescript
// Accept the change at cursor position or within selection
editor.commands.acceptChange()

// Reject the change at cursor position or within selection
editor.commands.rejectChange()

// Accept all changes in the document
editor.commands.acceptAllChanges()

// Reject all changes in the document
editor.commands.rejectAllChanges()
```

### User Attribution

```typescript
editor.commands.updateOpUserOption('user-123', 'Jane Smith')
```

Each tracked change stores `data-op-user-id`, `data-op-user-nickname`, and `data-op-date` attributes on the mark element.

## Styling

The extension renders insertions as `<insert>` elements and deletions as `<delete>` elements. Add CSS to style them:

```css
/* Insertions — blue underline */
.ProseMirror insert {
  color: #2563EB;
  text-decoration: underline;
  text-decoration-color: #2563EB;
  text-underline-offset: 2px;
}

/* Deletions — red strikethrough */
.ProseMirror delete {
  color: #E11D48;
  text-decoration: line-through;
  text-decoration-color: #E11D48;
  opacity: 0.7;
}
```

Or use any colors that match your design system.

## React Example

```tsx
import { useEditor, EditorContent } from '@tiptap/react'
import StarterKit from '@tiptap/starter-kit'
import { RedlinesExtension } from 'tiptap-redlines-extension'

function MyEditor() {
  const [isTracking, setIsTracking] = useState(false)

  const editor = useEditor({
    extensions: [
      StarterKit,
      RedlinesExtension.configure({
        enabled: false,
        onStatusChange: setIsTracking,
      }),
    ],
    content: '<p>Start editing...</p>',
  })

  return (
    <div>
      <div className="toolbar">
        <button onClick={() => editor?.commands.toggleTrackChangeStatus()}>
          {isTracking ? 'Suggesting' : 'Editing'}
        </button>
        <button onClick={() => editor?.commands.acceptAllChanges()}>
          Accept All
        </button>
        <button onClick={() => editor?.commands.rejectAllChanges()}>
          Reject All
        </button>
      </div>
      <EditorContent editor={editor} />
    </div>
  )
}
```

## Vue 3 Example

```vue
<script setup>
import { useEditor, EditorContent } from '@tiptap/vue-3'
import StarterKit from '@tiptap/starter-kit'
import { RedlinesExtension } from 'tiptap-redlines-extension'

const isTracking = ref(false)

const editor = useEditor({
  extensions: [
    StarterKit,
    RedlinesExtension.configure({
      enabled: false,
      onStatusChange: (status) => { isTracking.value = status },
    }),
  ],
  content: '<p>Start editing...</p>',
})
</script>

<template>
  <div>
    <button @click="editor?.commands.toggleTrackChangeStatus()">
      {{ isTracking ? 'Suggesting' : 'Editing' }}
    </button>
    <EditorContent :editor="editor" />
  </div>
</template>
```

## API Reference

### Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `enabled` | `boolean` | `false` | Whether track changes is enabled on init |
| `onStatusChange` | `(enabled: boolean) => void` | `undefined` | Callback when status changes |
| `dataOpUserId` | `string` | `''` | User ID for change attribution |
| `dataOpUserNickname` | `string` | `''` | User nickname for change attribution |

### Commands

| Command | Description |
|---------|-------------|
| `setTrackChangeStatus(enabled)` | Enable or disable tracking |
| `getTrackChangeStatus()` | Get current tracking status |
| `toggleTrackChangeStatus()` | Toggle tracking on/off |
| `acceptChange()` | Accept change at cursor/selection |
| `acceptAllChanges()` | Accept all changes |
| `rejectChange()` | Reject change at cursor/selection |
| `rejectAllChanges()` | Reject all changes |
| `updateOpUserOption(id, name)` | Set user info for changes |

### Exports

| Export | Description |
|--------|-------------|
| `RedlinesExtension` | Main extension (default export) |
| `InsertionMark` | TipTap Mark for insertions |
| `DeletionMark` | TipTap Mark for deletions |
| `MARK_INSERTION` | Mark name constant (`'insertion'`) |
| `MARK_DELETION` | Mark name constant (`'deletion'`) |

### HTML Output

Insertions render as:
```html
<insert data-op-user-id="..." data-op-user-nickname="..." data-op-date="...">new text</insert>
```

Deletions render as:
```html
<delete data-op-user-id="..." data-op-user-nickname="..." data-op-date="...">removed text</delete>
```

## How It Works

When tracking is enabled:

1. **Typing new text** — The `onTransaction` hook detects new content via ProseMirror `ReplaceStep` and applies the `insertion` mark
2. **Deleting text** — Instead of removing content, the extension re-adds it with the `deletion` mark (red strikethrough)
3. **Deleting tracked insertions** — If you delete text that was already marked as an insertion, it's removed for real (since it was a pending suggestion)
4. **Accepting a change** — Insertions: the mark is removed (text becomes normal). Deletions: the content is removed for real.
5. **Rejecting a change** — Insertions: the content is removed. Deletions: the mark is removed (text is restored).

## Compatibility

- TipTap v3 (`@tiptap/core ^3.0.0`)
- Works with `@tiptap/react`, `@tiptap/vue-3`, and vanilla JS
- Compatible with Y.js collaborative editing (ignores sync changes)

## Credits

Based on [chenyuncai/tiptap-track-change-extension](https://github.com/chenyuncai/tiptap-track-change-extension) (MIT License). Rewritten for TipTap v3 compatibility using `addStorage()` instead of the v2 `getSelfExt()` pattern.

## License

MIT
