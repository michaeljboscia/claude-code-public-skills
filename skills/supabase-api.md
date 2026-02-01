---
name: supabase-api
description: Supabase REST API (PostgREST) complete reference for HTTP/cURL operations
shareable: true
---

# Supabase REST API (PostgREST) - Claude Code Skill

**Last Updated:** 2026-02-01
**PostgREST Version:** v12-v14
**Purpose:** Complete HTTP/cURL reference for direct REST API calls to Supabase databases

---

## Overview

PostgREST transforms your PostgreSQL database into a RESTful API automatically. Supabase wraps PostgREST with an API gateway layer requiring the `apikey` header. This skill provides exact syntax, headers, query parameters, and examples for every operation—no guessing required.

**Primary Use Cases:**
- n8n HTTP Request nodes calling Supabase
- Direct cURL/API calls for automation
- Understanding filter operators and query syntax
- Avoiding common anti-patterns that cause data loss

**Related Skills:**
- `supabase-cli.md` - CLI commands, migrations, Edge Functions
- `supabase-audit.md` - Performance diagnostics, index optimization
- `n8n-workflow-development.md` - Workflow building patterns

---

## Authentication & Headers

### Required Headers

| Header | Required | Purpose |
|--------|----------|---------|
| `apikey` | **Always** | API gateway authentication |
| `Authorization` | For authenticated users | `Bearer <JWT>` format |
| `Content-Type` | POST/PATCH/PUT | `application/json` |
| `Prefer` | Optional | Control response behavior |

### API Key Types

**Anonymous key (`anon`):** Client-side applications. RLS policies control access. Safe to expose publicly.

**Service role key (`service_role`):** Server-side ONLY. **BYPASSES RLS completely**—never expose in client code.

**New format keys:** `sb_publishable_*` (anon) and `sb_secret_*` (service_role). Both formats work.

### Header Examples

```bash
# Anonymous read (public data)
curl "https://<project-ref>.supabase.co/rest/v1/todos?select=*" \
  -H "apikey: <ANON_KEY>"

# Authenticated user request
curl "https://<project-ref>.supabase.co/rest/v1/todos?select=*" \
  -H "apikey: <ANON_KEY>" \
  -H "Authorization: Bearer <USER_ACCESS_TOKEN>"

# Server-side admin (bypasses RLS)
curl "https://<project-ref>.supabase.co/rest/v1/todos" \
  -X DELETE \
  -H "apikey: <SERVICE_ROLE_KEY>" \
  -H "Authorization: Bearer <SERVICE_ROLE_KEY>" \
  -H "Content-Type: application/json"

# Write with return
curl "https://<project-ref>.supabase.co/rest/v1/todos" \
  -X POST \
  -H "apikey: <ANON_KEY>" \
  -H "Authorization: Bearer <USER_JWT>" \
  -H "Content-Type: application/json" \
  -H "Prefer: return=representation" \
  -d '{"task": "Complete project"}'
```

---

## Base URL & Methods

### Endpoint Structure

```
https://<project-ref>.supabase.co/rest/v1/<resource>
```

### HTTP Methods

| Operation | Method | Endpoint Example |
|-----------|--------|------------------|
| List/Read | GET | `/rest/v1/todos` |
| Create | POST | `/rest/v1/todos` |
| Update | PATCH | `/rest/v1/todos?id=eq.1` |
| Upsert | POST | `/rest/v1/todos` (with Prefer header) |
| Delete | DELETE | `/rest/v1/todos?id=eq.1` |
| RPC | POST | `/rest/v1/rpc/my_function` |

---

## Filter Operators Reference

### Comparison Operators

| Operator | SQL Equivalent | Example |
|----------|---------------|---------|
| `eq` | `=` | `?id=eq.5` |
| `neq` | `<>` | `?status=neq.deleted` |
| `gt` | `>` | `?age=gt.21` |
| `gte` | `>=` | `?age=gte.18` |
| `lt` | `<` | `?price=lt.100` |
| `lte` | `<=` | `?price=lte.50` |

### Pattern Matching

| Operator | Description | Example |
|----------|-------------|---------|
| `like` | Case-sensitive LIKE | `?name=like.John*` |
| `ilike` | Case-insensitive LIKE | `?name=ilike.*smith*` |
| `match` | POSIX regex | `?email=match.^[a-z]+@` |
| `imatch` | Case-insensitive regex | `?name=imatch.^j` |

**Use `*` as wildcard** (maps to SQL `%`).

### Null and Boolean (CRITICAL: use `is`, not `eq`)

```bash
# ✅ CORRECT
?deleted_at=is.null
?is_active=is.true
?is_admin=is.false

# ❌ WRONG - this doesn't work!
?deleted_at=eq.null
```

### Multiple Values (IN)

```bash
?status=in.(pending,processing,shipped)
?id=in.(1,2,3,4,5)
```

### Array Operators

| Operator | Description | Example |
|----------|-------------|---------|
| `cs` | Contains | `?tags=cs.{javascript,react}` |
| `cd` | Contained by | `?tags=cd.{a,b,c,d,e}` |
| `ov` | Overlaps | `?categories=ov.{tech,science}` |

### Negation

Prefix any operator with `not`:
```bash
?status=not.eq.deleted
?tags=not.cs.{archived}
?deleted_at=not.is.null
```

### Logical Operators

```bash
# OR condition
?or=(age.lt.18,age.gt.65)

# Nested AND within OR
?or=(status.eq.active,and(role.eq.admin,verified.is.true))

# NOT with logical operators
?not.or=(status.eq.deleted,status.eq.archived)
```

### Full-Text Search

| Operator | PostgreSQL Function | Example |
|----------|---------------------|---------|
| `fts` | `to_tsquery` | `?body=fts.coffee` |
| `plfts` | `plainto_tsquery` | `?content=plfts.fat cats` |
| `phfts` | `phraseto_tsquery` | `?content=phfts(english).The Fat Cats` |
| `wfts` | `websearch_to_tsquery` | `?content=wfts(english).coffee -tea` |

---

## Ordering & Pagination

### Ordering

```bash
?order=created_at              # Ascending (default)
?order=created_at.desc         # Descending
?order=priority.desc,created_at.asc  # Multiple columns
?order=due_date.asc.nullslast  # Null handling
```

### Pagination

```bash
# Query parameters (recommended)
?limit=20                      # First 20 rows
?limit=20&offset=20            # Page 2 (rows 21-40)
?limit=50&offset=100           # Skip first 100, get next 50

# Get total count
curl "https://<ref>.supabase.co/rest/v1/todos?limit=25" \
  -H "apikey: <KEY>" \
  -H "Prefer: count=exact"
# Response header: Content-Range: 0-24/1523
```

**Count options:**
- `count=exact` — Accurate but slow on large tables
- `count=planned` — Fast, uses PostgreSQL statistics (approximate)
- `count=estimated` — Exact up to threshold, then planned

---

## Embedded Resources (Joins)

```bash
# Basic join (left join by default)
?select=id,task,users(id,name,email)

# Nested joins
?select=*,orders(id,items(product_name,quantity))

# Inner join (filter parent rows)
?select=*,comments!inner(*)

# With filter on joined table
?select=*,comments!inner(*)&comments.created_at=gte.2024-01-01

# Rename embedded resources
?select=*,author:users(name,email)

# Disambiguate multiple foreign keys
?select=*,billing:addresses!billing_address_id(*),shipping:addresses!shipping_address_id(*)
```

---

## Insert Operations (POST)

### Single Row

```bash
curl "https://<ref>.supabase.co/rest/v1/todos" \
  -X POST \
  -H "apikey: <KEY>" \
  -H "Authorization: Bearer <KEY>" \
  -H "Content-Type: application/json" \
  -d '{"task": "Learn Supabase", "completed": false}'
```

### Bulk Insert

```bash
curl "https://<ref>.supabase.co/rest/v1/todos" \
  -X POST \
  -H "apikey: <KEY>" \
  -H "Content-Type: application/json" \
  -d '[
    {"task": "Task 1", "completed": false},
    {"task": "Task 2", "completed": false}
  ]'
```

**All objects must have matching keys.**

### Return Inserted Data

```bash
# Return full record(s)
curl "https://<ref>.supabase.co/rest/v1/todos" \
  -X POST \
  -H "Prefer: return=representation" \
  -d '{"task": "New task"}'

# Return specific columns only
curl "https://<ref>.supabase.co/rest/v1/todos?select=id,created_at" \
  -X POST \
  -H "Prefer: return=representation" \
  -d '{"task": "New task"}'
```

---

## Update Operations (PATCH)

### ⚠️ CRITICAL: Always Include Filters

**Without filters, PATCH updates EVERY ROW in the table.**

```bash
# ✅ CORRECT - with filter
curl "https://<ref>.supabase.co/rest/v1/todos?id=eq.5" \
  -X PATCH \
  -H "apikey: <KEY>" \
  -H "Content-Type: application/json" \
  -d '{"completed": true}'

# ❌ DANGEROUS - no filter, updates ALL rows
curl "https://<ref>.supabase.co/rest/v1/todos" \
  -X PATCH \
  -d '{"completed": true}'
```

### Limit Affected Rows (Safety)

```bash
curl "https://<ref>.supabase.co/rest/v1/todos?status=eq.pending" \
  -X PATCH \
  -H "Prefer: handling=strict, max-affected=10" \
  -d '{"status": "processed"}'
```

Returns error `PGRST124` if more than 10 rows would be affected.

---

## Upsert Operations

### Merge Duplicates (Update on Conflict)

```bash
curl "https://<ref>.supabase.co/rest/v1/products" \
  -X POST \
  -H "Content-Type: application/json" \
  -H "Prefer: resolution=merge-duplicates" \
  -d '[{"sku": "ABC123", "name": "Widget", "price": 29.99}]'
```

### Ignore Duplicates

```bash
curl "https://<ref>.supabase.co/rest/v1/products" \
  -X POST \
  -H "Prefer: resolution=ignore-duplicates" \
  -d '[{"sku": "ABC123", "name": "Different Name"}]'
```

### Specify Conflict Column

```bash
curl "https://<ref>.supabase.co/rest/v1/employees?on_conflict=email" \
  -X POST \
  -H "Prefer: resolution=merge-duplicates" \
  -d '[{"email": "john@example.com", "name": "John Updated"}]'
```

---

## Delete Operations

### ⚠️ CRITICAL: Always Include Filters

**Without filters, DELETE removes EVERY ROW in the table.**

```bash
# ✅ CORRECT - with filter
curl "https://<ref>.supabase.co/rest/v1/todos?id=eq.5" \
  -X DELETE \
  -H "apikey: <KEY>"

# ❌ DANGEROUS - no filter, deletes ALL rows
curl "https://<ref>.supabase.co/rest/v1/todos" \
  -X DELETE
```

### Soft Delete Pattern

```bash
curl "https://<ref>.supabase.co/rest/v1/todos?id=eq.5" \
  -X PATCH \
  -H "Prefer: return=representation" \
  -d '{"deleted_at": "2024-01-15T10:30:00Z"}'
```

---

## RPC (Remote Procedure Calls)

```bash
# Basic RPC call
curl "https://<ref>.supabase.co/rest/v1/rpc/add_numbers" \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{"a": 5, "b": 3}'

# GET for IMMUTABLE/STABLE functions
curl "https://<ref>.supabase.co/rest/v1/rpc/get_config?key=site_name" \
  -H "apikey: <KEY>"

# Functions returning tables (with filters)
curl "https://<ref>.supabase.co/rest/v1/rpc/search_products?name=ilike.*phone*&price=lt.500&order=price.asc&limit=10" \
  -H "Prefer: count=exact"
```

---

## Prefer Header Reference

Multiple values are **comma-separated**:

| Preference | Values | Description |
|------------|--------|-------------|
| `return` | `minimal`, `headers-only`, `representation` | Control response body |
| `count` | `exact`, `planned`, `estimated` | Row counting method |
| `resolution` | `merge-duplicates`, `ignore-duplicates` | Upsert behavior |
| `missing` | `default` | Use column DEFAULT values |
| `handling` | `strict`, `lenient` | Error on invalid prefs |
| `max-affected` | Integer | Limit affected rows |
| `tx` | `commit`, `rollback` | Transaction control |

### Combined Example

```bash
curl "https://<ref>.supabase.co/rest/v1/employees" \
  -X POST \
  -H "Prefer: resolution=merge-duplicates, missing=default, return=representation, count=exact" \
  -d '[...]'
```

### Testing Without Persisting (Rollback)

```bash
curl "https://<ref>.supabase.co/rest/v1/orders" \
  -X POST \
  -H "Prefer: tx=rollback, return=representation" \
  -d '{"product_id": 123}'
# Returns the would-be inserted row, but data is NOT persisted
```

---

## Response Handling

### HTTP Status Codes

| Code | Meaning | When Returned |
|------|---------|---------------|
| **200** | OK | Successful GET, PATCH, DELETE with return=representation |
| **201** | Created | Successful POST |
| **204** | No Content | Write with return=minimal |
| **400** | Bad Request | Invalid syntax, constraint violation |
| **401** | Unauthorized | Missing/invalid JWT |
| **403** | Forbidden | RLS denied access |
| **409** | Conflict | Unique/FK constraint violation |

### Error Response Format

```json
{
  "code": "PGRST204",
  "message": "Column 'nonexistent' not found",
  "details": null,
  "hint": "Perhaps you meant 'name'?"
}
```

### Common PostgREST Error Codes

| Code | HTTP | Description |
|------|------|-------------|
| PGRST000 | 503 | Database connection failed |
| PGRST100 | 400 | Query string parse error |
| PGRST116 | 406 | Singular response with 0 or >1 rows |
| PGRST124 | 400 | max-affected constraint exceeded |
| PGRST200 | 404 | Table/view not found |
| PGRST300 | 401 | Invalid JWT |
| PGRST301 | 401 | JWT expired |

---

## JSON/JSONB Column Queries

```bash
# Extract JSON field (returns JSON)
?select=id,metadata->settings

# Extract as text (returns string)
?select=id,metadata->>email

# Deep nested access
?select=id,data->address->city

# Filter on JSON field
?metadata->>status=eq.active

# JSONB containment
?metadata=cs.{"role":"admin"}
```

---

## n8n HTTP Request Node Configuration

### Header Authentication Setup

| Setting | Value |
|---------|-------|
| Authentication | Header Auth |
| Name 1 | `apikey` |
| Value 1 | `<your-supabase-key>` |
| Name 2 | `Authorization` |
| Value 2 | `Bearer <your-supabase-key>` |

### Query Parameters for Filters

| Name | Value |
|------|-------|
| `select` | `id,name,email` |
| `status` | `eq.active` |
| `created_at` | `gte.2024-01-01` |
| `order` | `created_at.desc` |
| `limit` | `50` |

### n8n Known Issue

The native Supabase node may fail with 403 errors due to conflicting auth headers. **Workaround:** Use HTTP Request node with manual header configuration.

---

## Rate Limits & Performance

**REST API (PostgREST):** No hard rate limits. Free tier sustains ~1,200 reads/second, ~1,000 inserts/second.

**Request size:** Default 2MB body limit (configurable on paid plans).

**Bulk insert:** Single POST with array is more efficient than multiple requests.

---

## Anti-Patterns & Common Mistakes

### ❌ Unfiltered UPDATE/DELETE

```bash
# DANGEROUS - affects ALL rows
curl /users -X PATCH -d '{"active": false}'
curl /users -X DELETE
```

**Always include filters for PATCH and DELETE.**

### ❌ Using `eq` for null checking

```bash
# WRONG
?deleted_at=eq.null

# CORRECT
?deleted_at=is.null
```

### ❌ Wrong Prefer syntax

```bash
# WRONG
Prefer: return-representation
Prefer: count: exact

# CORRECT
Prefer: return=representation
Prefer: count=exact
```

### ❌ Wrong schema header for mutations

```bash
# WRONG - Accept-Profile is for GET only
curl -X POST /items -H "Accept-Profile: custom"

# CORRECT
curl -X POST /items -H "Content-Profile: custom"
```

### URL Encoding

| Character | Encode As |
|-----------|-----------|
| Space | `%20` |
| `&` | `%26` |
| `{` `}` | `%7B` `%7D` |

Use `curl -g` flag to disable glob parsing when using `{}` or `[]`.

---

## Quick Reference Card

### Headers
```
apikey: <KEY>                           # Always required
Authorization: Bearer <JWT>             # For authenticated requests
Content-Type: application/json          # For POST/PATCH/PUT
Prefer: return=representation           # Return created/modified data
Prefer: count=exact                     # Include total count
Prefer: resolution=merge-duplicates     # Upsert mode
```

### Filter Syntax
```
?column=eq.value           # Equal
?column=neq.value          # Not equal
?column=gt.value           # Greater than
?column=is.null            # Is null
?column=is.true            # Is true
?column=in.(a,b,c)         # In list
?column=like.*pattern*     # Pattern match
?column=cs.{a,b}           # Array contains
?column=not.eq.value       # Negation
?or=(cond1,cond2)          # OR condition
```

### Pagination
```
?limit=20&offset=40        # Page 3 of 20 items per page
?order=col.desc            # Order descending
?order=col.asc.nullslast   # Nulls at end
```

### Joins
```
?select=*,related(*)       # Left join
?select=*,related!inner(*) # Inner join
?select=*,alias:table(*)   # Renamed join
```

---

**Source Document:** `/Users/mikeboscia/My Drive/GTM Machine content/Research & Strategy/Supabase REST API (PostgREST) Technical Reference.md`
