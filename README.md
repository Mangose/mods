<p align="center">
  <img src="https://cdn.mangose.app/assets/mangose.png" width="200" alt="Mangose Logo" />
</p>

This is the official Mangose Mods SDK tutorial.

# Tutorial: Build Your First Mangose Mod

This guide is for developers who want to build mods that run inside Mangose. It takes you from zero to a working mod with a button, basic state, and host API calls.

## Before you start

- A mod runs inside an isolated `iframe`.
- Your UI is not rendered directly in the mod’s DOM. Instead, you send a UI tree description built from Mangose components.
- In practice: build UI with `MangoseMod.*` factories and only send updates for what changed.

## 1. Prerequisites

- You can run a simple HTML page
- You can write basic JavaScript

> To add a new mod in Mangose, create a new view and choose **Design editor**. Then pick one of the available mods from the list.

> If you’re building your own mod, select **Empty Mod**. Next, open the mod in the new window and click the **gear icon** to open settings. From there, you can either:
> Enter your GitHub repository, or paste a direct URL to a hosted mod (for example, a mod running on `localhost` during development).

## Need help?

If you get stuck or want feedback, join our Discord community:
https://www.mangose.io/links/discord

You can ask questions, share what you’re building, and get help from the Mangose team and other mod developers. It’s also the best place to suggest improvements to the Mods SDK, including new host API functions and events you’d like to see supported.

## 2. Hello world: a counter

1. Create a file named `index.html`.
2. Paste the code below.
3. Serve it (a local server, GitHub Pages, or any hosting) and paste the URL into Mangose.

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

- Any node you plan to update must have a stable `mId` (the first argument, for example `counter-value`).
- `render(...)` sends the initial UI tree.
- `update(...)` sends only changed nodes. The host replaces nodes by `mId`.

## 3. The `MangoseMod` runtime

### Installation

If you build your mod as an npm project:

```bash
npm install @mangose/mod-runtime
```

Import the runtime:

```js
import { MangoseMod } from '@mangose/mod-runtime'
```

If you prefer a single HTML file (as shown above), you can import from the CDN:

```js
import { MangoseMod } from 'https://cdn.mangose.app/mod-runtime.js'
```

### Core APIs

- `MangoseMod.onReady(handler)`
  - Called when the host is ready. Render your initial UI tree here.
- `MangoseMod.onEvent(event, handler)`
  - Subscribes to a single UI event. Use the event name without the `event:` prefix.
- `MangoseMod.onEvents(items)`
  - Registers multiple events at once. `items` can be an array of tuples (`[event, handler]`) or an object that maps event names to handlers.
- `MangoseMod.render(node)`
  - Sends the initial UI tree to the host.
- `MangoseMod.update(...nodes)`
  - Sends incremental UI updates. You can send multiple nodes in a single call.
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
  - Creates a task in a given section. Returns the operation result (for example, the created task object).
- `MangoseMod.sections()`
  - Returns the list of sections for the current collection or view.
- `MangoseMod.getTaskInSection(sectionId)`
  - Returns tasks that belong to the given section.
- `MangoseMod.content()`
  - Returns additional collection content (for example, a description).
- `MangoseMod.setContent(value)`
  - Updates the collection content.
- `MangoseMod.getCollection()`
  - Returns metadata for the current collection (name, IDs, settings).
- `MangoseMod.getProps()`
  - Returns view or module configuration properties exposed to the mod.

## 5. Events: wiring clicks and form changes

### Clicks

Set the event name as a string (without a prefix) on the component prop:

```js
Button('save', { theme: 'primary', onClick: 'save' }, 'Save')
```

Then subscribe in your mod using `MangoseMod.onEvent('save', handler)`:

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

MangoseMod.onReady(() => {
  MangoseMod.render(ui())
})

MangoseMod.onEvent('nameChanged', (payload) => {
  // The host usually sends a primitive value or an object with `value`.
  const next = typeof payload === 'string' ? payload : payload?.value
  state.name = next ?? ''

  MangoseMod.update(Text('preview', { value: `Typed: ${state.name}` }))
})
```

### Registering multiple events at once

You can register several handlers in one call:

```js
MangoseMod.onEvents([
  ['ping', () => console.log('ping')],
  ['save', () => console.log('save')],
])

MangoseMod.onEvents({
  confirm: handleConfirm,
  cancel: handleCancel,
})
```

## 6. UI updates: simple rules that save time

- Use stable `mId`s for dynamic nodes.
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

Note: numeric `padding` and `margin` values are multiplied by 3px on the host side.

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
| `UI.DatePicker` | Date or range  | `value`, `range`, `time`, `autoApply`, `theme`, `size`       | `onChange`                                        |
| `UI.Files`      | File picker    | `selectMode`, `onlyImage`, `selectAfterUpload`               | `onChange`                                        |

### Modals and media

| Component   | Purpose         | Key props                                                                      | Events               |
| ----------- | --------------- | ------------------------------------------------------------------------------ | -------------------- |
| `UI.Modal`  | Modal dialog    | `title`, `confirmText`, `cancelText`, `showOnStart`, `width`, `disableConfirm` | `onDone`, `onCancel` |
| `UI.Iframe` | Embedded iframe | `src`, `title`, `width`, `height`                                              | none                 |

## 9. Debugging

- Use `console.log(...)` inside your mod. Check logs in the mod iframe devtools.
- Temporarily render state in the UI:

```js
Text('debug', { value: JSON.stringify(state) })
```

If the UI is not reacting:

- Ensure the updated node uses the same `mId` as in the initial render.
- Ensure the event name in props (for example, `'save'`) matches the name used in `onEvent('save', ...)`.
- Ensure you call `update(...)` after changing state.

## 10. Common pitfalls

- Missing `mId` on nodes you want to update.
- Changing `mId` between renders (the host cannot match nodes).
- Trying to style via `style` (it will be stripped).
- Typos in event names (`increment` vs `incement`).
- Do not render your entire mod exclusively inside the `UI.Iframe` component. `UI.Iframe` is intended only for auxiliary flows (for example, login, authorization, or informational screens). Mangose will not accept store mods whose full functionality lives solely inside an embedded iframe.

## 11. Publish a mod to the Mangose Store

All store mods are hosted in a public GitHub repository. To add a new mod (or update an existing one), open a Pull Request.

### Repository

- `Mangose/mods` (GitHub: https://github.com/Mangose/mods)

### Add a new mod

1. Fork the repository and create a new branch.
2. Add your mod folder under:

- `/mods/<your-mod>/`

Important:

- The entry file must be named **`index.html`**.
- Keep everything inside the mod folder (HTML, JS, CSS, assets).
- Avoid linking to third party assets unless there is a clear reason. Prefer assets committed to the repo.

3. Add an entry to:

- `/mods/store.json`

The entry should point to your mod folder and include basic metadata (name, description, author, version, and so on), following the schema in that file.

4. Open a Pull Request to `Mangose/mods` with a short description:

- What the mod does.
- How to test it.
- What changed (if this is an update).

### Update an existing mod

Updates follow the same flow as adding a new mod:

1. Update files in `/mods/<your-mod>/`.
2. If version or metadata changes, update the relevant entry in `/mods/store.json`.
3. Open a Pull Request describing the changes.

### File structure and asset size

We keep the store fast and reliable for everyone, so please follow these guidelines:

- Use a simple, predictable structure (for example, `index.html`, `assets/`, `styles/`, `scripts/`).
- Compress images and avoid shipping heavy exports.
  - Prefer JPEG, WebP, or AVIF when appropriate.
  - Keep resolution to what you actually need.
  - Avoid large PNG files unless transparency is required.
- If you add video or other large files, explain why in the PR and consider a lighter alternative.
- Ensure the mod works without additional host-side dependencies. Everything required should live in the mod folder.

These rules help us maintain high quality and fast loading times across the store.

## 12. Next steps

- Test your mod locally and inside Mangose.
- Submit it to the Mangose Store if you want to share it with others.
- Join the Mangose community to exchange ideas and best practices.
