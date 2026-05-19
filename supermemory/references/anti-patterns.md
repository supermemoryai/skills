# Supermemory anti-patterns

This file is for AI coding agents. It enumerates the patterns prior codegen has confidently invented or misused when integrating Supermemory, and the canonical replacement for each. If you are generating Supermemory code, **read this before writing**.

The patterns below were collected from a survey of 52 hackathon projects that import `supermemory` or call `api.supermemory.ai`. Every "wrong" example here is something AI actually wrote in production code.

---

## 1. Auth header confusion

The API accepts several auth headers for backward compatibility, but **new code should always use `Authorization: Bearer`**. Mixing headers is the easiest way to end up with code that works in one place and breaks in another.

| Pattern | Status | Use instead |
| --- | --- | --- |
| `Authorization: Bearer $KEY` | ✅ Canonical | — |
| `x-supermemory-api-key: $KEY` | Works, but stop emitting | `Authorization: Bearer $KEY` |
| `x-api-key: $KEY` | Works, but stop emitting | `Authorization: Bearer $KEY` |
| `x-sm-user-id: ...` | Real, **only** for `/v3/infinite-context/*` | Use `containerTag` in the body for everything else |
| `x-sm-project: ...` | Real, **only** for MCP routes | Use `containerTag` in the body for everything else |
| `x-project-id`, `X-Workspace-Id`, `x-sm-project-id` | Fabricated. Do not emit. | — |

Hallucinated multi-tenant headers are the most common form of "looks plausible, silently does nothing." If you find yourself adding a custom header to scope a request, **stop** — scoping goes in the body via `containerTag`.

---

## 2. Endpoint confusion

| Pattern | Status | Use instead |
| --- | --- | --- |
| `POST /v3/documents` | ✅ Canonical write | — |
| `POST /v4/search` | ✅ Canonical search | — |
| `POST /v4/profile` | ✅ Canonical profile + inline search | — |
| `POST /v3/search` | Legacy; still works but missing v4 features | `POST /v4/search` |
| `POST /v3/memories` | 308-redirects to `/v3/documents`; only works if your HTTP client follows redirects | `POST /v3/documents` |
| `POST /v1/*` (`/v1/search`, `/v1/memories`, `/v1/add`) | Retired before v3 shipped. Will 404. | `/v3/documents`, `/v4/search` |
| `GET /search?...` | Search is `POST` with a JSON body. There is no GET form. | `POST /v4/search` |

**Don't confidently document an endpoint you haven't verified.** Several projects in the survey shipped doc comments like `Endpoints (verified 2026-05-17): POST /v3/search` — the verification was AI-fabricated and locked the wrong endpoint into the codebase.

---

## 3. Body shape

The canonical write body is `{ content, containerTag, metadata? }`. The canonical search body is `{ q, containerTag, searchMode?, limit?, threshold?, rerank?, rewriteQuery?, filters? }`.

| Pattern | Status | Use instead |
| --- | --- | --- |
| `containerTag: "user_123"` | ✅ Canonical (singular string) | — |
| `containerTags: ["user_123"]` | Deprecated plural array; still accepted on `/v3/*` for back-compat | `containerTag: "user_123"` |
| `userId: "..."` | Fabricated. Not a real body field. | `containerTag: userId` |
| `spaces: [...]`, `namespace: "..."` | Fabricated. Supermemory has no "spaces" concept. | `containerTag` |
| `schema: "my-envelope-v1"` | Fabricated. Don't wrap requests in custom envelopes. | Pass `content` directly |
| `container: "..."` (without `Tag`) | Fabricated. | `containerTag` |
| top-level `tags: [...]` | Fabricated. | Put labels in `metadata` |
| `filter: { ... }` (singular) | Fabricated. | `filters: { AND: [...] }` |

---

## 4. Search parameters (v4-only features)

`rerank`, `rewriteQuery`, `filters`, and `threshold` are valid on `POST /v4/search` and ignored on `POST /v3/search`. If you want any of those, you must hit `/v4/search`.

| Pattern | Status | Use instead |
| --- | --- | --- |
| `threshold: 0.3` | ✅ Canonical similarity threshold | — |
| `chunk_threshold` / `chunkThreshold` | Fabricated kwarg. | `threshold` |
| `rerank: true` on `/v3/search` | Silently ignored | Use `/v4/search` |
| `rewriteQuery: true` on `/v3/search` | Silently ignored | Use `/v4/search` |
| `searchMode: 'hybrid' \| 'memories' \| 'documents'` | ✅ Canonical | — |
| `mode`, `search_type`, `type` | Fabricated. | `searchMode` |

---

## 5. SDK methods

The TypeScript and Python SDKs share method names (in their respective casing). The shape below is what actually exists.

**Top-level helpers:**

- `client.add({ content, containerTag, metadata })` — write content
- `client.profile({ containerTag, q })` — fetch profile + optional inline search

**Namespaced:**

- `client.search.memories({ q, containerTag, ... })` — semantic search
- `client.documents.list({ containerTag, limit })` — list documents
- `client.documents.delete({ docId })` — delete one document
- `client.memories.update(...)`, `client.memories.delete(...)`, `client.memories.forget(...)` — direct memory mutation. **Real, but most apps don't need these.** Memories are auto-extracted; only reach for these if you're exposing a "manage my memories" UI.

**Do not emit:**

| Pattern | Reason |
| --- | --- |
| `client.search.execute(...)` | Doesn't exist. Use `client.search.memories`. |
| `client.search.documents(...)` | Older name. Use `client.search.memories`. |
| `client.documents.add(...)` | Doesn't exist. Use top-level `client.add`. |
| `client.documents.deleteBulk(...)` / `delete_bulk` | Doesn't exist. Loop `client.documents.delete` or write to a tag you can later wipe. |
| `client.documents.batch_add(...)` | Doesn't exist. Loop `client.add`. |
| `client.memories.list(...)` | Doesn't exist. Use `client.documents.list`. |
| `client.memories.updateMemory(...)` | Typo. Real method is `client.memories.update`. |
| kwargs: `sort`, `order`, `include_content`, `include_full_docs`, `timeout` | None of these are accepted by SDK methods. |

---

## 6. The single biggest bug: missing `containerTag`

About a sixth of the projects surveyed call `/v3/search` or `/v3/documents` with **no `containerTag` at all**, usually because the AI substituted an invented `userId` field or pushed the user identity into a fake header. The endpoint returns 200 and the code "works" in dev with one test user — then in prod, every user's writes and reads collapse into a single shared bucket.

Two rules:

1. **Every write must include `containerTag`.** If you don't have a tenant key yet, pass `containerTag: "default"` explicitly so the omission is visible in code review.
2. **Every search must include `containerTag`.** Don't try to filter by user id inside the `q` string ("user_phone:+1555... move history") — `q` is semantic search input, not a where-clause.

---

## 7. Doc-comment hygiene

A surprising amount of the damage in the survey came from AI-written doc comments that *asserted* a wrong API was correct:

```ts
/**
 * Endpoints (verified 2026-05-17 against api.supermemory.ai):
 *   - POST /v3/search     body { q, limit? }   → { results: [...], total, timing }
 *   - POST /v3/documents  body { content, containerTags?, metadata? } → { id, status }
 */
```

Three things wrong in five lines, all asserted as "verified." If you are generating doc comments for a Supermemory module:

- Don't claim verification you didn't perform.
- Link to `https://supermemory.ai/docs` rather than restating the shape inline.
- If you must describe the shape inline, copy it from `references/api-reference.md` in this skill.
