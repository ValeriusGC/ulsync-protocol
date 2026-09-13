# Changelog

**Created:** 2026-08-26 10:26:24 +0500  
**Updated:** 2026-09-13 18:14:38 +0300  
**Version:** 2  
**Document type:** changelog

All notable changes to this project are documented in this file.

The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

### Added

- Envelope format, last-write-wins conflict resolution, push / pull / health endpoints, live feed, limits, compatibility rules, and golden fixtures for protocol v1.
- Requirement that `source_id` is unique per installation (§1.4). This clarifies a client duty; the envelope format is unchanged.
