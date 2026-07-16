---
layout: post
title: "Guiding Principles"
date: 2026-07-16
---


---

This exists so that anyone touching our codebase, now or in a year, when
none of us remember why a line was written the way it was, can make calls
that fit with how the rest of it was built. It isn't exhaustive, and it
isn't a rulebook you follow blindly. It covers what matters most given what
we're building: something that handles database credentials, runs inside
other people's production infrastructure, and has to be trusted by teams
who are taking a real risk wiring it into their pipelines.

Most of what follows exists because we already got it wrong once. Where
that's the case, the reasoning is included, because remembering why a rule
exists is what keeps someone from quietly working around it six months
from now.

There are two parts here, and they're doing different jobs. Part I is about
writing good code inside the system we already have. It's tactical, and it
will keep changing as the codebase grows. Part II is shorter, on purpose. It
covers the handful of moments where a decision is hard to undo, and it
should barely ever need to change at all.

This sits next to two other documents, and it's worth being clear on who
owns what, so nobody has to guess later:

- Our FRD covers what the product promises users: read-only access,
  nothing gets deleted, people come before tooling.
- Our roadmap covers how we sequence and pace the work: ship small, go
  deep before wide.
- This document covers how we write code, and how we decide on the things
  that are expensive to take back.

If something ever seems to pull against one of the other two, that's worth
saying out loud rather than just picking whichever is convenient at the
time.

---

# Part I: How We Build

## 1. Simplicity over cleverness

Given two implementations of the same thing, the simpler one is usually
right. Data models, API shapes, package layout, individual functions, all
of it. Code that's easy to read is easy to review, easy to debug at 11pm,
and easy to change without breaking something three files away.

Complexity isn't free. Every abstraction and every clever pattern is
something the next person has to load into their head before they can do
anything. That cost doesn't stay flat. It compounds across contributors and
across time. A clever solution someone wrote in an afternoon can cost a
stranger a full day to safely touch eighteen months later, because now
they're reverse-engineering intent as well as reading code.

If something feels complicated, the useful question is whether the
complexity is coming from the actual problem or from how it was solved. A
breach diff that has to handle five real schema-change cases should read
like it handles five cases, not like a general-purpose diffing engine that
happens to be pressed into service for five of them today.

## 2. Explicit over implicit

Go is built to make things explicit. Work with that instead of around it.

Functions take what they need as parameters (a database handle, a config
value, a secret) rather than reaching for a package-level global or reading
the environment themselves. It's a small habit with a real payoff: the
dependency is visible right at the call site, and the function is testable
without any setup at all.

```go
func ClassifyChange(change ProposedChange, rules ClassificationRules) Classification
```

Identity works the same way. Middleware that resolves who's making a
request (an authenticated user, the org an API key belongs to) attaches
that to the request context, where a downstream handler can actually read
it. It never goes on the response, because a response header goes back to
the caller, and now you've handed the client a UUID they had no reason to
see. Context keys for this live in one shared file per package, not
redefined independently wherever someone happens to need one that week.

```go
ctx := context.WithValue(r.Context(), contextKeyOrgID, orgID)
next.ServeHTTP(w, r.WithContext(ctx))
```

Same instinct applies to queries. If a proposal listing is only supposed to
return the open ones by default, that's a `WHERE` clause, not a loop that
fetches everything and throws rows away afterward.

## 3. Errors are information

Go treats errors as values, not exceptions to be thrown and caught
somewhere far away, and that should show in how this codebase handles them.
Deal with an error where it happens. Don't discard it, don't quietly turn
it into a panic unless the situation genuinely warrants that, and don't
hand it up the stack with nothing added.

```go
return fmt.Errorf("activating contract %s: %w", contractID, err)
```

Wrapping like this means the call chain is readable later without having to
step through every intermediate function in a debugger.

What the client sees and what the log sees are two different things. The
client gets something clean and deliberate. The log gets the full internal
picture. A raw Go error string should never make it into an API response.
`pq: duplicate key value violates unique constraint "proposals_contract_id_open_idx"`
tells an attacker something about your schema and tells a legitimate caller
nothing useful. What they should get instead is `{"error": "a proposal is
already open on this contract"}`.

## 4. Configuration lives at the boundary

Anything that changes between environments (database URLs, secrets, feature
flags, SMTP credentials, how often breach detection runs) comes from
environment variables, loaded once at startup into a config struct. None of
it is hardcoded, and that includes tests, comments, and anything labeled
"temporary."

A secret sitting in source code stopped being a secret the moment it was
committed. It's in version control, on every contributor's laptop, and in
whatever CI system happens to print the diff. So functions that need a
secret take it as a parameter, the same way they'd take any other
dependency. The caller decides where it comes from, which also means the
tests can pass in a value that's obviously fake rather than depending on
whatever's hardcoded.

The config struct gets built once and handed down explicitly to whatever
needs it. A service that reads `os.Getenv` on its own has quietly hidden
one of its dependencies from anyone reading its constructor.

If something required is missing, the application should refuse to start,
loudly, with a message that says what's missing. Finding out about a
missing secret three days into a deployment, mid-request, is a much worse
way to learn the same thing.

## 5. Security gets designed in, not added on

A handful of properties have to hold everywhere in this codebase,
consistently, and they don't get an exception for convenience.

Revocation works the moment it happens. If an API key is revoked or a user
deactivated, the very next request with those credentials fails. No cache
holding onto a validity decision from a minute ago, no grace period. Every
authorization check hits the database directly for exactly this reason.

Credentials don't show up in logs. Not a plaintext key, not a session
token, not a decrypted password, not even a fragment of one. If a library's
own error text happens to include the input that caused it, strip that
before it gets logged.

Encryption uses the right primitive, used correctly: AES-256-GCM with a
fresh random nonce every time, so encrypting the same value twice produces
two different ciphertexts. Password and key hashing is bcrypt. Fast hashing
of values that are already random, like response tokens, is SHA-256.
Anything that needs to be unpredictable comes from `crypto/rand`, never
`math/rand`. The latter is deterministic if you know the seed, which makes
it useless for anything security-adjacent.

Where a token is meant to be single-use, it's actually enforced that way. A
consumer's proposal-response link gets invalidated the instant it's used,
because the FRD is explicit that this link is how someone without an
account gets to participate. It needs to hold up entirely on its own.

Signing tokens carry the standard claims: issued time, expiry, issuer,
subject. And the algorithm gets checked before the verification key is ever
handed back to the parser. Skip that step and you've left the door open for
an `alg: none` forgery.

```go
func(token *jwt.Token) (interface{}, error) {
    if _, ok := token.Method.(*jwt.SigningMethodHMAC); !ok {
        return nil, fmt.Errorf("unexpected signing method: %v", token.Header["alg"])
    }
    return []byte(secret), nil
}
```

And any code touching authentication, authorization, or credentials gets a
second reviewer before it merges. That's not a judgment on whoever wrote
it. A second set of eyes on cryptographic code is just how this works,
everywhere, regardless of who's on the team.

## 6. The database is the record, not the application's memory

If a fact matters, it lives in Postgres. Application memory doesn't count,
and nothing in the request lifecycle should be written as though it does.

A revoked key or a deprecated contract keeps its row. Status changes,
nothing gets deleted. Hard-deleting it throws away exactly the information
an incident review needs later: when it was created, who created it, what
it was used for. The audit trail goes further than that: no delete, no
update, ever, enforced at the database privilege level rather than trusted
to application code that a bug (or a compromise) could bypass. If the
application itself were ever breached, the audit trail should still hold.

Validity filtering happens in the query. A key that's revoked or expired
shouldn't come back from the database at all:

```sql
SELECT * FROM api_keys
WHERE key_prefix = $1
  AND is_active = true
  AND (expires_at IS NULL OR expires_at > NOW());
```

Leave off the last two lines and a revoked key keeps authenticating,
because nothing downstream is checking `is_active` again.

Timestamps are `TIMESTAMPTZ`. Bare `TIMESTAMP` looks fine until servers
span timezones, or data gets exported, or two systems compare timestamps
against each other, and then it's silently wrong instead of loudly wrong,
which is worse.

## 7. Background work doesn't block, and doesn't take the process with it

If something doesn't affect the response the caller is waiting on, it
shouldn't delay that response. Updating `last_used_at` after a successful
auth, sending a breach alert, writing a secondary log line: these run in a
goroutine after the response has already gone out.

```go
go func() {
    defer func() {
        if r := recover(); r != nil {
            slog.Error("panic recovered in background task", "panic", r)
        }
    }()

    if err := queries.UpdateAPIKeyLastUsed(context.Background(), keyID); err != nil {
        slog.Error("failed to update api key last used", "error", err)
    }
}()
```

Two things about this pattern are easy to get wrong. The context has to be
`context.Background()`, not the request's. The request context gets
cancelled the moment the response is written, so a goroutine inheriting it
may find the work already dead before it starts. And every background
goroutine needs that deferred recover. An unhandled panic in a goroutine
takes down the whole process, not just the one request. Every in-flight
request on that instance goes with it, and in a container, the whole thing
restarts. "Fire and forget" means you don't wait for the result. It doesn't
mean failures vanish. Log them.

## 8. Test what actually matters

A test should verify behavior, not mirror the implementation underneath it.
One that breaks because an internal function got renamed isn't testing
anything useful. One that catches a revoked key still authenticating is.

Every acceptance criterion on a Trello card is a test case. If a card says
a schema change inside the grace period shouldn't raise a breach, there's a
test proving exactly that, not something adjacent to it.

Unit tests lean on fakes and interfaces rather than a real database. The
interface sits on the handler, a fake implements it, the test passes the
fake in. Fast, no external dependencies. Integration tests go the other
direction entirely: real Postgres, via the testcontainers helper in
`internal/testutil/db.go`, which spins up a container, runs the migrations,
and hands back a live connection. Mocking the database in an integration
test defeats the point. You're specifically trying to catch what only shows
up against the real thing, like a constraint violation or actual
transaction behavior.

Table-driven tests once there are more than two input variants worth
covering:

```go
tests := []struct {
    name     string
    change   ProposedChange
    expected Classification
}{
    {"add nullable column", addNullableColumnChange, Additive},
    {"drop column", dropColumnChange, Breaking},
    {"widen varchar", widenVarcharChange, Additive},
    {"narrow varchar", narrowVarcharChange, Breaking},
}
for _, tt := range tests {
    t.Run(tt.name, func(t *testing.T) {
        got := Classify(tt.change)
        if got != tt.expected {
            t.Errorf("got %v, want %v", got, tt.expected)
        }
    })
}
```

## 9. Logging that's actually useful later

Structured logging with `log/slog`, everywhere. No `fmt.Printf`, no
`fmt.Println`, no bare `log.Printf`. Those write to stdout outside the
structured pipeline, and the moment logs get aggregated anywhere, that
output becomes noise nobody can search.

```go
slog.Error("breach detection failed",
    "contract_id", contractID,
    "org_id", orgID,
    "error", err,
)
```

Pick the level based on what someone reading it actually needs to do about
it. Error means something failed that shouldn't have and a person should
look. Warn means something unexpected happened but the system handled it:
an expired token hitting login, a bounced email, a connectivity check that
came back negative on schedule. Info is a lifecycle event worth knowing
happened: the app started, the scheduler started, a migration ran. Debug is
for when you're actually in the middle of debugging and don't want this
running by default otherwise.

Key material never appears in a log line, full stop. Not a key, not a
token, not a credential, not a fragment of one, and if a library's own
error text happens to carry the input along with it, strip that before
logging. And don't log an error and also return it from the same function.
Pick one, or the caller ends up logging the same failure a second time
right after you already did.

## 10. The API is a contract, not a suggestion

Other things depend on this surface: the CLI, the web UI, whatever gets
built against it later. Changing it carelessly breaks all of them at once,
usually without warning.

A list endpoint returns an array. Not `null`, and not some different shape
because the list happened to be empty this time. HTTP status codes mean
what they're supposed to mean: 404 when something doesn't exist or belongs
to a different org, 409 when a request conflicts with existing state, 422
when the request is well-formed but semantically wrong, 401 for missing
credentials, 403 for credentials that just aren't enough. And an
authentication failure says "unauthorized." It doesn't specify whether the
key was missing, malformed, expired, or revoked, because each of those is
one more piece of information an attacker can use to narrow things down.

## 11. One thing, clearly named

A package does one thing. A function does one thing. Names describe what
something holds or does, not the implementation detail of how it happens
to do it today. `notification` sends notifications. It doesn't also carry
audit-log writing logic just because a notification happens to trigger one
downstream. `VerifyAPIKey` verifies a key; it doesn't quietly bump
`last_used_at` as a side effect while it's in there. Once concerns like
that get mixed into one function, both halves get harder to test and
harder to change independently later.

Go acronym conventions hold: `UserID`, not `UserId`. `HTTP`, not `Http`.
The JSON tag stays snake_case for the wire format regardless of what the Go
field is called. Both are correct at the same time, for different
audiences.

Exported functions get a godoc comment describing what they do, not a
narrated walkthrough of how.

## 12. Migrations don't get rewritten, only extended

Every schema change is a numbered migration with a matching `.up.sql` and
`.down.sql` that fully reverses it. Once a migration has run anywhere, it
doesn't get edited. If it turns out to be wrong, a new migration fixes it.
Editing an already-applied file means the file and the actual database are
now describing two different histories, and there's no clean way back from
that.

A migration that drops a column or table says why in a comment, since the
removal is permanent and the reasoning is worth having later. And down
migrations get tested, not assumed to work. Running up, then down, then up
again should land on the exact same schema as running up once, and that
gets checked before the migration merges, not after something breaks.

## 13. Stateless, because it has to scale sideways

Ratify runs as a containerized process expected to run more than one
replica at once, and that shapes what kind of state is allowed where.

Nothing security-relevant gets cached locally: not session validity, not
key status, not org membership. A local cache like that creates a
split-brain the moment there's a second replica, where one instance is
still honoring a key that got revoked five seconds ago on another. Every
authorization check goes to Postgres directly, every time, which is the
only thing that makes "revocation works immediately" from section 5 true
once there's more than one instance running.

---

# Part II: How We Decide

Part I is about writing good code inside decisions that are already made.
This part covers something narrower and higher-stakes: the moments where
the decision itself is the risk. A new core dependency, replacing a
subsystem that already works, a different language for a component, a
different deployment model.

A bad call in Part I costs a bug fix. A bad call here can cost months, and
it can cost the trust of people who adopted the tool in good faith. The
JavaScript runtime Bun's internal rewrite of a core component from Zig to
Rust is a recent, public example of what that looks like in practice. It
was a migration undertaken without much of a reversible path back out, it
ended up consuming far more time than planned, it destabilized something
that had been reliable, and it cost real community trust along the way.
Nothing about that was a code-quality problem. It was a decision that got
made and executed without the kind of guardrails this section is trying to
put in place.

These six apply regardless of deadline pressure, team size at the time, or
how sure any one of us feels about a given idea.

### I. No one person makes an irreversible call alone

A new core dependency, replacing a subsystem, a real architecture change:
these need a short written case and agreement from the other two people on
the team before any code gets written, not a heads-up once a branch already
exists. Three paragraphs is enough. What matters is that the reasoning
exists somewhere other than one person's head, and that agreement happened
while it was still cheap to say no.

### II. Boring technology stays the default

Our tech stack decisions already made this call once, with reasons attached, and
that decision doesn't get reopened casually. When something new looks
appealing, the actual bar isn't "is this good." Plenty of things are good.
The bar is whether what we already have has a specific, documented failure
that the new thing would fix. "Newer" doesn't clear that bar. Neither does
"I'd enjoy using it," even though that one's honest at least. Every swap
resets years of some other tool's accumulated handling of edge cases back
to zero, and the boring choice was made deliberately, not by default.

### III. Big changes prove themselves before the old path gets deleted

Nothing lands as one PR where the old implementation disappears in the same
breath the new one shows up. The new path runs behind a flag, or alongside
the old one, until it's handled real traffic without incident, and only
then does the old path come out, as its own separate, easy-to-revert
change. The point at which you can no longer easily go back is the point a
bad bet turns into a genuine crisis instead of a minor annoyance, and
keeping that exit open is worth a little short-term duplication.

### IV. Direction changes get argued before they get built

A different database, a different language for a piece of the system, a
different deployment model, dropping something that was already committed
to: that conversation happens with the whole team before any code exists,
not as a surprise two-thousand-line PR someone's now expected to litigate
under time pressure. By the time there's a PR on the table, the social
pressure runs toward finding a way to approve the code rather than
re-opening whether the premise was right in the first place. Have that
argument while it's still just an argument.

### V. Every dependency earns its place

Before it goes into `go.mod` or `package.json`, ask what it costs to remove
later, whether it's actually maintained, and whether the problem is big
enough to justify an external dependency over fifty lines of our own code.
Something added casually today is something every future contributor now
has to trust, audit, and eventually maybe migrate off of. Dependency
sprawl rarely comes from one bad call. It comes from a lot of individually
reasonable ones that nobody added up.

### VI. The roadmap is the plan of record

If a phase's scope needs to shift because something learned along the way
genuinely calls for it, that's the roadmap doing its job. "Ship, learn,
adjust" is the whole idea. What's not fine is someone quietly building
Phase 3 functionality while Phase 1 is still open, or dropping something
that was scoped in without saying so, and the rest of the team finding out
from the diff instead of from a conversation.

---

# On Following These Principles

Most of Part I exists because skipping it caused a specific, real problem
at some point, and that's reason enough to default to it. If something here
feels wrong for a particular situation, say so before writing the code, not
in the PR thread. It's a two-minute conversation beforehand and a much
longer one after the diff already exists.

A short list in Part I has no exceptions at all: secrets come from the
environment and nowhere else, credentials never appear in a log, revocation
takes effect immediately, and anything touching auth gets a second
reviewer. Everything else in Part I is the considered default, not a law.
Deviate from it deliberately, say why, and leave that reasoning somewhere
findable, whether that's a comment or a commit message.

The six items in Part II don't have exceptions either. That's deliberate,
and it's the entire reason the list stays this short.
