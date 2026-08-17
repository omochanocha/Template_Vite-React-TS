# ESLint + Prettier → oxc (oxlint / oxfmt) 移行計画

## Context

このリポジトリは Vite + React + TypeScript の**環境構築用テンプレート**であり、ここで敷いた lint / format 構成が「Use this template」で作られる全プロジェクトに複製される。現状は ESLint 9 (flat config, 15 パッケージ) + Prettier + stylelint の 3 本立てで、`eslint.config.js` は 205 行・`recommendedTypeChecked` を含む重い構成になっている。

これを oxc スタック (Rust 製の oxlint / oxfmt) に移行し、**依存パッケージを大幅に削減しつつ実行速度を上げる**のが目的。ESLint と Prettier は完全に削除する。stylelint は oxc に等価物が無いため維持する。

移行判断はすべて scratchpad 上の複製で**実際にコマンドを実行して裏を取った**（本リポジトリは未変更）。バージョンは 2026-08-16 時点で `oxlint@1.78.0` / `oxfmt@0.63.0` / `oxlint-tsgolint@7.0.2001` / `typescript@7.0.2`。

---

## 決定事項サマリ

| 項目                | 決定                                                                                    |
| ------------------- | --------------------------------------------------------------------------------------- |
| 移行手段            | 公式ガイド (oxc.rs 英語版) に従い**設定は手書き**。`@oxlint/migrate` の生成物は使わない |
| Prettier            | **完全削除**。md / html / css も含めて oxfmt が全面的に担当                             |
| stylelint           | **維持**。CSS の _整形_ は oxfmt、_プロパティ順序とバグ検出_ は stylelint               |
| TypeScript          | **7.0.2 へ昇格**し、`oxlint --type-aware` で型情報ルールを維持                          |
| ルールセット        | `categories.correctness` + `unicorn` + 明示ルール（現行の検出範囲を維持）               |
| type-aware の有効化 | **CLI フラグと VSCode 設定のみ**。`.oxlintrc.json` には書かない（後述 R3）              |
| 分割                | **3 ステップ / 3 コミット**                                                             |
| VSCode              | `.vscode/settings.json` を git 追跡対象に変更し、整形設定をテンプレートに載せる         |

**対象外（別案件）**: pnpm 移行、vite 6→8 / `@vitejs/plugin-react` 4→6 の更新、CI (`.github/workflows`) の新規追加。

---

## 検証済みの事実

### 互換性: 失われるルールは 3 つだけ

| 現行ルール                                                                                                                                                | 移行後                                                                                                                                                                                                                                                |
| --------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `@stylistic/padding-line-between-statements`                                                                                                              | ❌ **失われる**。oxlint に `@stylistic` プラグインが無く、oxfmt にも文間の空行制御が無い                                                                                                                                                              |
| `simple-import-sort/exports`                                                                                                                              | ❌ **失われる**。oxfmt に `sortExports` は存在しない                                                                                                                                                                                                  |
| `import/no-unresolved`                                                                                                                                    | ❌ 失われるが `tsc -b` が実質カバー。`@oxlint/migrate` も「モジュール解決の複雑さで必ず false positive が出る」として意図的に未実装                                                                                                                   |
| `simple-import-sort/imports`                                                                                                                              | ✅ oxfmt の `sortImports` で再現。現行 `src/App.tsx` の並び（react → 空行 → assets → css）と一致することを実測                                                                                                                                        |
| `unused-imports/*`                                                                                                                                        | ✅ `eslint/no-unused-vars` に統合。`fix: { imports: "safe-fix" }` 指定で `oxlint --fix` により未使用 import が削除されることを実測（`--quiet` 併用でも動作）                                                                                          |
| `@typescript-eslint/no-magic-numbers`                                                                                                                     | ✅ `eslint/no-magic-numbers` に統合。`typescript/no-magic-numbers` は**存在しない**（docs 404）が、core 版が `ignoreEnums` / `ignoreReadonlyClassProperties` / `ignoreTypeIndexes` を実装しており、enum 値・readonly プロパティが除外されることを実測 |
| `import/named`                                                                                                                                            | ❌ **失われる**。ルール自体は存在するが、存在しない named export を import しても発火しないことを実測（未実装に近い挙動）。`tsc -b` が `TS2724` で正確に代替することを確認済み（`import/no-unresolved` と同じ扱い）                                   |
| その他（react / react-hooks / react-refresh / jsx-a11y / import の一部 / no-console / complexity / consistent-type-imports / strict-boolean-expressions） | ✅ すべて移行可能。ただし多くは `categories.correctness` だけでは有効化されず、下記 R1 の通り明示列挙が必要                                                                                                                                           |
| `eslint-plugin-jest`                                                                                                                                      | ✅ **vitest に置き換えて移行可能**。oxlint に `jest` プラグイン（12 correctness ルール）と `vitest` プラグイン（17 correctness ルール）が両方あり、後者を採用する（下記参照）                                                                         |

### TypeScript 7 のブロッカーは `baseUrl` のみ

```
tsconfig.app.json(33,5): error TS5102: Option 'baseUrl' has been removed.
```

`baseUrl` を削除し `paths: { "@/*": ["./src/*"] }` のみ残した構成で、TS **7.0.2** / TS **5.9.3**（現行）の**どちらもエラー 0**。→ `baseUrl` 削除は後方互換なので Step 1 として単独でコミットできる。

エコシステム: 隔離環境で `typescript@7.0.2` + `@types/react@19.2.18` + `oxlint` + `oxlint-tsgolint` を `npm install` → **peer 依存の競合なし**。`oxlint --type-aware` で `typescript/strict-boolean-expressions` が発火することも、**現行と同じ solution-style tsconfig（`files: []` + `references`）で tsconfig が自動探索されること**も実測済み。

### `@oxlint/migrate` の生成物は使えない

`--details --type-aware` の実行結果は `202 rules created / Skipped 9 rules` と成功に見えるが、生成物を oxlint に渡すと失敗する:

1. `Invalid configuration for rule 'jsx_a11y/control-has-associated-label': unknown field 'includeRoles'`
2. `Plugin '@stylistic' not found` / `'simple-import-sort' not found` / `'unused-imports' not found` — **これら 5 ルールを「skipped」と報告せずそのまま書き出している**
3. `"no-unused-vars": "off"` を src 配下に出力する。**修正すると未使用変数・未使用 import の検出が丸ごと消える**
4. WSL2 上で、生成された設定を渡すと oxlint が `oxc_allocator/src/pool/fixed_size.rs:112` で panic（再現性 100%）。手書き設定では再現しない

→ **設定は手書きする。** ただし移行対象ルールの洗い出しには有用だったので、本計画の `typescript/*` ルール一覧はこの出力から抽出済み（下記に確定リストとして記載してあるので、実装時に migrate を再実行する必要はない）。

### oxfmt は Prettier の出力を再現する

`printWidth: 80` + `singleQuote: true` を指定し、**意図的に崩した入力**で md / html / css を整形させて Rust 側で実際に処理されることを確認済み（インデント破壊・長行・`*` → `-`・table 整列・`<!DOCTYPE>` → `<!doctype>`・CSS の 1 行圧縮展開がすべて正しく行われた）。その上でリポジトリの実ファイル `src/App.tsx` / `src/main.tsx` / `vite.config.ts` / `src/App.css` / `src/index.css` / `README.md` / `index.html` すべてで **prettier の出力とバイト単位で一致**。

`oxfmt@0.63.0` の依存は `tinypool` + ネイティブバイナリのみで **prettier を含まない**。よって Prettier は完全に削除できる。

### ⚠️ 既存の stylelint 違反が 1 件ある

現状の `npm run lint` は `;` 連結のため stylelint の失敗を握り潰しており、以下が隠れている:

```
src/index.css
  9:3  ✖  Expected "color-scheme" to come before "background-color"  order/properties-order
```

**Step 2 で終了コードの伝播を修正すると `npm run lint` がここで失敗するようになる。** Step 2 の作業に `npm run lint:style:fix` の実行を含めること。なお現行 ESLint 側は `EXIT=0`（クリーン）であることを確認済み。

---

## Step 1 — `baseUrl` の削除（TS 5.9 のまま、単独コミット）

- **`tsconfig.json`** — `compilerOptions.baseUrl` の行を削除（`paths` は残す）
- **`tsconfig.app.json`** — 同じく `baseUrl` の行を削除（`paths` は残す）

検証: `npm run build` が通ること。加えて `npm run dev` で画面が表示されること（`src/main.tsx:3` の `@/App.tsx` が `vite-tsconfig-paths` 経由で解決されるのは tsc とは別経路のため、型チェックだけでは確認できない）。

---

## Step 2 — oxc への移行（ESLint / Prettier 完全削除）

### 新規 `.oxlintrc.json`

```jsonc
{
  "$schema": "./node_modules/oxlint/configuration_schema.json",
  "plugins": [
    "typescript",
    "react",
    "jsx-a11y",
    "import",
    "vitest",
    "unicorn",
    "oxc",
  ],
  "categories": { "correctness": "error" },
  "env": { "builtin": true, "browser": true, "es2026": true },
  "ignorePatterns": [
    "{dist,build,public,node_modules}/**",
    "**/lib/utils.{js,ts}",
    "**/components/ui/**/*.{jsx,tsx}",
    "**/*.config.*",
  ],
  "rules": {
    // --- eslint:recommended のうち correctness カテゴリに含まれないもの（下記 R1 参照） ---
    "no-case-declarations": "error",
    "no-empty": "error",
    "no-fallthrough": "error",
    "no-prototype-builtins": "error",
    "no-redeclare": "error",
    "no-regex-spaces": "error",
    "no-unexpected-multiline": "error",
    "no-var": "error",
    "no-array-constructor": "error",

    // --- 現行 eslint.config.js の明示ルール ---
    "no-console": "warn",
    "complexity": ["warn", { "max": 10 }],
    "no-unused-vars": [
      "warn",
      {
        "args": "after-used",
        "argsIgnorePattern": "^_",
        "caughtErrorsIgnorePattern": "^_",
        "destructuredArrayIgnorePattern": "^_",
        "vars": "all",
        "varsIgnorePattern": "^_",
        "fix": { "imports": "safe-fix" },
      },
    ],
    "react/react-in-jsx-scope": "off",
    "react/only-export-components": "error",
    "react/display-name": "error",
    "react/jsx-no-comment-textnodes": "error",
    "react/jsx-no-target-blank": "error",
    "react/no-unescaped-entities": "error",
    "react/no-unknown-property": "error",
    "jsx-a11y/alt-text": "error",
    "jsx-a11y/anchor-ambiguous-text": "error",
    "jsx-a11y/anchor-has-content": "error",
    "jsx-a11y/anchor-is-valid": "error",
    "jsx-a11y/aria-activedescendant-has-tabindex": "error",
    "jsx-a11y/aria-props": "error",
    "jsx-a11y/aria-proptypes": "error",
    "jsx-a11y/aria-role": "error",
    "jsx-a11y/aria-unsupported-elements": "error",
    "jsx-a11y/autocomplete-valid": "error",
    "jsx-a11y/click-events-have-key-events": "error",
    "jsx-a11y/control-has-associated-label": "error",
    "jsx-a11y/heading-has-content": "error",
    "jsx-a11y/html-has-lang": "error",
    "jsx-a11y/iframe-has-title": "error",
    "jsx-a11y/img-redundant-alt": "error",
    "jsx-a11y/interactive-supports-focus": "error",
    "jsx-a11y/label-has-associated-control": "error",
    "jsx-a11y/media-has-caption": "error",
    "jsx-a11y/mouse-events-have-key-events": "error",
    "jsx-a11y/no-access-key": "error",
    "jsx-a11y/no-autofocus": "error",
    "jsx-a11y/no-distracting-elements": "error",
    "jsx-a11y/no-interactive-element-to-noninteractive-role": "error",
    "jsx-a11y/no-noninteractive-element-interactions": "error",
    "jsx-a11y/no-noninteractive-element-to-interactive-role": "error",
    "jsx-a11y/no-noninteractive-tabindex": "error",
    "jsx-a11y/no-redundant-roles": "error",
    "jsx-a11y/no-static-element-interactions": "error",
    "jsx-a11y/role-has-required-aria-props": "error",
    "jsx-a11y/role-supports-aria-props": "error",
    "jsx-a11y/scope": "error",
    "jsx-a11y/tabindex-no-positive": "error",
    "import/extensions": [
      "error",
      "always",
      {
        "js": "always",
        "jsx": "always",
        "ts": "always",
        "tsx": "always",
        "ignorePackages": true,
      },
    ],
    "import/first": "error",
    "import/newline-after-import": "error",
    "import/no-duplicates": "error",
    "import/no-named-as-default": "warn",
    "import/no-named-as-default-member": "warn",

    // --- typescript-eslint recommendedTypeChecked 相当（42 件、下記 R2 参照） ---
    "typescript/await-thenable": "error",
    "typescript/ban-ts-comment": "error",
    "typescript/no-array-delete": "error",
    "typescript/no-base-to-string": "error",
    "typescript/no-duplicate-enum-values": "error",
    "typescript/no-duplicate-type-constituents": "error",
    "typescript/no-empty-object-type": "error",
    "typescript/no-explicit-any": "error",
    "typescript/no-extra-non-null-assertion": "error",
    "typescript/no-floating-promises": "error",
    "typescript/no-for-in-array": "error",
    "typescript/no-implied-eval": "error",
    "typescript/no-misused-new": "error",
    "typescript/no-misused-promises": "error",
    "typescript/no-namespace": "error",
    "typescript/no-non-null-asserted-optional-chain": "error",
    "typescript/no-redundant-type-constituents": "error",
    "typescript/no-require-imports": "error",
    "typescript/no-this-alias": "error",
    "typescript/no-unnecessary-type-assertion": "error",
    "typescript/no-unnecessary-type-constraint": "error",
    "typescript/no-unsafe-argument": "error",
    "typescript/no-unsafe-assignment": "error",
    "typescript/no-unsafe-call": "error",
    "typescript/no-unsafe-declaration-merging": "error",
    "typescript/no-unsafe-enum-comparison": "error",
    "typescript/no-unsafe-function-type": "error",
    "typescript/no-unsafe-member-access": "error",
    "typescript/no-unsafe-return": "error",
    "typescript/no-unsafe-unary-minus": "error",
    "typescript/no-wrapper-object-types": "error",
    "typescript/only-throw-error": "error",
    "typescript/prefer-as-const": "error",
    "typescript/prefer-namespace-keyword": "error",
    "typescript/prefer-promise-reject-errors": "error",
    "typescript/require-await": "error",
    "typescript/restrict-plus-operands": "error",
    "typescript/restrict-template-expressions": "error",
    "typescript/triple-slash-reference": "error",
    "typescript/unbound-method": "error",

    // --- typescript-eslint stylistic 相当（13 件） ---
    "typescript/adjacent-overload-signatures": "error",
    "typescript/array-type": "error",
    "typescript/ban-tslint-comment": "error",
    "typescript/class-literal-property-style": "error",
    "typescript/consistent-generic-constructors": "error",
    "typescript/consistent-indexed-object-style": "error",
    "typescript/consistent-type-assertions": "error",
    "typescript/consistent-type-definitions": "error",
    "typescript/no-confusing-non-null-assertion": "error",
    "typescript/no-inferrable-types": "error",
    "typescript/no-empty-function": "error",
    "typescript/prefer-for-of": "error",
    "typescript/prefer-function-type": "error",
  },
  "overrides": [
    {
      "files": ["{src,app,pages}/**/*.{ts,tsx}"],
      "rules": {
        "typescript/consistent-type-imports": [
          "warn",
          { "prefer": "type-imports" },
        ],
        "prefer-const": "error",
        "prefer-rest-params": "error",
        "prefer-spread": "error",
        "no-magic-numbers": [
          "warn",
          {
            "ignore": [-1, 0, 1],
            "ignoreEnums": true,
            "ignoreReadonlyClassProperties": true,
            "ignoreTypeIndexes": true,
          },
        ],
      },
    },
  ],
}
```

**R1 — なぜ `categories.correctness` だけでは足りないか**
`@eslint/js` の recommended 61 ルールと oxlint の `correctness` カテゴリ展開結果（`oxlint --print-config` で取得）を突き合わせた結果、**10 件が漏れる**ことを確認した。うち 7 件を上記で明示的に有効化している。残る 3 件は不要:

- `no-undef` — oxlint では nursery。TypeScript を使う本構成では `tsc` が同等の検査をする
- `no-dupe-args` / `no-octal` — `@oxlint/migrate` が "Superseded by strict mode" として除外。ESM では常に strict mode

なお `categories` を `suspicious` まで広げる案は棄却した。実 src に対して `no-shadow` と `import/no-unassigned-import`（`import './App.css'` に誤爆）が新たに発火し、現行 ESLint には無いノイズが出るため。上記 7 件の個別指定なら**実 src で指摘ゼロ**であることを確認済み。

**同じ手法を `recommendedTypeChecked` / `stylistic`（typescript-eslint パッケージ本体を import して実ルール集合を取得）と react / jsx-a11y / import の recommended（実際の違反コードで発火確認）にも適用した。** 前者では `no-var` / `prefer-const` / `prefer-rest-params` / `prefer-spread` / `no-array-constructor` の 5 件、後者では react 5 件・jsx-a11y 33 件・`import/no-named-as-default(-member)` の欠落が見つかり、上記 `.oxlintrc.json` に反映済み（詳細は本文末尾の棄却事項リスト）。`oxlint --print-config` の `rules` フィールドは「デフォルトで有効なルール」を必ずしも反映しないため、この 2 系統の突き合わせは `--print-config` ではなく実行結果で判定している。

**R2 — 型情報が必要なルールを Step 2 の時点で書いてよいか**
よい。`--type-aware` を付けずに実行しても**エラーにならず静かにスキップされる**（実測）。同時に `typescript/array-type` などの非 type-aware ルールは Step 2 の時点から正しく発火する。Step 3 でフラグを足すだけで型情報ルールが起動する。

**その他の補足**

- `argsIgnorePattern` / `varsIgnorePattern` は**明示必須**。oxlint はオプションをオブジェクトで書くとデフォルトの `^_` が消える（ESLint と挙動が違う）
- `react-hooks/*` と `react-refresh/*` は oxlint では `react` プラグインに統合されている。`react/only-export-components` の severity `"error"` は現行 `eslint-plugin-react-refresh` の recommended（`["error", {}]`）と一致することを確認済み
- **テストランナーは jest ではなく vitest に切り替える。** 現行 `jest` devDependency は未使用（設定もテストも無い）なので、oxlint 側も `jest` プラグインではなく `vitest` プラグインを採用する。`plugins` に `vitest` を含めるだけで `correctness` カテゴリの 17 ルール（`expect-expect` / `no-disabled-tests` を含む）が発火することを実測済み（jest プラグインは 12 ルール）。ファイル名による絞り込みは不要 ―― `describe`/`it`/`test` の呼び出しパターンで判定するため、現行 `eslint.config.js:170` にあった `files` glob（タイポ `{js,ts,jsxt,sx}` を含む）ごと丸ごと不要になる
- **`vitest` 自体（テストランナー）を devDependency に追加するかは本計画のスコープ外とする。** 今回はテスト 0 件の状態でのリンタ設定の置き換えのみを行う。実際にテストを書き始める際に `vitest` を追加すること
- `unicorn` は `correctness` のみ有効な本構成では `no-null` / `prefer-node-protocol` / `filename-case` 等の意見の強いルールが発火しないことを実測済み。一方 `no-await-in-promise-methods` / `no-invalid-remove-event-listener` / `no-invalid-fetch-options` / `no-single-promise-in-promise-methods` / `no-useless-fallback-in-spread` など**他のプラグインが拾わない実バグ**を検出できることを、別途用意したサンプルコードで確認済み（本リポジトリの src には該当箇所は無い）
- `env.es2026` と `ignorePatterns` のブレース展開（`{dist,build,...}/**`）は、いずれも oxlint が受理・解釈することを `--print-config` と実行で確認済み

### 新規 `.oxfmtrc.json`

```jsonc
{
  "$schema": "./node_modules/oxfmt/configuration_schema.json",
  "printWidth": 80,
  "singleQuote": true,
  "ignorePatterns": [
    "{dist,build,public,node_modules}/**",
    "**/*.min.*",
    "**/*-lock.{json,yaml,yml}",
  ],
  "sortImports": {
    "newlinesBetween": false,
    "internalPattern": ["@/"],
    "customGroups": [
      {
        "groupName": "react",
        "elementNamePattern": [
          "react",
          "react-dom",
          "react/**",
          "react-dom/**",
        ],
      },
      {
        "groupName": "assets",
        "elementNamePattern": [
          "*.json",
          "**/*.json",
          "*.svg",
          "**/*.svg",
          "*.png",
          "**/*.png",
          "*.jpg",
          "**/*.jpg",
        ],
      },
    ],
    "groups": [
      "react",
      "builtin",
      "external",
      "internal",
      ["parent", "sibling", "index"],
      { "newlinesBetween": true },
      "assets",
      ["style", "side_effect_style"],
      "side_effect",
      "unknown",
    ],
  },
}
```

- **`printWidth: 80` は必須**。oxfmt のデフォルトは 100 で、省略すると全ファイルが再整形される
- `side_effect_style` を `style` と同じ配列に入れるのが要点。無いと `import './App.css'` が先頭に飛ぶ
- `ignorePatterns` により `package-lock.json` が対象外になることを実測で確認済み
- `sortPackageJson` はデフォルト ON のまま。初回に `package.json` の `private` / `version` の順序が入れ替わるが以後安定する

### 削除するファイル

`eslint.config.js` / `prettier.config.js`

### `package.json`

**削除する devDependencies（20 個）**:
`@eslint/config-inspector`, `@eslint/js`, `@stylistic/eslint-plugin`, `@types/eslint-plugin-jsx-a11y`, `eslint`, `eslint-config-prettier`, `eslint-import-resolver-typescript`, `eslint-plugin-import`, `eslint-plugin-jest`, `eslint-plugin-jsx-a11y`, `eslint-plugin-react`, `eslint-plugin-react-hooks`, `eslint-plugin-react-refresh`, `eslint-plugin-simple-import-sort`, `eslint-plugin-unused-imports`, `globals`, `prettier`, `typescript-eslint`, `@types/lint-staged`（lint-staged 本体は未導入）, `jest`（設定もテストも無い）

**追加**: `oxlint`, `oxfmt`

**scripts**:

```jsonc
{
  "lint:oxlint": "oxlint",
  "lint:oxlint:fix": "oxlint --fix",
  "lint:style": "stylelint 'src/**/*.{css,less,sass,scss}'",
  "lint:style:fix": "stylelint --fix 'src/**/*.{css,less,sass,scss}'",
  "lint": "npm run --silent lint:style; ST=$?; npm run --silent lint:oxlint; ES=$?; [ $ST -eq 0 ] && [ $ES -eq 0 ]",
  "lint:fix": "npm run --silent lint:style:fix; ST=$?; npm run --silent lint:oxlint:fix; ES=$?; [ $ST -eq 0 ] && [ $ES -eq 0 ]",
  "format": "oxfmt",
  "format:check": "oxfmt --check",
}
```

- `lint` / `lint:fix` は**両方を必ず実行した上で、どちらかが失敗したら失敗を返す**。現行の `;` 連結は stylelint の失敗を握り潰していた（実際に `src/index.css` の違反が隠れている）
- **トレードオフ**: `ST=$?` / `[ ]` は POSIX sh 構文で Windows の `cmd.exe` では動かない。現行の `;` 連結も同じく POSIX 前提であり、このテンプレートは元から WSL / macOS / Linux を想定しているため踏襲する。Windows ネイティブ対応が必要になった時点で `npm-run-all2` の導入を検討する
- `oxlint` を引数なしで実行し、対象範囲は `.oxlintrc.json` の `ignorePatterns` を唯一の定義とする。現行の `'src/**/*.{js,jsx,ts,tsx}'` に対し、今後 src 外に置かれる `.ts` が lint 対象になる点だけ挙動が広がる（`**/*.config.*` は除外されるため現状の差はゼロ）

### `lefthook.yaml`

```yaml
pre-commit:
  parallel: false
  commands:
    1_oxfmt:
      glob: '*.{js,mjs,cjs,jsx,ts,mts,cts,tsx,html,htm,css,scss,sass,less,json,jsonc,yaml,yml,graphql,gql,md,mdx}'
      exclude:
        - '**/*.min.*'
        - '**/*-lock.{json,yaml,yml}'
      run: npx oxfmt {staged_files}
      stage_fixed: true
    2_oxlint:
      glob: '*.{js,mjs,cjs,ts,mts,cts,jsx,tsx}'
      run: npx oxlint {staged_files} --fix --quiet
      stage_fixed: true
```

**`parallel: true` → `false` に変更する。** 整形（import ソート）と lint fix（未使用 import 削除）は順序が結果に影響するため。数字プレフィックスで実行順を固定する。type-aware は付けない（速度優先）。

### `.gitignore`

`!.vscode/extensions.json` の直後に 1 行追加:

```
!.vscode/settings.json
```

現状 `.vscode/settings.json` は git 管理外で、`editor.formatOnSave: true` はローカルにしか存在せずテンプレートに引き継がれていない。

### `.vscode/settings.json`

```jsonc
{
  "typescript.tsdk": "node_modules/typescript/lib",
  "editor.defaultFormatter": "oxc.oxc-vscode",
  "editor.formatOnSave": false,
  "editor.codeActionsOnSave": {
    "source.format.oxc": "always",
    "source.fixAll.oxc": "always",
  },
}
```

- 拡張は `oxc.oxc-vscode` 1 つ（oxlint と oxfmt が統合済み）。バイナリは同梱されずプロジェクトローカルの `oxlint` / `oxfmt` を `--lsp` で使うため、devDependencies に入れる本構成と噛み合う
- `formatOnSave: false` + code action 2 本は「整形 → lint fix の順序を確定させたい場合」の公式推奨パターン。`formatOnSaveMode` は `formatOnSave: false` 下では効かないため指定しない
- ⚠️ **既存の `"github.copilot.enable": { "*": false }` を削除する。** settings.json を追跡対象にする以上、個人的な Copilot 無効化設定を全派生プロジェクトに配ることになるため。**これは作業者本人のローカル設定を実際に消す変更なので、実施前に一声かけること**

### `.vscode/extensions.json`（新規）

```json
{
  "recommendations": ["oxc.oxc-vscode", "stylelint.vscode-stylelint"]
}
```

### `README.md`

5 行目 `eslint, prettier, stylelintインストール済みで、コミットすると自動でリントと整形を行う。` を oxlint / oxfmt / stylelint 構成に書き換える。**README.md には現在あなたの未コミットの変更（pnpm / ghq の追記）があるので、それを保持したままこの 1 行のみを修正する。**

### Step 2 の作業に含めること

`npm run lint:style:fix` を実行し、既存の `src/index.css` の `order/properties-order` 違反を解消する（上記の通り、終了コード伝播の修正によりこれが表面化するため）。

---

## Step 3 — TypeScript 7 昇格と type-aware linting

### `package.json`

- `typescript`: `^5.9.3` → `^7.0.2`
- devDependencies に `oxlint-tsgolint` を追加
- `lint:oxlint`: `oxlint` → `oxlint --type-aware`
- `lint:oxlint:fix`: `oxlint --fix` → `oxlint --type-aware --fix`

### `.oxlintrc.json`

`{src,app,pages}/**/*.{ts,tsx}` の override に 1 行追加するのみ:

```jsonc
"typescript/strict-boolean-expressions": "error"
```

**R3 — `options: { "typeAware": true }` は書かない。** 設定ファイルに書くと CLI フラグ無しの `npx oxlint {staged_files}`（lefthook）まで type-aware になり、「pre-commit は速度優先で type-aware なし」という方針と矛盾する（実測で確認）。有効化は `lint:oxlint` の CLI フラグと、下記の VSCode 設定で行う。

**R4 — `tsc -b`（Step 3 以降）は tsgo（型検査）を含むか。** 含む。TypeScript 7.0.2 の `tsc` バイナリの実体は Node.js の薄いラッパー（`lib/tsc.js`、28 行）で、`getExePath()` が返すネイティブ実行ファイルに `execFileSync` で処理を委譲するだけであることをパッケージの中身を直接確認した。つまり **TypeScript 7 系では `tsc` コマンド自体が Project Corsa（旧 tsgo）のネイティブコンパイラそのもの**であり、`tsgo` という別コマンドを追加で叩く必要はない。Step 3 で `typescript` を `^7.0.2` に上げれば、既存の `npm run build`（`tsc -b && vite build`）の型検査は自動的にネイティブコンパイラで実行される。`oxlint-tsgolint`（lint 用）とは別物で、こちらは lint ルール実行のために typescript-go を内部的に利用するツールであり、`tsc` とは独立したバイナリ。

Step 2 で既に `typescript/*` ルールを 54 個（recommendedTypeChecked 相当 40 + stylistic 相当 13 + `consistent-type-imports`）記述済みなので、**Step 3 で追加する型情報ルールは `strict-boolean-expressions` の 1 行だけ**。それらが Step 3 で初めて起動する。

### `.vscode/settings.json`

エディタでも型情報ルールを効かせるため 1 行追加:

```jsonc
"oxc.typeAware": true
```

### 補足

`tsgolint` は typescript-go を内包しており `node_modules/typescript` を読まない（npm メタデータ上も typescript への依存・peer 制約が無い）。実際に効くのは tsconfig が TS 7 で解釈できるかどうかで、それは Step 1 で解消済み。solution-style tsconfig でも自動探索が働くことは実測済みなので、`--tsconfig` の明示指定は不要。

---

## 検証手順

**Step 1**

```bash
npm run build          # tsc -b && vite build が通ること
npm run dev            # 画面表示 = @/App.tsx の解決確認
```

**Step 2**

```bash
npm install            # ESLint 系 20 パッケージが消え oxlint/oxfmt が入ること
npm run format         # 初回は差分が出る（下記注記）
npm run lint:style:fix # 既存の src/index.css 違反を解消
npm run lint           # stylelint + oxlint。exit 0 になること
npm run format:check   # 差分なし
npm run build && npm run dev
```

> 注記: `package.json` は `preinstall: npx typesync`、oxfmt の `sortPackageJson`、そして本作業自体の 3 者が触る。初回の差分がどれ由来か切り分けられるよう、`npm install` と `npm run format` は分けて実行し、それぞれの後に `git diff package.json` を見ること。

移行前後で検出内容が変わっていないことは、意図的な違反コードを**一時ファイル**に書いて確認する（**確認後は必ず削除すること**）。単に exit 0 を見るのではなく、以下が**実際に検出されること**を確認する:

| 検証項目                                 | 期待されるルール                                          |
| ---------------------------------------- | --------------------------------------------------------- |
| `console.log`                            | `no-console`                                              |
| 同一モジュールの二重 import              | `import/no-duplicates`                                    |
| `useEffect` の依存配列漏れ               | `react-hooks/exhaustive-deps`                             |
| `<img>` の `alt` 欠落                    | `jsx-a11y/alt-text`                                       |
| 拡張子なしの相対 import                  | `import/extensions`                                       |
| 式中のマジックナンバー（`100 * 7` など） | `no-magic-numbers`                                        |
| `o.hasOwnProperty('x')`                  | `no-prototype-builtins`（R1 の追加ルール）                |
| `switch` の `case` 内 `const`            | `no-case-declarations`（同上）                            |
| 未使用の変数・import                     | `no-unused-vars`（`oxlint --fix` で import が消えること） |

pre-commit の確認: import 順序が崩れ未使用 import を含むファイルを `git add` → `git commit` し、oxfmt が並べ替えた後に oxlint が未使用 import を削除してコミットされること。

**Step 3**

```bash
npm install
npx tsc -b             # TS 7.0.2 でエラー 0
npm run lint           # --type-aware 込みで exit 0
npm run build
```

型情報ルールが**実際に起動していること**の確認（一時ファイル、確認後削除）。ここで何も出なければ type-aware が動いていないので必ず落とすこと:

```ts
export function f(s: string | undefined): string {
  if (s) {
    return s;
  } // ← typescript(strict-boolean-expressions) が出るはず
  return 'x';
}
```

---

## リスクと留意点

| 項目                                    | 内容                                                                                                                                                                                                                                                                                                           |
| --------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **oxlint の panic**                     | WSL2 上で `@oxlint/migrate` 生成の設定を渡すと oxlint 1.78.0 がクラッシュする（再現性 100%）。本計画の `.oxlintrc.json` 最終形（root 106 ルール + override 6 ルール）を実 src に 5 回連続実行し panic ゼロ・ノイズゼロを確認済み                                                                               |
| **oxfmt が Beta**                       | `oxfmt@0.63.0` は 0.x 系で stable/1.0 のアナウンスがまだ無い。全ファイルで Prettier との出力一致を実測したが、将来のバージョンで整形結果が変わる可能性はある。バージョン固定するかは運用判断                                                                                                                   |
| **失われる 3 ルール**                   | `@stylistic/padding-line-between-statements`（`return` / `if` / `function` 前の空行強制）は代替が無く慣習で維持することになる。直近コミット `042a7e4` 周辺で整備した設定なので、運用してみて不便なら ESLint の部分的再導入を再検討する                                                                         |
| **`vite.config.ts` の import 並び替え** | 現行 ESLint が `**/*.config.*` を除外していたため未整列。oxfmt は整形するので Step 2 で差分が出る（`@vitejs/plugin-react` と `vite` の順序入れ替えのみ、確認済み）                                                                                                                                             |
| **settings.json の追跡開始**            | 派生プロジェクトでのローカルな VSCode 設定変更が git 差分に現れるようになる                                                                                                                                                                                                                                    |
| **`no-unsafe-*` 系の感度**              | `typescript/no-unsafe-assignment` 等は型宣言が解決できないと大量に発火する（検証環境で `vite/client` が無い状態で再現した）。現行 ESLint も同じルール群を `recommendedTypeChecked` 経由で有効化しており `EXIT=0` なので実プロジェクトでは問題ないが、Step 3 で大量に出た場合はまず tsconfig の型解決を疑うこと |

## レビューで指摘され、実測により棄却した事項

同じ点を再検討しなくて済むよう記録しておく。

- 「`typescript/no-magic-numbers` を使うべき」→ **存在しない**（docs 404）。core の `eslint/no-magic-numbers` が TS 専用オプションを実装しており、`--print-config` でオプション保持を、enum/readonly の除外動作を実行で確認済み
- 「oxfmt の md/html/css 一致検証は循環している」→ 意図的に崩した入力で再検証し、Rust 側で実際に整形されることを確認済み
- 「`--quiet` だと `--fix` で未使用 import が消えない」→ 消える（実測）
- 「solution-style tsconfig では type-aware が動かない」→ 動く（実測）
- 「`ignorePatterns` のブレース展開が効かない」→ 効く（実測）
- 「`react/only-export-components` は warn にすべき」→ 現行 recommended が `["error", {}]` なので `error` が正しい
- 「`import/extensions` は未実装かもしれない」→ 実装済み、発火を確認
- 「`env.es2026` は未知キーかもしれない」→ 受理される（`--print-config` で確認）
- 「README.md の pnpm 追記は既にコミット済み」→ **これはレビューアの誤り**。`git status` は現在も `M README.md` を示しており未コミット（コミット `726c5f4` は pnpm/ghq 追記より前）。当初の記述（保持したまま 1 行のみ修正）が正しい
- 「`recommendedTypeChecked`/`stylistic` 相当の `typescript/*` リストに漏れがある（`no-array-constructor` / `no-unused-expressions` / `no-empty-function` および `no-var`/`prefer-const`/`prefer-rest-params`/`prefer-spread`）」→ **正当な指摘。** `typescript-eslint` パッケージを直接 import して実ルール集合を確認し、6 ルールを `.oxlintrc.json` に追加済み（`no-unused-expressions` のみ `correctness` カテゴリで既にカバー済みと確認したため追加不要）
- 「react / jsx-a11y / import の recommended セットについて oxlint 側の突き合わせが未実施」→ **正当な指摘。** `oxlint --print-config` による突き合わせは plugins 未指定で誤った結果を出したため、実際の違反コード（`target="_blank"` に `rel` 無し、存在しない named import 等）を書いて実行確認する方式に切り替えた。結果: react 5 件・jsx-a11y 33 件・`import/no-named-as-default(-member)` を明示追加。`import/named` は実装が named export を解決せず発火しないため「失われるルール」に追加し `tsc -b` の `TS2724` で代替されることを確認
- 「`react-hooks/exhaustive-deps` という表記はプラグイン統合の説明（`react` プラグインに統合）と矛盾する」→ 実測すると `react-hooks/exhaustive-deps` と `react/exhaustive-deps` は**同じルールを指す別名**で、どちらの表記でも設定・無効化が効く。診断メッセージの表示上は `react-hooks(...)` のラベルが使われる。検証手順の表記はそのままで問題ない
- 「`no-redeclare` が TS の関数オーバーロードで誤爆する」→ 誤爆しない（実測。オーバーロード宣言 3 つに対し指摘 0 件）
- 「lefthook の glob に含まれる `.scss`/`.less`/`.graphql`/`.mdx`/`.yaml` を oxfmt が処理できない可能性」→ すべて実行時エラーなく処理されることを確認
- 「oxfmt の `ignorePatterns` に `dist` が無く、ビルド成果物を書き換えうる」→ **正当な指摘。** `.oxfmtrc.json` にも `.oxlintrc.json` と同じ `{dist,build,public,node_modules}/**` を追加し、`dist/` が除外されることを確認
