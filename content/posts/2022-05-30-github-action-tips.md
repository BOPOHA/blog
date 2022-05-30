---
title: "Github Action Tips"
date: 2022-05-30T10:00:25+02:00
draft: true
categories:
  - github
---

# how to debug events

```yaml
- name: debig GitHub context
  env:
    GITHUB_CONTEXT: ${{ toJson(github) }}
  run: |
    echo "$GITHUB_CONTEXT"
```
