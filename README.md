<p align="center">
  <img src="https://cdn.mangose.app/assets/mangose.png" width="200" alt="Mangose Logo" />
</p>

This is the official Mangose Mods SDK tutorial.

# Tutorial: Build Your First Mangose Mod

This guide is for developers who want to build mods on that runs inside Mangose. It walks you from zero to a working mods with a button, basic state, and host API calls.

## Before you start

- A mods runs inside an isolated `iframe`.
- Your UI is not rendered directly in the mod DOM. Instead, you send a UI tree description built from Mangose components.
- The host renders that UI, enforces safety rules (for example it strips unsafe attributes such as `style`), and performs diffing.
- In practice: build UI with `MangoseMod.*` factories and only send updates for what changed.

## 1. Prerequisites

- You can run a simple HTML page (locally or hosted).
- You can write basic JavaScript.
- You have a place in Mangose where you can paste a mod URL (for example a developer screen or integration configuration, depending on the app version).

## 2. Hello world: a counter

1. Create a file named `index.html`.
2. Paste the code below.
3. Serve it (local server, GitHub Pages, any hosting) and provide the URL in Mangose.

```html
<!DOCTYPE html>
<html>
  <body>
    <script type="module">
      import { MangoseMod } from 'https://cdn.mangose.app/mod-runtime.js'

      // Local state inside the mod
      const state = { count: 0 }

      // UI component factories
      const { Container, Text, Button } = MangoseMod.UI

      // Build the UI tree
      const buildUI = () =>
        Container('counter-root', { direction: 'column', gap: 8 }, [
          Text('counter-title', { value: 'Mod counter' }),
          Text('counter-value', { value: `Current value: ${state.count}` }),
          Button(
            'counter-inc',
            { theme: 'primary', onClick: 'increment' },
            '+1',
          ),
        ])

      // Render only after the host signals readiness
      MangoseMod.onReady(() => {
        MangoseMod.render(buildUI())
      })

      // Handle events coming back from the host
      MangoseMod.onEvent('increment', () => {
        state.count += 1

        // Update only what changed
        MangoseMod.update(
          Text('counter-value', { value: `Current value: ${state.count}` }),
        )
      })
    </script>
  </body>
</html>
```

Key points:

- Any element you want to update should have a stable `mId` (the first argument, for example `counter-value`).
- `render(...)` sends the initial UI tree.
- `update(...)` sends only changed nodes. The host replaces nodes by `mId`.

## 3. The `MangoseMod` runtime

### Installation

If you build your mod as an npm project:

```bash
npm install @mangose/mod-runtime
```

Module import:

```js
import { MangoseMod } from '@mangose/mod-runtime'
```

If you prefer a single HTML file (as above), you can import from the CDN:

```html
<script type="module" src="https://cdn.mangose.dev/mod-runtime.js"></script>
```

### Core APIs

- `MangoseMod.onReady(handler)`
  - Fired when the host is ready. Render your initial UI tree here.
- `MangoseMod.onEvent(event, handler)`
  - Subscribe to a single UI event (use the name without the `event:` prefix).
- `MangoseMod.onEvents(items)`
  - Register multiple events at once. `items` can be an array of tuples `[event, handler]` or an object mapping event names to handlers.
- `MangoseMod.render(node)`
  - Sends the initial UI tree to the host.
- `MangoseMod.update(...nodes)`
  - Sends incremental UI updates. You can send multiple nodes in one call.
- `MangoseMod.<Component>([id], [props], [children])`
  - Factory functions for building UI nodes.

Shorthand examples:

```js
Text({ value: 'No id' })
Text('mod-status', { value: 'Status' })
Container('box', [Text({ value: 'Child' })])
```

## 4. Host API: functions available to mods

`MangoseMod` exposes a set of helpers that call back into the host application. All functions return a `Promise`.

- `MangoseMod.openTask(id)`
  - Opens a task in the main UI. Pass `null` to close the task view.
- `MangoseMod.createTask(taskName, sectionId)`
  - Creates a task in a given section. Returns the operation result (for example the created task object).
- `MangoseMod.sections()`
  - Returns the list of sections for the current collection/view.
- `MangoseMod.getTaskInSection(sectionId)`
  - Returns tasks that belong to the given section.
- `MangoseMod.content()`
  - Returns additional collection content (for example a description).
- `MangoseMod.setContent(value)`
  - Updates the collection content.
- `MangoseMod.getCollection()`
  - Returns metadata for the current collection (name, ids, settings).
- `MangoseMod.getProps()`
  - Returns view/module configuration properties exposed to the mod.

## 5. Events: wiring clicks and form changes

### Clicks

Set the event name as a string (without a prefix) on the component prop, for example:

```js
Button('save', { theme: 'primary', onClick: 'save' }, 'Save')
```

Then subscribe in your mod using `MangoseMod.onEvent('save', handler)` (the runtime will prefix it internally):

```js
MangoseMod.onEvent('save', () => {
  console.log('Saved!')
})
```

### Forms: `value` + `onChange`

Form components are controlled:

- Set `value` to the current value from your state.
- Set `onChange` to the event name string (without a prefix).

Example:

```js
const state = { name: '' }

const ui = () =>
  Container('root', { direction: 'column', gap: 8 }, [
    Input('name', {
      value: state.name,
      placeholder: 'Your name',
      onChange: 'nameChanged',
    }),
    Text('preview', { value: `Typed: ${state.name}` }),
  ])

MangoseMod.onEvent('nameChanged', (payload) => {
  // The host usually sends a primitive value or an object with `value`
  const next = typeof payload === 'string' ? payload : payload?.value
  state.name = next ?? ''

  MangoseMod.update(Text('preview', { value: `Typed: ${state.name}` }))
})
```

### Registering multiple events at once

You can register several handlers in one call:

```js
MangoseMod.onEvents([
  ['ping', () => console.log('ping sent')],
  {
    confirm: handleConfirm,
    cancel: handleCancel,
  },
])
```

## 6. UI updates: simple rules that save time

- Use stable `mId`s for dynamic elements.
- Generate UI from state.
- Update only what changed.
- Sending multiple updates at once is fine:

```js
MangoseMod.update(Text('a', { value: '...' }), Text('b', { value: '...' }))
```

## 7. Limitations and safety

- Do not use inline styles. The host strips `style`.
- Do not rely on direct access to the host DOM. The mod runs in an `iframe`.
- Use component props and supported themes to control layout and appearance.

## 8. Component reference (quick)

Below are the most commonly used building blocks. If you use a less common component, add a short comment in your code explaining why.

### Layout

| Component      | Purpose                  | Key props                                                                                      |
| -------------- | ------------------------ | ---------------------------------------------------------------------------------------------- |
| `UI.Container` | Flex layout              | `direction`, `justify`, `align`, `gap`, `padding`, `margin`, `wrap`, `grow`, `shrink`, `basis` |
| `UI.Fragment`  | Render without a wrapper | none                                                                                           |

Note: numeric `padding` and `margin` are multiplied by 3px on the host side.

### Text and messages

| Component  | Purpose | Key props                                       |
| ---------- | ------- | ----------------------------------------------- |
| `UI.Text`  | Text    | `value`                                         |
| `UI.Alert` | Message | `theme`, `hideOnStart`, `closeButton`, `margin` |

### Actions

| Component        | Purpose      | Key props                                       | Events                    |
| ---------------- | ------------ | ----------------------------------------------- | ------------------------- |
| `UI.Button`      | Button       | `theme`, `size`, `disabled`, `fill`, `selected` | `onClick`                 |
| `UI.ColorPicker` | Color picker | `theme`, `size`                                 | `onColor`, `onBackground` |

### Forms

| Component       | Purpose        | Key props                                                    | Events                                            |
| --------------- | -------------- | ------------------------------------------------------------ | ------------------------------------------------- |
| `UI.Input`      | Text input     | `value`, `placeholder`, `label`, `theme`, `size`, `readonly` | `onChange`, optionally `onKey_enter`, `onKey_esc` |
| `UI.Textarea`   | Multiline text | `value`, `placeholder`, `height`, `readonly`, `theme`        | `onEnter`                                         |
| `UI.Select`     | Dropdown       | `value`, `options`, `disabled`, `theme`, `size`              | `onChange`                                        |
| `UI.Checkbox`   | Checkbox       | `value`, `label`, `theme`, `disabled`                        | `onChange`                                        |
| `UI.DatePicker` | Date/range     | `value`, `range`, `time`, `autoApply`, `theme`, `size`       | `onChange`                                        |
| `UI.Files`      | File picker    | `selectMode`, `onlyImage`, `selectAfterUpload`               | `onChange`                                        |

### Modals and media

| Component   | Purpose         | Key props                                                                      | Events                   |
| ----------- | --------------- | ------------------------------------------------------------------------------ | ------------------------ |
| `UI.Modal`  | Modal dialog    | `title`, `confirmText`, `cancelText`, `showOnStart`, `width`, `disableConfirm` | `onOnDone`, `onOnCancel` |
| `UI.Iframe` | Embedded iframe | `src`, `title`, `width`, `height`                                              | none                     |

## 9. Debugging

- Use `console.log(...)` inside your mod. Check logs in the mod iframe devtools.
- Temporarily render state in the UI:

```js
Text('debug', { value: JSON.stringify(state) })
```

If the UI is not reacting:

- ensure the updated element has the same `mId` as in the initial render,
- ensure the event name in props (for example `'save'`) matches the name used in `onEvent('save', ...)`,
- ensure you call `update(...)` after changing state.

## 10. Common pitfalls

- Missing `mId` for elements you want to update.
- Changing `mId` between renders (the host cannot match nodes).
- Trying to style via `style` (it will be stripped).
- Typos in event names (`increment` vs `incement`).
- Do not render your entire mod exclusively inside the `UI.Iframe` component. `UI.Iframe` is intended only for auxiliary flows (for example login, authorization, or informational screens). Mangose will not accept store mods whose full functionality lives solely inside an embedded iframe.

## 11. Publish a mod to the Mangose Store

All store mods are hosted in a public GitHub repository. Adding a new mod or updating an existing one is done via Pull Request.

### Repository

Repo: `Mangose/mods`.

### Add a new mod

1. Fork the repository and create a new branch.
2. Add your mod folder under:

- `/mods/<your-mod>/`

Important:

- The entry file must be named **`index.html`**.
- Keep all mod files inside the mod folder (HTML, JS, CSS, assets).
- Avoid linking to third party assets without a clear reason. Prefer assets committed to the repo.

3. Add an entry to:

- `/mods/store.json`

The entry should point to your mod folder and include basic metadata (name, description, author, version, etc., per the file schema).

4. Open a Pull Request to `Mangose/mods` with a short description:

- what the mod does,
- how to test it,
- what changed (if this is an update).

### Update an existing mod

Updates follow the same flow as adding a new mod:

1. Update files in `/mods/<your-mod>/`.
2. If version or metadata changes, update the relevant entry in `/mods/store.json`.
3. Open a Pull Request describing the changes.

### File structure and asset size

We keep the store fast and reliable for everyone, so please follow these guidelines:

- Use a simple, predictable structure (for example `index.html`, `assets/`, `styles/`, `scripts/`).
- Compress images and avoid shipping heavy exports.
  - Prefer JPEG/WebP/AVIF when appropriate.
  - Keep resolution to what you actually need.
  - Avoid large PNG files unless transparency is required.
- If you add video or other large files, explain why in the PR and consider a “light” alternative.
- Ensure the mod works without additional host side dependencies. Everything required should live in the mod folder.

These rules help us maintain high quality and fast loading times across the store.

## 12. Next steps

- Test your mod locally and inside Mangose.
- Submit it to the Mangose Store if you want to share it with others.
- Join the Mangose community to exchange ideas and best practices.
