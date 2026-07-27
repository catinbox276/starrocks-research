# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

A Korean-language research documentation project (no code, no build/test commands) investigating whether open-source StarRocks supports (1) embedding/vector similarity search, (2) tokenizer-based full-text search, and (3) hybrid search combining both. Research date: 2026-07-24, pinned to **OSS StarRocks 4.1.1**.

## Document structure

- `README.md` — TL;DR conclusion table, version notes, and index of all documents
- `01`–`05` numbered topic files, each covering one research question and cross-linking the others by relative path
- `sources.md` — all cited sources (official docs, GitHub issues, vendor blogs)

## Conventions

- All prose is in Korean; SQL examples and technical terms stay in English.
- Every document is scoped to OSS 4.1.1. If content is version-dependent, state the version explicitly (e.g., vector index v3.4+, GIN inverted index v3.3+).
- Claims are backed by sources listed in `sources.md`; add new sources there when introducing new claims.
- When updating conclusions in a topic file, keep the README TL;DR table and the cross-references in other files consistent.

## Load-bearing research conclusions (keep consistent across all files)

The central theme of this research is distinguishing OSS capabilities from commercial/cloud distribution features. Do not blur this line when editing:

1. **BM25 scoring, engine-native RRF, weighted fusion, and learning-to-rank do NOT exist in OSS 4.1.1** — these are features of commercial distributions (e.g., Alibaba Cloud EMR Serverless StarRocks). In OSS, hybrid search means StarRocks handles recall only; fusion/re-ranking is assembled manually (SQL prefilter + vector ordering, or app-side RRF — see `04-fusion-and-oss-implementation.md`).
2. **Vector index is experimental** — requires FE config `enable_experimental_vector = true`, one vector index per table, HNSW/IVFPQ via `approx_*` functions.
3. **No Korean morphological tokenizer** — only `none`/`english`/`chinese`/`standard` parsers exist; proper Korean full-text search requires external morphological preprocessing (mecab-ko/Nori) injected into a separate tokenized column.
