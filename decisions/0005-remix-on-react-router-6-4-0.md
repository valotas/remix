# Layering Remix on top of React Router 6.4

Date: 2022-08-16

Status: proposed

## Context

Now that we're almost done [Remixing React Router][remixing-react-router] and will be shipping `react-router@6.4.0` shortly, it's time for us to start thinking about how we can layer Remix on top of the latest React Router. This will allow us to delete a _bunch_ of code from Remix for handling the Data APIs. This document aims to discuss the changes we foresee making and some potential iterative implementation approaches to avoid a big-bang merge.

## Decision

The high level approach is as follows

1.  Update `handleResourceRequest` to use `createStaticHandler` behind an ENV flag
    1.  Get unit and integration tests asserting both flows?
2.  Update `handleDataRequest` in the same manner
3.  Update `handleDocumentRequest` in the same manner
    1.  Confirm unit and integration tests are all passing
4.  Write new `RemixContext` data into `EntryContext` and remove old flow
5.  Deploy `@remix-run/server-runtime` changes once comfortable
6.  Handle `@remix-run/react` in a short-lived feature branch
    1.  server render without hydration (replace `EntryContext` with `RemixContext`)
    2.  client-side hydration
    3.  backwards compatible changes
    4.  deploy

## Details

There are 2 main areas where we have to make changes:

1. Handling server-side requests in `@remix-run/server-runtime` (mainly in the `server.ts` file)
2. Handling client-side hydration + routing in `@remix-run/react` (mainly in the `components.ts` file)

Since these are separated by the network chasm, we can actually implement these independent of one another for smaller merges, iterative development, and easier rollbacks should something go wrong.

### Do the server data-fetching migration first

There's two primary reasons it makes sense to handle the server-side data-fetching logic first:

1. It's a smaller surface area change since there's effectively only 1 new API to work with in `createStaticHandler`
2. It's easier to implement in a feature-flagged manner since we're on the server and bundle size is not a concern

We can do this on the server using the [strangler pattern][strangler-pattern] so that we can confirm the new approach is functionally equivalent to the old approach. Depending on how far we take it, we can assert this through unit tests, integration tests, as well as run-time feature flags if desired.

We can also split this into iterative approaches on the server too, and do `handleResourceRequest`, `handleDataRequest`, and `handleDocumentRequest` independently (either just implementation or implementation + release). Doing them in that order would also likely go from least to most complex.

#### Open Questions

- Plan to do this behind an ENV var to start?
  - Can we add unit test assertions?
  - Could we run unit and integration tests with/without the ENV var enabled?

#### Notes

- The `remixContext` sent through `entry.server.ts` will be altered in shape. We consider this an opaque API so not a breaking change.

#### Implementation approach

1. Use `createHierarchicalRoutes` to build RR `DataRouteObject` instances
   1. See `createStaticHandlerDataRoutes` in the `brophdawg11/rrr` branch
2. Create a static handler per-request using `unstable_createStaticHandler`
3. `handleResourceRequest`
   1. This one should be _really_ simple since it should just send back the raw `Response` from `queryRoute`
4. `handleDataRequest`
   1. This is only slightly more complicated than resource routes, as it needs to handle serializing errors and processing redirects into 204 Responses for the client
5. `handleDocumentRequest`
   1. This is the big one. It simplifies down pretty far, but has the biggest surface area for something to not quite match up
   2. We need to map query "errors" to Remix's definition of error/catch and bubble them upwards accordingly.
      1. For example, in a URL like `/a/b/c`, if C exports a a `CatchBoundary` but not an ErrorBoundary`, then it'll be represented in the `DataRouteObject`with`hasErrorBoundary=true`since the`@remix-run/router` doesn't distinguish
      2. If C's loader throws an error, the router will "catch" that at C's `errorElement`, but we then need to re-bubble that upwards to the nearest `ErrorBoundary`
      3. See `differentiateCatchVersusErrorBoundaries` in the `brophdawg11/rrr` branch
   3. New `RemixContext`
      1. `manifest`, `routeModules`, `staticHandlerContext`, `serverHandoffString`
      2. Create this alongside `EntryContext` assert the values match
   4. If we catch an error during render, we'll have tracked the boundaries on `staticHandlerContext` and can use `getStaticContextFromError` to get a new context for the second pass (note the need to re-call `differentiateCatchVersusErrorBoundaries`)

</details>

### Do the UI rendering layer second

The rendering layer in `@remix-run/react` is a bit more of a whole-sale replacement and comes with backwards-compatibility concerns, so it makes sense to do second. However, we can still do this iteratively, we just can't deploy iteratively since the SSR and client HTML need to stay synced.First, we can focus on getting the SSR document rendered properly without `<Scripts/>`. Then second we'll add in client-side hydration.

_TODO...Still in progress_

#### Backwards Compatibility Notes

- `useTransition` needs `submission` and `type` added
  - `<Form method="get">` no longer goes into a "submitting" state in `react-router-dom`
- `useFetcher` needs `type` added
- `unstable_shouldReload` replaced by `shouldRevalidate`
  - Can we use it if it's there but prefer `shouldRevalidate`?
- Distinction between error and catch boundaries
- `Request.signal` - continue to send separate `signal` param

#### Implementation approach

_TODO...Still in progress_

## Consequences

[remixing-react-router]: https://remix.run/blog/remixing-react-router
[strangler-pattern]: https://martinfowler.com/bliki/StranglerFigApplication.html