---
title: "Update Go Mod"
date: 2021-09-26T15:54:47+07:00
tags: ['go']
---

Auto update direct dependencies in `go.mod` into latest stable version.

```bash
go list -f '{{if not (or .Main .Indirect)}}{{.Path}}{{end}}' -m all\
  | xargs -n 1 sh -c 'echo $0; go get -d $0 || true'
```

