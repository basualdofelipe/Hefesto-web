---
created: 2026-03-29T00:38:32.444Z
title: Shareable scenarios between investors with is_public flag
area: database
files:
  - nemea-back/src/scenarios/entities/scenario.entity.ts
---

## Problem

Scenarios are currently user-scoped (each investor sees only their own). But the user wants investors to be able to share their simulation scenarios with other investors, so everyone can see and discuss the same pricing plans.

## Solution

Add `isPublic: boolean` (default false) column to the `scenarios` table. When an investor marks a scenario as public, other users can view it (read-only) in their scenarios list. Only the owner can edit or delete a public scenario.

UI: toggle or checkbox "Compartir con otros inversores" in the scenario editor. Public scenarios show with a badge "(compartido por [user])" in other users' lists.

Backend: GET /scenarios returns own scenarios + public scenarios from other users. PUT/DELETE still restricted to owner only.
