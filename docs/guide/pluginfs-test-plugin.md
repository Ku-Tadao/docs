# PluginFS Test Plugins

PluginFS is easiest to test with small folder plugins that read and write their own local data.

Use this guide if you want to help test PluginFS or send a reproducible plugin to a maintainer.

## Folder layout

Create one of these layouts inside your Pengu Loader `plugins` folder:

```text
plugins/my-plugin/index.js
```

or:

```text
plugins/@your-name/my-plugin/index.js
```

Do not use a top-level file such as `plugins/my-plugin.js` for PluginFS testing. Top-level plugins do not receive `context.fs`.

## Minimal persistence test

Create `index.js` with this content:

```js
export async function init(context) {
  console.log('[PluginFS test] context', {
    hasFs: Boolean(context.fs),
    hasBus: Boolean(context.bus),
    meta: context.meta,
  })

  if (!context.fs) {
    console.error('[PluginFS test] context.fs is missing')
    return
  }

  await context.fs.mkdir('data')

  const previousText = await context.fs.read('data/state.json')
  const previous = previousText ? JSON.parse(previousText) : { launches: 0 }

  const next = {
    launches: previous.launches + 1,
    lastRunAt: new Date().toISOString(),
    userAgent: navigator.userAgent,
  }

  const writeOk = await context.fs.write('data/state.json', JSON.stringify(next, null, 2))
  const files = await context.fs.ls('data')
  const stat = await context.fs.stat('data/state.json')

  console.log('[PluginFS test] result', {
    writeOk,
    files,
    stat,
    next,
  })
}
```

Launch the League Client, open DevTools, and check the console for `[PluginFS test] result`.

Then restart the client and confirm `launches` increments. The file should be written inside your plugin folder:

```text
plugins/my-plugin/data/state.json
```

## Isolation test

This test confirms a plugin cannot read outside its own folder:

```js
export async function init(context) {
  if (!context.fs) return

  const ownWrite = await context.fs.write('owned.txt', 'hello')
  const ownRead = await context.fs.read('owned.txt')

  const crossRead = await context.fs.read('../some-other-plugin/index.js')
  const absoluteRead = await context.fs.read('C:/Windows/win.ini')
  const traversalWrite = await context.fs.write('../some-other-plugin/pwned.txt', 'nope')

  console.log('[PluginFS isolation]', {
    ownWrite,
    ownRead,
    crossReadDenied: crossRead === undefined,
    absoluteReadDenied: absoluteRead === undefined,
    traversalWriteDenied: traversalWrite === false,
  })
}
```

Expected result:

```js
{
  ownWrite: true,
  ownRead: 'hello',
  crossReadDenied: true,
  absoluteReadDenied: true,
  traversalWriteDenied: true,
}
```

## What to send with a test report

When reporting PluginFS test results, include:

- Your plugin folder layout.
- Console output from the test.
- Whether the written file persisted after restarting the client.
- Windows version and Pengu Loader build or artifact name.
- Any console errors.

If you are sending a plugin for someone else to test, include the whole plugin folder, not just `index.js`, so the folder layout is preserved.
