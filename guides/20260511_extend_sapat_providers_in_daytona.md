---
title: 'Extend Sapat Providers in Daytona'
description:
  'Build and verify a new Sapat transcription provider in a reproducible
  Daytona workspace before opening an upstream PR.'
date: 2026-05-11
author: 'Jean-Claude Joanna'
tags: ['sapat', 'daytona', 'transcription']
---

# Extend Sapat Providers in Daytona

# Introduction

Sapat is a Python command-line tool for turning video files into transcripts. It
converts video to MP3 with FFmpeg, sends the audio to a selected transcription
provider, and writes a same-name `.txt` file next to the source video. Today,
the project supports OpenAI, Groq, and Azure OpenAI through the required
`--api` flag.

That makes Sapat a useful small project for AI engineers who want to learn how a
provider adapter should be added, tested, and documented. This guide shows a
repeatable Daytona workflow for adding another provider without turning the
change into guesswork. The concrete example is an AssemblyAI provider, but the
same checklist works for Deepgram, Speechmatics, or any service that accepts
uploaded audio and returns a transcript.

![Sapat provider extension workflow](assets/20260511_extend_sapat_providers_in_daytona_img1.png)

## TL;DR

- Create a Daytona workspace from the Sapat repository so the same branch can be
  tested from a clean environment.
- Map the current provider contract before writing code: CLI flag, environment
  variables, upload/transcribe behavior, correction behavior, and output file.
- Add the provider as a small [transcription provider adapter](../definitions/20260511_definition_transcription_provider_adapter.md),
  then prove it with mocked tests and a CLI smoke check.
- Open one focused upstream PR and reference it from your Daytona content PR so
  readers can inspect real code, not just a tutorial narrative.

## Prerequisites

You need:

- A GitHub account and a fork of `nkkko/sapat`.
- Daytona installed and authenticated.
- Python 3.10 or later for local validation.
- FFmpeg available in the workspace.
- API credentials for the provider you want to test live.

The live provider call is optional while developing. You can validate most of
the adapter contract with mocked HTTP tests, then run one credentialed smoke
test before you ask maintainers to review.

## Step 1: Create a Daytona Workspace

Start from the upstream repository or your fork. Daytona's current docs show
`daytona create` as the CLI entry point for creating a sandbox, and the Sapat
README also documents creating a workspace directly from the repository.

```bash
daytona create https://github.com/nkkko/sapat --code
```

Inside the workspace, create a feature branch:

```bash
git switch -c add-assemblyai-provider
```

Install the package in editable mode so the `sapat` command points at your
working tree:

```bash
python -m venv .venv
. .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install -e .
```

Confirm the CLI works before changing anything:

```bash
sapat --help
```

You should see the required `--api` option with the currently supported
providers.

## Step 2: Map the Existing Provider Contract

Before adding a provider, read the current code paths:

```bash
sed -n '1,180p' src/sapat/script.py
sed -n '1,220p' src/sapat/transcription/base.py
sed -n '1,220p' src/sapat/transcription/openai.py
sed -n '1,220p' src/sapat/transcription/groq.py
sed -n '1,220p' src/sapat/transcription/azure.py
```

The important contract is small:

- `src/sapat/script.py` selects a provider from the required `--api` flag.
- `TranscriptionBase.process_file()` converts the input video to MP3, calls
  `transcribe_audio()`, writes a `.txt` file, and removes the temporary MP3.
- Each provider reads credentials from `.env` using `python-dotenv`.
- Providers return either a text string or a dictionary with a `text` value.
- The optional `--correct` flow calls `generate_corrected_transcript()`.

Write the new provider to fit that shape. Avoid changing the base flow unless
the new API truly requires it.

## Step 3: Add the Provider Adapter

For AssemblyAI, the REST flow has three steps:

1. Upload the local MP3 file to `/v2/upload`.
2. Submit a transcript job to `/v2/transcript`.
3. Poll `/v2/transcript/{id}` until the status is `completed` or `error`.

Add a file such as `src/sapat/transcription/assemblyai.py`. Keep the
implementation direct and easy to review:

```python
class AssemblyAITranscription(TranscriptionBase):
    def transcribe_audio(self, audio_file: str, **kwargs):
        self._validate_audio_file(audio_file)
        upload_url = self._upload_audio(audio_file)
        transcript_id = self._submit_transcript(upload_url, **kwargs)
        transcript = self._poll_transcript(transcript_id)
        return {
            "text": transcript.get("text", ""),
            "id": transcript_id,
            "status": transcript.get("status"),
        }
```

Then wire the class into `src/sapat/script.py`:

```python
from .transcription.assemblyai import AssemblyAITranscription

@click.option(
    "--api",
    "-a",
    type=click.Choice(["openai", "groq", "azure", "assemblyai"]),
    required=True,
)
```

The branch should now expose `assemblyai` in `sapat --help`.

## Step 4: Document the Environment Variables

Add the provider's configuration to the README `.env` example:

```bash
ASSEMBLYAI_API_KEY=your_assemblyai_api_key_here
ASSEMBLYAI_API_ENDPOINT=https://api.assemblyai.com/v2
ASSEMBLYAI_POLL_INTERVAL_SECONDS=3
ASSEMBLYAI_TIMEOUT_SECONDS=600
```

Also add one usage example:

```bash
sapat product_demo.mp4 --quality M --language en --api assemblyai
```

This matters because maintainers should be able to see the new provider from
the CLI, the README, and the tests without reverse-engineering your branch.

## Step 5: Test Without Spending API Credits

Start with mocked tests. A good test proves that the adapter:

- uploads the converted audio file,
- submits the returned upload URL to the transcript endpoint,
- passes the selected language as `language_code`,
- returns the final transcript text, and
- raises a useful error when the provider reports a failed job.

Run the focused test first:

```bash
python -m unittest tests.test_assemblyai
```

Then run source compilation and the CLI smoke check:

```bash
python -m compileall src tests
sapat --help
git diff --check
```

These checks do not prove the provider account works, but they prove the local
adapter contract is sound.

## Step 6: Run One Credentialed Smoke Test

When you are ready to verify the live API, keep the sample tiny. Generate a
short video with a clear spoken sentence, or use a short internal test clip you
are allowed to upload to the provider.

Create a `.env` file in the workspace:

```bash
ASSEMBLYAI_API_KEY=your_key_here
ASSEMBLYAI_API_ENDPOINT=https://api.assemblyai.com/v2
ASSEMBLYAI_POLL_INTERVAL_SECONDS=3
ASSEMBLYAI_TIMEOUT_SECONDS=600
```

Run Sapat:

```bash
sapat sample.mp4 --quality M --language en --api assemblyai
```

Check the output:

```bash
cat sample.txt
```

If the transcript is empty, inspect each step in the provider adapter: upload
response, transcript submission response, final transcript status, and the
language code you sent.

## Step 7: Open a Focused Upstream PR

Keep the upstream provider PR small. A good PR body includes:

- the provider name and CLI flag,
- the new environment variables,
- the mocked test coverage,
- the exact validation commands, and
- one note if live provider validation was skipped because credentials are not
  available in the development environment.

For the AssemblyAI example, the PR should be shaped like this:

```md
## Summary
- add an AssemblyAI transcription backend available through `--api assemblyai`
- upload local audio to AssemblyAI, submit a transcript job, and poll until completion
- document AssemblyAI environment variables and usage
- add mocked unit tests for successful transcription and API error handling

## Validation
- `python -m compileall src tests`
- `python -m unittest tests.test_assemblyai`
- `sapat --help`
- `git diff --check`
```

After the provider PR is open, link it in your content PR. That gives the
article a real implementation trail and lets readers inspect the exact code.

## Troubleshooting

**Problem:** `sapat --help` does not show the new provider.

**Solution:** Confirm the new class is imported in `src/sapat/script.py` and
that `click.Choice()` includes the provider name.

**Problem:** the test fails before it reaches your mocked HTTP calls.

**Solution:** install the package in editable mode inside the Daytona workspace
with `python -m pip install -e .`, then run the test from the repository root.

**Problem:** the live smoke test times out.

**Solution:** increase `ASSEMBLYAI_TIMEOUT_SECONDS`, then check the provider's
job status response. Polling APIs often return a valid transcript ID before the
text is ready.

**Problem:** the output `.txt` file exists but is empty.

**Solution:** make sure the adapter returns either a string or a dictionary with
the `text` key. `TranscriptionBase.process_file()` writes that value to disk.

## Conclusion

Adding a transcription provider is not just an API call. It is a small
integration contract: CLI selection, workspace configuration, upload behavior,
result polling, transcript output, and tests that prove the flow is reviewable.

Daytona gives you a clean place to build and repeat that contract. Once the
provider PR is open, the guide becomes more useful too: readers can follow the
workflow and compare it against a real upstream implementation.

## References

- [Sapat repository](https://github.com/nkkko/sapat)
- [AssemblyAI transcription API](https://www.assemblyai.com/docs/api-reference/transcripts/submit)
- [AssemblyAI local-file transcription guide](https://www.assemblyai.com/docs/getting-started/transcribe-an-audio-file/)
- [OpenAI speech-to-text prompting guide](https://developers.openai.com/api/docs/guides/speech-to-text#prompting)
- [Daytona getting started docs](https://www.daytona.io/docs/getting-started)
- [Daytona environment configuration docs](https://www.daytona.io/docs/configuration)
- [AssemblyAI provider PR for Sapat](https://github.com/nibzard/sapat/pull/12)
