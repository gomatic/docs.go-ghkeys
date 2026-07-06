---
title: go-ghkeys
---

**Fetch a GitHub user's SSH public keys from `https://github.com/<username>.keys` and convert the supported ones into [age](https://age-encryption.org) recipients.** Unsupported keys are skipped with a warning; failures carry a sentinel error matchable with `errors.Is`.

- **Source:** [gomatic/go-ghkeys](https://github.com/gomatic/go-ghkeys)
- **API reference:** [pkg.go.dev/github.com/gomatic/go-ghkeys](https://pkg.go.dev/github.com/gomatic/go-ghkeys)

## Install

```sh
go get github.com/gomatic/go-ghkeys
```

## Usage

`FetchRecipients` GETs the user's `.keys` listing through a caller-supplied `HTTPClient`, parses the authorized-keys body, and returns the parsed `age.Recipient` values. The HTTP transport is injected — any type with a `Do(*http.Request) (*http.Response, error)` method satisfies `HTTPClient`, so the standard `*http.Client` works directly.

```go
package main

import (
	"context"
	"fmt"
	"log"
	"net/http"

	"github.com/gomatic/go-ghkeys"
)

func main() {
	recipients, err := ghkeys.FetchRecipients(
		context.Background(),
		http.DefaultClient,
		ghkeys.Username("octocat"),
	)
	if err != nil {
		log.Fatal(err)
	}

	for _, r := range recipients {
		fmt.Printf("%#v\n", r)
	}
}
```

The returned `[]age.Recipient` can be handed straight to `age.Encrypt`.

### Errors

Every failure wraps a sentinel `errs.Const` from [gomatic/go-error](https://github.com/gomatic/go-error), recoverable with `errors.Is`:

- `ErrFetchKeys` — the endpoint could not be requested, returned a non-200 status, or its body could not be read.
- `ErrNoValidKeys` — the response contained no SSH public key that could be parsed into an age recipient.

```go
recipients, err := ghkeys.FetchRecipients(ctx, client, ghkeys.Username("octocat"))
if errors.Is(err, ghkeys.ErrNoValidKeys) {
	// the user has no age-compatible keys
}
```

### Options

`FetchRecipients` accepts type-based options. `Logger` routes the skip warning — emitted for each key age cannot represent — to a specific `*slog.Logger` instead of `slog.Default`:

```go
recipients, err := ghkeys.FetchRecipients(
	ctx,
	http.DefaultClient,
	ghkeys.Username("octocat"),
	ghkeys.Logger{Logger: slog.New(handler)},
)
```

## Design

- **Injected transport.** The GitHub call goes through the `HTTPClient` interface, so the network can be stubbed in tests and the endpoint stays fully covered.
- **Escaped request target.** The username is path-escaped into a single path segment, so a slash-, query-, or fragment-bearing value cannot rewrite the request target.
- **Bounded read.** The response body is read through an `io.LimitReader` capped at 1 MiB, so a compromised or MITM'd response cannot exhaust memory.
- **Skip, don't fail, on unsupported keys.** Keys `age` cannot represent (for example ECDSA) are logged and skipped; only an outright fetch or scan failure, or the total absence of a valid key, produces an error.
