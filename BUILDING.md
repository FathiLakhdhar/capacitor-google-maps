# Building the plugin (main-7.x)

How to build this fork of `@capacitor/google-maps` and use it from the
HereItIs app (`../HereItIs`).

## Prerequisites

| Tool    | Version used    |
| ------- | --------------- |
| Node    | >= 20 (22.x OK) |
| pnpm    | >= 8 (10.x OK)  |
| JDK     | 17+ (21 OK)     |
| Gradle  | 8.13 (wrapper)  |
| Xcode   | 16+ (26.x OK)   |

The Android build uses AGP 8.13 with compileSdk/targetSdk 36. You need
Android SDK Platform 36 installed.

## 1. Install dependencies

From the repo root:

```sh
git checkout main-7.x
pnpm install
```

## 2. Build the web / TypeScript bundle

```sh
cd plugin
pnpm build
```

This runs, in order:

1. `clean`: deletes `dist/`.
2. `docgen`: regenerates the API section of `plugin/README.md` and writes
   `dist/docs.json`. Run it whenever you change the public API in `src/definitions.ts` or `src/map.ts`,
   and commit the README diff.
3. `tsc`, then `rollup`: produces `dist/esm`, `dist/plugin.js` and `dist/plugin.cjs.js`.
4. `downlevel-dts`: produces `dist/typings/ts_old`.

You can ignore the rollup warning `Missing global variable name ... @googlemaps/markerclusterer`.

`pnpm build` from the repo root does the same, then copies
`plugin/README.md` to the root `README.md`.

For TypeScript-only iteration you can use `pnpm watch`. It doesn't run rollup, so run
`pnpm build` before syncing into an app.

## 3. Verify native code

Run these from `plugin/`:

```sh
pnpm verify:android   # cd android && ./gradlew clean build test
pnpm verify:ios       # xcodebuild build -scheme CapacitorGoogleMaps -destination generic/platform=iOS
pnpm verify           # ios + android + web
```

The Android module compiles against `node_modules/@capacitor/android`, so
`pnpm install` must have run first.

## 4. Use it in HereItIs

HereItIs references the plugin by local path, so it always picks up the code in
this checkout:

- `HereItIs/package.json`: `"@capacitor/google-maps": "../capacitor-google-maps/plugin"`
  (npm symlinks it into `node_modules`)
- `HereItIs/ios/App/Podfile`: `pod 'CapacitorGoogleMaps', :path => '../../../capacitor-google-maps/plugin'`

After changing the plugin:

```sh
# 1. rebuild the plugin
cd ~/devs/capacitor-google-maps/plugin && pnpm build

# 2. sync into the app
cd ~/devs/HereItIs
npm install            # only needed the first time / after dependency changes
npm run build:dev      # or build:prod
npx cap sync           # copies web assets + updates native projects (runs pod install on iOS)
```

Then open and run the native project with `npx cap open android` or `npx cap open ios`.

Native changes (Kotlin/Swift) are compiled by the app's own Android Studio or
Xcode build, straight from this folder. You don't need a separate build step for
them, but `npx cap sync` is still required if you add or remove a plugin.

## 5. Package a tarball (optional)

To install into an app without a path link:

```sh
cd plugin
pnpm pack-local        # -> plugin/capacitor-google-maps.tgz
# in the app:
npm install /path/to/capacitor-google-maps.tgz
```

## Troubleshooting

- **`Minimum supported Gradle version is 8.13`**: the Gradle wrapper must match AGP.
  `plugin/android/gradle/wrapper/gradle-wrapper.properties` should point to
  `gradle-8.13-all.zip`.
- **README API docs describe GeoJSON types (e.g. `Point` with RFC 7946 text)**:
  docgen picked up the interfaces from `@types/geojson`. Always run docgen through
  `pnpm build` or `pnpm docgen`, because `scripts/docgen_updates.js` hides
  `@types/geojson` while it runs.
- **pnpm warns `Ignored build scripts: ...`**: this is harmless for building the
  plugin.
