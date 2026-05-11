---
title: 'Transcription Provider Adapter'
description:
  'A small integration layer that connects one speech-to-text provider to a
  common transcription workflow.'
date: 2026-05-11
author: 'Jean-Claude Joanna'
---

# Transcription Provider Adapter

## Definition

A transcription provider adapter is a small integration layer that lets a
speech-to-text service fit into a shared transcription workflow.

## Context and Usage

In a tool like Sapat, the adapter hides provider-specific details such as API
keys, upload endpoints, polling, response formats, and error handling. The rest
of the application can keep using the same CLI option, file conversion flow, and
transcript output behavior even when a new provider is added.
