# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

This is a collection of LLM rulesets — markdown documents that serve as comprehensive instruction sets for LLMs building or modifying applications. There is no application code, build system, or test suite. The repo contains only documentation files.

## Structure

Each `*_LLM.md` file is a self-contained specification for a feature domain (e.g., `AUTHENTICATION_REQUIREMENTS_LLM.md` covers a full Laravel auth system). These documents are designed to be fed to an LLM as context when building that feature in a project.

## Writing Guidelines

- Each ruleset should be implementable without ambiguity — include exact code snippets, database schemas, route definitions, and middleware configurations.
- Call out common pitfalls and "gotchas" explicitly (e.g., package setup steps that cause silent failures if missed).
- Specify the complete package dependency list with install commands at the top.
- Use consistent section structure: package deps, database schema, model changes, controllers, routes, frontend/JS, UI requirements.
- Include migration summaries and middleware stack summaries as reference sections at the end.
