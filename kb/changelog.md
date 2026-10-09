# Changelog — notable changes, newest first.

The ONLY place release history lives (don't duplicate it in other notes).
One line per change that future-you or the agent should know about.

## Unreleased
- Added `audit/BEHAVIORAL_SPECIFICATION.md` + `audit/behavioral-specification.json` +
  `audit/BEHAVIORAL_SPECIFICATION_COVERAGE.md` - complete 24-dimension behavioural extraction
  (rules, entry points, flows, state machines, temporals, concurrency, caches, event topology,
  config, doc drift) produced with the `business-logic-workflow-extractor` skill; the coverage
  map ties every named skill item to its result. Read-only: no code changed. See
  [[behavioral-spec]] for the summary, [[gotchas]] for the ZMQ-port drift it records.

## (date) - first version
- Initial knowledge base scaffolded.
