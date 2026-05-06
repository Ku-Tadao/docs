# context.fs

`context.fs` gives a folder plugin asynchronous access to files inside its own plugin directory.

It is intended for plugin-local data such as settings, generated cache files, and small text assets that need to persist across League Client restarts.

::: warning

`context.fs` is only available to folder plugins:

```text
plugins/my-plugin/index.js
plugins/@author/my-plugin/index.js
```

Top-level plugins such as `plugins/my-plugin.js` do not own a directory, so they do not receive `context.fs`.

:::

::: tip

All paths are relative to the plugin's own root directory. A plugin cannot read, write, list, stat, or remove another plugin's files.

:::

## Basic usage

```js
export async function init(context) {
  if (!context.fs) {
    console.warn('PluginFS is only available to folder plugins')
    return
  }

  await context.fs.mkdir('data')
  await context.fs.write('data/settings.json', JSON.stringify({ enabled: true }, null, 2))

  const text = await context.fs.read('data/settings.json')
  const settings = text ? JSON.parse(text) : {}

  console.log(settings)
}
```

## Security model

PluginFS is scoped by the native renderer process, not just by JavaScript helpers.

- A plugin receives access only to its own folder.
- Path traversal such as `../other-plugin/file.txt` is rejected.
- Absolute paths are rejected.
- Symlinks and reparse-point escapes are rejected.
- The native grant function is hidden from plugins after preload setup.
- Operations run asynchronously through a bounded native worker pool.

Plugins that need to communicate with each other should use JavaScript APIs, events, or shared in-memory coordination instead of editing each other's files.

## context.fs.read(path)

<Badge type="info" text="function" />

Reads a text file from the plugin directory.

### Params

- `path` - File path relative to the plugin root.

### Return value

Returns `Promise<string | undefined>`.

`undefined` means the read failed, the path was invalid, the target was not a file, or the file exceeded the text size limit.

```js
const text = await context.fs.read('data/settings.json')
```

## context.fs.write(path, content, options?)

<Badge type="info" text="function" />

Writes text to a file in the plugin directory.

### Params

- `path` - File path relative to the plugin root.
- `content` - Text content to write. It is converted to a string.
- `options.append` - Append when `true`; overwrite when omitted or `false`.

### Return value

Returns `Promise<boolean>`.

```js
await context.fs.write('data/log.txt', 'Started\n')
await context.fs.write('data/log.txt', 'Ready\n', { append: true })
```

::: tip

Create parent directories with `context.fs.mkdir()` before writing nested files.

:::

## context.fs.mkdir(path)

<Badge type="info" text="function" />

Creates a directory path inside the plugin directory.

### Params

- `path` - Directory path relative to the plugin root.

### Return value

Returns `Promise<boolean>`.

```js
await context.fs.mkdir('data/cache')
```

## context.fs.stat(path?)

<Badge type="info" text="function" />

Gets basic information about a file or directory.

### Params

- `path` - File or directory path relative to the plugin root. Omit it to stat the plugin root.

### Return value

Returns `Promise<FileStat | undefined>`.

```ts
interface FileStat {
  fileName: string
  length: number
  isDir: boolean
  isFile: boolean
}
```

```js
const stat = await context.fs.stat('data/settings.json')
if (stat?.isFile) {
  console.log(`${stat.fileName}: ${stat.length} bytes`)
}
```

## context.fs.ls(path?)

<Badge type="info" text="function" />

Lists direct children of a directory.

### Params

- `path` - Directory path relative to the plugin root. Omit it to list the plugin root.

### Return value

Returns `Promise<string[] | undefined>`.

```js
const files = await context.fs.ls('data')
console.log(files)
```

## context.fs.rm(path, options?)

<Badge type="info" text="function" />

Removes a file or directory inside the plugin directory.

### Params

- `path` - File or directory path relative to the plugin root.
- `options.recursive` - Remove a directory tree when `true`.

### Return value

Returns `Promise<number>`, the number of removed filesystem entries.

```js
await context.fs.rm('data/old-cache.txt')
await context.fs.rm('data/cache', { recursive: true })
```

::: danger

Only remove files your plugin created or clearly owns. Recursive removal cannot affect other plugin folders, but it can still delete your plugin's own data.

:::
