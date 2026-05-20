# Idea: [Investigation] Create improvement tasks for ohttp_dart (AW-2865)

## Metadata

- **Jira ticket:** AW-2865
- **Type:** Task
- **Status:** In Progress
- **Priority:** P2: High
- **Reporter:** Anatoli Gonchar
- **Assignee:** Dmitry Seloustev
- **Labels:** _(none)_
- **Components:** App
- **Fix versions:** _(none)_
- **Linked issues:** - blocks AW-2857

## Summary

An investigation of the current state of the `ohttp_dart` package is needed, along with preparing a list of tasks to bring the package to production-ready status for potential use in a non-custodial crypto wallet. Within the scope of the investigation, the goal is not to implement improvements, but to identify risks and form subsequent engineering tasks.

## Motivation

## Task Description

An investigation of the current state of the `ohttp_dart` package is needed, along with preparing a list of tasks that need to be completed to bring the package closer to production-ready status for potential use in a non-custodial crypto wallet.

Within the scope of the investigation, the goal is not to implement improvements, but to identify risks and form subsequent engineering tasks.

## Technical Details

Investigation vectors:

- Cryptographic correctness of the HPKE/OHTTP implementation. Compatibility of the implementation with RFC 9180, RFC 9292, and RFC 9458
- Interoperability with real OHTTP gateway/server implementations
- Robustness of parsers against malformed input and potential DoS scenarios
- Privacy risks when used in a crypto wallet
- Network client reliability: timeout, retry, cancellation, error handling
- OHTTP KeyConfig management policy: caching, updates, downgrade risk
- Test suite: negative tests, fuzz/property tests, end-to-end tests
- Documentation, threat model, and usage limitations

| **Platform** | All (default) |
|---|---|
| **URLs** | ohttp_dart: https://github.com/AdguardTeam/ohttp_dart |
| **Figma** | |
| **Notion** | https://www.notion.so/adguard/OHTTP-Dart-360aa56b773080619205ff76a2a360f9 |

## Acceptance Criteria

As a result of the investigation, a set of Jira tasks should be prepared with a brief description, priority, and estimated urgency. Each task should be specific enough to be taken into work separately.

- A review of the current state of the `ohttp_dart` package has been conducted
- Main improvement directions have been identified
- A list of follow-up tasks has been formed
- For each follow-up task, the expected priority is indicated
- Blockers for production use in a crypto wallet are separately noted
- Changes to the `ohttp_dart` package are not implied within this task

## Additional

- Testing is not applicable

## Scope

### In Scope

_(to be filled)_

### Out of Scope

_(to be filled)_

## Open Questions

_(to be filled)_

## Discussion

_No comments on the Jira ticket._
