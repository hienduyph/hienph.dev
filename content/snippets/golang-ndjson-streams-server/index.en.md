---
title: "Golang Ndjson Streams Server"
date: 2021-12-25T14:17:47+07:00
draft: true
tags: ['go', 'json']
---

Some time we got a large json data so clients can not buffer and read at once.

So New Lines JSON is the rescue, this a POC version of golang that response a streams of lines into client.

```go
package main

import (
	"context"
	"net/http"
	"os/signal"
	"syscall"
	"time"

	"github.com/go-chi/chi"
	"github.com/go-chi/chi/middleware"
	"github.com/hienduyph/goss/httpx"
	"github.com/hienduyph/goss/jsonx"
)

// let's generate some fake data
func sampleStream() <-chan []byte {
	out := make(chan []byte, 10)
	go func() {
		for i := 0; i < 10; i++ {
			d := map[string]interface{}{"hello": i}
			buf, _ := jsonx.Marshal(d)
			out <- buf
			time.Sleep(200 * time.Millisecond)
		}
		close(out)
	}()
	return out
}

func streamJSON(w http.ResponseWriter, r *http.Request) {
	flusher, ok := w.(http.Flusher)
	if !ok {
		w.WriteHeader(500)
		w.Write([]byte("Server unsupport flush"))
		return
	}

	w.Header().Set("Content-Type", "application/octet-stream")
	w.Header().Set("Connection", "Keep-Alive")
	w.Header().Set("X-Content-Type-Options", "nosniff")
	for buf := range sampleStream() {
		w.Write(buf)
		w.Write([]byte("\n"))
		flusher.Flush()
	}
}

func main() {
	ctx, done := signal.NotifyContext(context.Background(), syscall.SIGINT, syscall.SIGTERM)
	defer done()

	r := chi.NewRouter()
	r.Use(middleware.Logger)
	r.Get("/", streamJSON)
	s := &http.Server{
		Addr:    ":8080",
		Handler: r,
	}
	httpx.Run(ctx, s)
}

```

Let's make some calls.

```bash
curl -v http://127.0.0.1:8000

# or pull the data into files

curl -v http://127.0.0.1:8080 > data.json
```
