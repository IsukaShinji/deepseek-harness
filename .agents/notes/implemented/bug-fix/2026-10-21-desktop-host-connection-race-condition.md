
# Desktop Host Connection Race Condition Fix

## Problem

During the startup phase of the DSH Desktop application (e.g., `0.2.1-rc.2`), the desktop host process occasionally crashes with a `TypeError: Cannot read properties of undefined (reading 'authenticatedUrl')`. This prevents the `type: "ready"` IPC message from being sent to the main process, resulting in the UI hanging indefinitely on a blank screen while the tray icon is present.

## Root Cause

In `apps/desktop-host/src/index.ts`, after `await application` and the `desktopOffice` plugin initialization, the context (`ctx`) is accessed synchronously to generate the `authenticatedUrl`.

However, because plugin and service registrations (like `connection` and `webServer`) operate asynchronously, there is no strict guarantee that `ctx.connection` is populated at this exact tick, especially under heavy disk I/O or CPU load. Attempting to synchronously read `.authenticatedUrl` from an `undefined` connection object causes the fatal crash.

## Resolution

Wrapped the URL generation and the `type: 'ready'` IPC emission inside a `ctx.inject(['connection', 'webServer'], ...)` block. This aligns with the framework's dependency injection design, ensuring that the host process explicitly waits for both the connection and web server capabilities to be fully resolved and mounted before attempting to compute the authenticated URL and signaling the renderer to boot.
