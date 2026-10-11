<!-- Template: copy to your project or place a copy at projects/<name>/threat_model.md. Free text; the scanner reads it before it starts. -->
# Threat model

## What this project does and where untrusted input enters
- We get called as a REST web api from various OS setups with untrusted package data.

## Components that matter most / least
- Authentication via JWT for all writing access
- public anonymous access for all reading actions

## How to exercise it
- most parts covered by unit tests

## How you rate severity
- Anything written without authorization should be a high level failure.

## Anything to leave alone
- Style and complexity warnings can be ignored.
