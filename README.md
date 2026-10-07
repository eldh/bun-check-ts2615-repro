# `bun check`: false TS2615 that TypeScript 7 does not report

`bun check` reports a circularity error (TS2615) on code that `tsc` from TypeScript 7.0.2 accepts. The
[`bun check` docs](https://bun.com/docs/runtime/check) say that a difference from TypeScript 7 is a bug in Bun.

## Versions

- Bun `1.4.3-canary.1+bd599f5af` (the latest canary at the time of writing; `bun check` is not in a stable release yet)
- TypeScript `7.0.2` (`typescript@7.0.2` from npm)
- macOS (Darwin 25.6.0), arm64

## Steps

```sh
npm install
npx tsc -p .    # no output, exit code 0
bun check       # 1 error, exit code 1
```

## Expected

No errors, the same as `tsc`.

## Actual

```
repro.ts(22,1): error TS2615: Type of property 'assertHydrated' circularly references itself in mapped type '{ isHydrated: (() => this is {}) & (() => this is {}); assertHydrated: any; }'.
```

The type in the message shows `isHydrated` as an intersection with itself.

## Code

[`repro.ts`](./repro.ts):

```ts
type Hydrated<T> = T extends { x: {} }
  ? T
  : T & {
      [P in keyof T]: T[P] extends Lazy<infer U> ? Lazy<U> : T[P];
    };

declare class Lazy<T> {
  brand: T;
}

declare class Model {
  isHydrated(): this is {};
  assertHydrated(): this is Hydrated<this>;
}
declare class Collection<T> {
  hydrate(): Hydrated<T>;
}
class ChildCollection extends Collection<Model> {}

declare function relation<T extends Model>(ctor: new () => Collection<NoInfer<T>>): void;

relation(ChildCollection);
```

## The trigger: the `isHydrated` method

Without `isHydrated`, the two checkers agree. Both report TS2615 at the same place:

```diff
 declare class Model {
-  isHydrated(): this is {};
   assertHydrated(): this is Hydrated<this>;
 }
```

```
tsc:  repro.ts(22,1): error TS2615: Type of property 'assertHydrated' circularly references itself in mapped type '{ [P in keyof (Model & { assertHydrated: () => this is Model & ...; })]: ... }'.
bun:  repro.ts(22,1): error TS2615: Type of property 'assertHydrated' circularly references itself in mapped type '{ assertHydrated: any; }'.
```

With `isHydrated(): this is {}` added, TypeScript 7 does not report the error, but Bun still does. If `isHydrated`
returns `boolean` instead of a `this` type predicate, both checkers report the error again.

## Other parts that are necessary

Each change below removes the error from both checkers:

| Change | Result |
| --- | --- |
| Remove `NoInfer` (`Collection<T>`) | no errors |
| Give the type argument: `relation<Model>(ChildCollection)` | no errors |
| Remove the outer conditional from `Hydrated` (`T extends { x: {} } ? T : ...`) | no errors |
| Remove `T &` from `Hydrated` | no errors |
| Use `U` instead of `Lazy<U>` in the true branch | no errors |
| Make `Lazy` non-generic | no errors |
| Replace `class ChildCollection` with `declare const ChildCollection: new () => Collection<Model>` | no errors |
| Pass `Collection<Model>` directly instead of `ChildCollection` | no errors |

The first two rows show that `T` must have no inference candidates, so that it falls back to its constraint `Model`.

## Things that do not matter

- Compiler options. It reproduces with `"compilerOptions": {}`, with `strict` on or off, and with no tsconfig
  (`bun check repro.ts`).
- Threads. It reproduces with `bun check --threads=1`.

## Origin

Found while trying `bun check` on a large TypeScript monorepo, where it otherwise reported the same errors as `tsc`
7.0.2. The original code used `NoInfer` in the options of a decorator factory, and a `Hydrated<T>` helper type over
model classes. This file was reduced automatically from that code. Every reduction step required that Bun report
exactly this one error and that `tsc` report none.
