# Review resolution addendum

- PR: #9
- Base: `main`
- Resolution scope: the three existing review threads listed below.
- This document is a normative design addendum. It records the accepted resolution contract; it does not claim that implementation or tests have already run.
- Bot review is not retriggered.

## PRRT_kwDOTN3-Jc6OYqqC — Input Monitoring permission purpose

Problem: the app requests `CGRequestListenEventAccess()` without an explicit privacy-purpose declaration in the app bundle.

Resolution: the macOS app skeleton and release configuration MUST include a non-empty `NSInputMonitoringUsageDescription` in `Info.plist` before requesting global keyboard event access. The onboarding path MUST present the purpose and verify that the key is present before the request. A missing or empty purpose string is a release-blocking configuration error.

Focused verification before resolving this thread: inspect the generated app bundle and plist; assert the key is present and non-empty; exercise onboarding at least once with the request path and once with the missing-key failure path.

## PRRT_kwDOTN3-Jc6OYqqD — macOS 13 display-link compatibility

Problem: `CADisplayLink` is not a macOS 13-compatible choice for this desktop target.

Resolution: the macOS 13 path MUST NOT depend on `CADisplayLink`. Use a macOS-compatible `Timer`, Core Animation mechanism, or `CVDisplayLink`, chosen consistently with the rendering target. The timing mechanism MUST preserve the existing animation and keyboard-hook behavior on macOS 13 and newer.

Focused verification before resolving this thread: compile the target with the minimum supported macOS SDK; scan shipped source for the incompatible API in the macOS path; run an animation smoke check on macOS 13-compatible and newer environments.

## PRRT_kwDOTN3-Jc6OYqqE — privacy scan scope

Problem: a repository-wide grep reports privacy-sensitive tokens that occur in documentation examples, not executable behavior.

Resolution: the privacy gate MUST scan executable source, generated build artifacts, entitlements, and release configuration. Documentation and historical design examples MUST be excluded by explicit path rules or recorded allowlists. The gate MUST fail on forbidden symbols in executable sources while not failing solely on the documented tokens in this addendum or other non-executable docs.

Focused verification before resolving this thread: run the scoped scan against source/build paths and a separate audit that proves documented allowlisted matches are not silently excluded from executable paths.

## Verification status

The checks above are required acceptance criteria for implementation. This addendum intentionally reports no test result and no implementation-complete status.