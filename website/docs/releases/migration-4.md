# Migration guide from 3.x to 4.x

## Backward incompatible changes

Minimal required versions are:

- TypeScript (if used on the project): 4.1
- Node: v16.x

:::info

If you encounter any difficulties during the migration process, don't hesitate to ask for assistance on the Lingui [Discord server](https://discord.gg/gFWwAYnMtA).

:::

### Extractor configuration changes

The big change in v4 is in extractor internals. Now it is less fragile, and doesn't depend on the host project settings.

For most projects, it should work without extra configuration as long as it is valid ES code.

`extractBabelOptions` is not useful anymore, please delete it from your config.

```diff title="lingui.config.js"
module.exports = {
-  extractBabelOptions: { [...] }
}
```

### I18nProvider no longer remounts its children on locale change

Previously, the `I18nProvider` remounted its children on locale change. This had the effect that the whole app was re-rendered with the new locale and all strings were rendered correctly translated.
Apart from not being very performant, this approach had the drawback that the state of the app was lost - all components were re-mounted and their state was reset. This is not a standard behavior for React Context providers and could cause some confusion.

In v4, the `I18nProvider` no longer remounts its children on locale change. Instead, when locale changes, the context value provided by `I18nProvider` is updated and all components that consume the provided React Context are re-rendered with the new locale.
This includes components provided by Lingui, such as `Trans` or `Plural` and also custom components that use the `useLingui` hook. This should result in a more predictable behavior.

If the changes to the `I18nProvider` pose a problem to you, please open an issue and explain what the problem is. Additionally, you can keep using the [v3 implementation](https://github.com/lingui/js-lingui/blob/31dcab5a9a8f88bfa8b3a2c7cd12aa2d908a1d80/packages/react/src/I18nProvider.tsx#L58) by copying it into your code base and using that instead.

No migration steps are necessary for components provided by Lingui, such as `Trans` or `Plural`. However, if you rendered translations in React components using the `t` macro, you need to be sure that the `useLingui` hook is called in the component, as seen [here](/docs/ref/react.md#uselingui).

### Hash-based message ID generation and Context feature

The previous implementation had a flaw: there is an original message in the bundle at least 2 times + 1 translation.

For the line "Hello world" it'll exist in the source code as ID in i18n call, then as a key in the message catalog, and then as a translation itself. Strings could be very long, not just a couple of words, so this may bring more kB to the bundle.

A much better option is generating a "stable" ID based on the msg + context as a hash with a fixed length.

Hash would be calculated at build time by macros. So macros instead of:

```js
const message = t({
   context: 'My context',
   message: `Hello`
})

// ↓ ↓ ↓ ↓ ↓ ↓

import { i18n } from "@lingui/core"
const message = i18n._(/*i18n*/{
   context: 'My context',
   id: `Hello`
})
```

now generates:

```js
import { i18n } from "@lingui/core"
const message = i18n._(/*i18n*/{
   id: "<hash(message + context)>",
   message: `Hello`,
})
```

Also, we've added a possibility to provide a context for the message. For more details, see the [Providing a context for a message](/docs/tutorials/react-patterns.md#providing-a-context-for-a-message).

The context feature affects the message ID generation and adds the `msgctxt` parameter in case of the PO catalog format extraction.

This also affects the `orderBy` with `messageId` as now the generated id is used when custom id is absent. To avoid confusion, we switched the default `orderBy` to use the source message (`message`) instead.

### Change in generated ICU messages for nested JSX Macros

We have made a small change in how Lingui generates ICU messages for nested JSX Macros. We have removed leading spaces from the texts in all cases.

The generated code from the following nested component:
``` jsx
<Plural
  id="message.id"
  one={
    <Trans>
      One hello
    </Trans>
  }
  other={
    <Trans>
      Other hello
    </Trans>
  }
  value={count}
/>
```

was changed as follows:
``` diff
  <Trans
    id="message.id"
    message={
-     "{count, plural, one { One hello} other { Other hello}}"
+     "{count, plural, one {One hello} other {Other hello}}"
    }
    values={{
      count: count
    }}
  />
```

### Flow Syntax supported in the Extractor with the flag

If your project uses Flow, you need to explicitly enable support in the extractor:

```js title="lingui.config.js"
module.exports = {
  extractorParserOptions: {
    flow: true
  }
}
```

### `@lingui/cli/api/extractors/typescript` was deleted

Extractor supports TypeScript out of the box. Please delete it from your configuration file.

### No need to have `NODE_ENV=development` before `lingui-extract`

If your extract command looks like:

```bash
NODE_ENV=development lingui-extract
```

Now you can safely change it to just:

```bash
lingui-extract
```

### Public interface of `ExtractorType` was changed

```diff
declare type ExtractorType = {
  match(filename: string): boolean
-   extract(filename: string, targetDir: string, options?: any): void
+   extract(
+    filename: string,
+    code: string,
+    onMessageExtracted: (msg: ExtractedMessage) => void,
+    ctx?: ExtractorCtx
+  ): Promise<void> | void
}
```

Read more about custom extractors on the [Advanced: Custom Extractor](/guides/custom-extractor) page.

### Configuration migrations for deprecated options were deleted

Migration for the following older options:
- `localeDir`,
- `srcPathDirs`,
- `srcPathIgnorePatterns`,
- `fallbackLocale`

were deleted from the source code. This should only affect users who are migrating from `v2` to `v4` directly.

### Plural Rules now work out of the box without manual configuration

You can safely remove `i18n.loadLocaleData` calls because Lingui v4 uses `Intl.PluralRules` internally.

```diff
- import { en, cs } from "make-plural/plurals"

- i18n.loadLocaleData("en", { plurals: en })
- i18n.loadLocaleData("cs", { plurals: cs })
```

Don't forget to delete `make-plural` from your `package.json`.

### withI18n HOC was removed

`withI18n` HOC was removed in favor of `useLingui` hook. Please use `useLingui` instead.

If you need to use `withI18n` in your project, you can copy [the original implementation](https://github.com/lingui/js-lingui/blob/31dcab5a9a8f88bfa8b3a2c7cd12aa2d908a1d80/packages/react/src/I18nProvider.tsx#L33).
