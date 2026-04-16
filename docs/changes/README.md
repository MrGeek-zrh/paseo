# Change Tracking Documents

## Purpose

This directory stores factual records for behavior changes, incident analysis, deployment notes, and validation history.

Design intent belongs in `docs/plans/`. Change tracking belongs here when a reader needs to answer what changed, why it changed, what was verified, and what state was deployed.

## When to add a document

Add a document under `docs/changes/` when a change includes at least one of these conditions:

- a production or user-facing incident
- a host-specific debugging session that changes product behavior
- a deployment that needs traceable evidence
- a compatibility-sensitive fix that should be easy to audit later

## File naming

Use this filename format:

`YYYY-MM-DD-short-topic.md`

The date should match the day the record is written or materially updated.

## Required sections

Each change tracking document should contain these sections in this order:

1. `Summary`
2. `Background`
3. `Evidence`
4. `Behavior Change`
5. `Compatibility`
6. `Code Index`
7. `Verification`
8. `Deployment Status`
9. `Rollback References`
10. `Known Limits`

## Writing rules

Write only facts that are known.

Record exact versions, commit hashes, branch names, commands, host names, file paths, and visible user messages when they are available.

If a value is unknown or was not verified, state that plainly. Do not fill gaps with assumptions.

If the change also has a design document, link the related file in `docs/plans/`.

## Scope

These documents are for tracing real changes over time. They are not product specs and they are not release notes.
