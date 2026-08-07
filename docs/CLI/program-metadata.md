---
id: program-metadata
title: Program Metadata
sidebar_position: 2
---

Meta

## Structure

```json
{
  "version": <int>,
  "timestamp": <int>,
  "base": <int>,
  "start_address": <int>,
  "insts": [
    {
      "extension": "<EXTENSION|x16|x32|x64|...>",
      "address": <int>,
      "bytes": [<int 0-255>, ...],
      "flow": "<CALL|JUMP|RETN|null>",
      "edges": [[<src int>, <dest int>], ...]
    },
    ...
  ]
}
```