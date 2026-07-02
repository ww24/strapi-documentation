---
title: Using publicationFilter with the Document Service API
description: Use the publicationFilter parameter with Strapi's Document Service API to query documents by the relationship between their draft and published versions, such as never-published or modified documents.
displayed_sidebar: cmsSidebar
sidebar_label: Publication filter
toc_max_heading_level: 4
tags:
- API
- Content API
- count()
- Document Service API
- Draft & Publish
- findMany()
- findFirst()
- findOne()
- publicationFilter
- status
---

# Document Service API: `publicationFilter`

<Tldr>

Use the optional `publicationFilter` parameter to query documents by the relationship between their draft and published versions, for example drafts that were never published, or entries modified since they were last published. It works with `findOne()`, `findFirst()`, `findMany()`, and `count()`, and combines with other query parameters. `status` still decides whether you get the draft or the published row.

</Tldr>

The [`status`](/cms/api/document-service/status) parameter answers "do I want the draft or the published version?". The `publicationFilter` parameter answers a different question: "which documents do I want, based on how their draft and published versions relate?".

For example, you can ask for drafts that were never published, or for entries whose draft has unsaved changes compared to what is live. `publicationFilter` selects that group of documents first; `status` then decides which row (draft or published) is returned for each.

:::note Key terms
Strapi Draft & Publish stores each entry as up to 2 database rows for the same document and locale:

- a *draft row* (`publishedAt` is empty)
- a *published row* (`publishedAt` is set)

The `status` parameter picks which of the 2 rows to read.

`publicationFilter` instead selects a group of documents based on how their draft and published rows relate (for example, never published, or draft newer than published). Some of these questions compare the 2 rows, so they cannot be expressed by filtering on `publishedAt` alone.
:::

:::prerequisites
The [Draft & Publish](/cms/features/draft-and-publish) feature must be enabled on the content-type. If Draft & Publish is disabled, `publicationFilter` has no effect.
:::

## Quick example {#quick-example}

To read the drafts that have never been published, pass:

- `status: 'draft'` (so you read the draft row)
- and `publicationFilter: 'never-published'` (so you only keep documents with no published version):

<Endpoint
  kind="js"
  path="strapi.documents().findMany()"
  title="findMany() with publicationFilter: 'never-published'"
  description="Return the drafts that have never been published for their locale."
  codeTabs={[
    {
      label: 'JavaScript',
      code: `await strapi.documents('api::restaurant.restaurant').findMany({
    status: 'draft',
    publicationFilter: 'never-published',
});`
    }
  ]}
  responses={[
    {
      status: 200,
      statusText: 'OK',
  body: `[
    {
      documentId: "a1b2c3d4e5f6g7h8i9j0klm",
      name: "New Restaurant",
      publishedAt: null,
      locale: "en", // default locale
      // …
    }
    // …
]`
    }
  ]}
/>

<br/>
The rest of this page is organized in 2 parts: [Understand `status` vs `publicationFilter`](#understand) explains the model behind the filter, and the [API reference](#reference) lists all values, their exact definitions, and more examples. You can use the reference directly and come back to the explanations when a combination surprises you.

## `status` vs `publicationFilter` {#understand}

This section explains the model behind `publicationFilter`. You do not need it to use the common values shown in the [examples](#examples), but it is what lets you predict the result of any `status` × `publicationFilter` combination.

### The model {#the-model}

A document can have up to 2 rows for the same `(documentId, locale)` pair: a *draft row* (`publishedAt: null`) and a *published row* (non-null `publishedAt`). The [`status`](/cms/api/document-service/status) parameter picks which of these 2 rows to read, while `publicationFilter` first selects the group of documents to consider, based on how their draft and published rows relate.

Strapi selects the matching documents first, then returns the row that matches the resolved `status`. (Strapi internals refer to these groups of documents as *publication cohorts*, but you never need that term to use the API.)

### Scope: pair vs. document {#scope}

Each `publicationFilter` value has a *scope*, which matters when [Internationalization (i18n)](/cms/features/internationalization) is enabled:

- a *pair-scoped* value looks at 1 locale at a time (a `documentId` + `locale` pair). For example, `never-published` matches documents never published in that locale.
- a *document-scoped* value (its name ends in `-document`) looks at the whole document across all its locales. For example, `never-published-document` matches documents never published in any locale.

Without i18n, both scopes behave the same.

### Default `status` per API surface {#default-status}

`publicationFilter` is applied *after* `status` is resolved, so the default `status` of your API surface affects what you get back:

| API surface | Default `status` when omitted |
| ----------- | ----------------------------- |
| Document Service API (direct) | `draft` |
| [REST API](/cms/api/rest/publication-filter) | `published` |
| [GraphQL API](/cms/api/graphql#publication-filter) | `PUBLISHED` |

This matters for pair-scoped values that only contain draft rows, such as `never-published`: with the REST or GraphQL defaults (`published`), the query returns an empty result set unless you explicitly pass `status=draft` / `status: DRAFT`. The Document Service defaults to `draft`, so the [quick example](#quick-example) above works without setting `status`.

## API reference {#reference}

This section lists the accepted values, their exact definitions, what each `status` × `publicationFilter` combination returns, and more examples.

### Available values {#values}

`publicationFilter` accepts one of the following kebab-case values. REST and the Document Service API use these strings directly; GraphQL exposes the same set through the [`PublicationFilter` enum](/cms/api/graphql#publication-filter). The `Scope (i18n)` column refers to the [pair vs. document scope](#scope).

| Value | What it selects | Scope (i18n) |
| ----- | --------------- | ----- |
| `never-published` | Documents never published in that locale | Pair |
| `has-published-version` | Documents that have both a draft and a published version | Pair |
| `modified` | Documents whose draft was edited since it was last published | Pair |
| `unmodified` | Documents whose draft has not changed since it was last published | Pair |
| `published-without-draft` | Published entries with no matching draft | Pair |
| `published-with-draft` | Published entries that also have a matching draft | Pair |
| `never-published-document` | Documents never published in any locale | Document |
| `has-published-version-document` | Documents published in at least one locale | Document |

Unknown values raise a validation error (REST returns HTTP `400`; GraphQL fails at query validation).

`published-without-draft` and `published-with-draft` describe published rows, so they only return results with `status: 'published'`.

### Exact definitions {#cohort-definitions}

The following table lists the same 8 values in the same order as [above](#values), this time with the exact predicate each one applies:

| Value | Scope (i18n) | A `(documentId, locale)` pair matches when… |
| ----- | ----- | -------------------------------------------- |
| `never-published` | Pair | no row with a non-null `publishedAt` exists for the pair |
| `has-published-version` | Pair | both a draft row and a published row exist for the pair |
| `modified` | Pair | both rows exist and `draft.updatedAt > published.updatedAt` |
| `unmodified` | Pair | both rows exist and `draft.updatedAt <= published.updatedAt` |
| `published-without-draft` | Pair | a published row exists for the pair and no draft row exists |
| `published-with-draft` | Pair | a published row exists for the pair and a draft row also exists |
| `never-published-document` | Document | no row with a non-null `publishedAt` exists for the `documentId` in any locale |
| `has-published-version-document` | Document | at least one published row exists for the `documentId` in any locale |

:::note
`has-published-version` excludes orphan published rows (a published row with no draft sibling for the same pair). Those rows match `published-without-draft` instead when `status` is `published`.
:::

### Which rows a combination returns {#status-combination}

Because the matching documents are selected before `status` picks a row, some combinations always return nothing. The grid below shows which `status` × `publicationFilter` pairs return rows and which are always empty:

| `publicationFilter` | With `status: 'draft'` | With `status: 'published'` |
| ------------------- | :--------------------: | :------------------------: |
| `never-published` | ✅ | ∅ |
| `has-published-version` | ✅ | ✅ |
| `modified` | ✅ | ✅ |
| `unmodified` | ✅ | ✅ |
| `published-without-draft` | ∅ | ✅ |
| `published-with-draft` | ∅ | ✅ |
| `never-published-document` | ✅ | ∅ |
| `has-published-version-document` | ✅ | ✅ |

✅ returns the rows of the resolved status within the selected group of documents; ∅ is always empty because that group has no row of that status.

When a combination returns rows, it returns them as follows:

- With `status: 'draft'`, you get the draft rows of the selected documents.
- With `status: 'published'`, you get the published rows of the selected documents.

For example, `modified` with `status: 'draft'` returns the newer draft rows, while `modified` with `status: 'published'` returns the currently-live published rows of those same documents.

:::note
Valid but empty combinations (the ∅ cells) do not return validation errors, they return no rows.
:::

### More examples {#examples}

The following examples show the most common values. For the exact rows each combination returns, see [Which rows a combination returns](#status-combination).

#### Modified documents {#modified}

`modified` selects documents whose draft was edited since it was last published; `status` then chooses which row you get for those same documents: the newer draft or the currently-live published version.

<Endpoint
  kind="js"
  path="strapi.documents().findMany()"
  title="findMany() with publicationFilter: 'modified' and status: 'draft'"
  description="Return the newer draft rows of documents modified since their last publication."
  codeTabs={[
    {
      label: 'JavaScript',
      code: `await strapi.documents('api::restaurant.restaurant').findMany({
  status: 'draft',
  publicationFilter: 'modified',
});`
    }
  ]}
  responses={[
    {
      status: 200,
      statusText: 'OK',
      body: `[
  {
    documentId: "a1b2c3d4e5f6g7h8i9j0klm",
    name: "Biscotte Restaurant (updated)",
    publishedAt: null,
    locale: "en", // default locale
    // …
  }
  // …
]`
    }
  ]}
/>

<Endpoint
  kind="js"
  path="strapi.documents().findMany()"
  title="findMany() with publicationFilter: 'modified' and status: 'published'"
  description="Return the currently-live published rows of those same modified documents."
  codeTabs={[
    {
      label: 'JavaScript',
      code: `await strapi.documents('api::restaurant.restaurant').findMany({
  status: 'published',
  publicationFilter: 'modified',
});`
    }
  ]}
  responses={[
    {
      status: 200,
      statusText: 'OK',
      body: `[
  {
    documentId: "a1b2c3d4e5f6g7h8i9j0klm",
    name: "Biscotte Restaurant",
    publishedAt: "2024-03-14T15:40:45.330Z",
    locale: "en", // default locale
    // …
  }
  // …
]`
    }
  ]}
/>

#### Documents never published in any locale {#document-scoped}

`never-published-document` is document-scoped, so a multi-locale document with even one published locale is excluded entirely, including its draft-only locales:

<Endpoint
  kind="js"
  path="strapi.documents().findMany()"
  title="findMany() with publicationFilter: 'never-published-document'"
  description="Return the drafts of documents never published in any locale."
  codeTabs={[
    {
      label: 'JavaScript',
      code: `await strapi.documents('api::restaurant.restaurant').findMany({
  status: 'draft',
  publicationFilter: 'never-published-document',
});`
    }
  ]}
  responses={[
    {
      status: 200,
      statusText: 'OK',
      body: `[
  {
    documentId: "d41r46wac4xix5vpba7561at",
    name: "New Restaurant",
    publishedAt: null,
    locale: "en", // default locale
    // …
  }
  // …
]`
    }
  ]}
/>

#### Published entries without a draft {#published-without-draft-example}

`published-without-draft` describes published rows, so it requires `status: 'published'`:

<Endpoint
  kind="js"
  path="strapi.documents().findMany()"
  title="findMany() with publicationFilter: 'published-without-draft'"
  description="Return published entries with no matching draft row for the same locale."
  codeTabs={[
    {
      label: 'JavaScript',
      code: `await strapi.documents('api::restaurant.restaurant').findMany({
  status: 'published',
  publicationFilter: 'published-without-draft',
});`
    }
  ]}
  responses={[
    {
      status: 200,
      statusText: 'OK',
      body: `[
  {
    documentId: "j0klm1n2o3p4q5r6s7t8u9v",
    name: "Legacy Restaurant",
    publishedAt: "2024-01-10T09:15:00.000Z",
    locale: "en", // default locale
    // …
  }
  // …
]`
    }
  ]}
/>

#### Published entries with a draft {#published-with-draft-example}

`published-with-draft` also describes published rows and requires `status: 'published'`:

<Endpoint
  kind="js"
  path="strapi.documents().findMany()"
  title="findMany() with publicationFilter: 'published-with-draft'"
  description="Return published entries that also have a matching draft row for the same locale."
  codeTabs={[
    {
      label: 'JavaScript',
      code: `await strapi.documents('api::restaurant.restaurant').findMany({
  status: 'published',
  publicationFilter: 'published-with-draft',
});`
    }
  ]}
  responses={[
    {
      status: 200,
      statusText: 'OK',
      body: `[
  {
    documentId: "a1b2c3d4e5f6g7h8i9j0klm",
    name: "Biscotte Restaurant",
    publishedAt: "2024-03-14T15:40:45.330Z",
    locale: "en", // default locale
    // …
  }
  // …
]`
    }
  ]}
/>

#### Use with `findOne()` and `findFirst()` {#find-one-find-first}

If the requested document (and locale, when applicable) does not match the filter, `findOne()` and `findFirst()` return `null` even when the `documentId` exists:

<Endpoint
  kind="js"
  path="strapi.documents().findOne()"
  title="findOne() with publicationFilter: 'never-published'"
  description="Return the document only if it matches the filter, null otherwise."
  codeTabs={[
    {
      label: 'JavaScript',
      code: `await strapi.documents('api::restaurant.restaurant').findOne({
  documentId: 'a1b2c3d4e5f6g7h8i9j0klm',
  status: 'draft',
  publicationFilter: 'never-published',
});`
    }
  ]}
  responses={[
    {
      status: 200,
      statusText: 'OK',
      body: `null // the documentId exists, but the document does not match never-published`
    }
  ]}
/>

#### Count only matching documents {#count}

Without `publicationFilter`, `count({ status: 'draft' })` counts every draft row, including drafts whose document already has a published version. Add `publicationFilter` to count only the documents that match a given value (see the [`status` documentation](/cms/api/document-service/status#count)):

```js
const neverPublishedCount = await strapi
  .documents('api::restaurant.restaurant')
  .count({
    status: 'draft',
    publicationFilter: 'never-published',
  });
```

### Combine with other parameters {#combine}

`publicationFilter` is combined with other query parameters as a logical `AND`, including [`filters`](/cms/api/document-service/filters) and [`populate`](/cms/api/document-service/populate). When populating draft & publish relations, nested queries inherit the same filter logic.

### Content Manager mapping {#content-manager}

In the Content Manager, the **Draft (never published)** list filter maps to `status: 'draft'` and `publicationFilter: 'never-published-document'` (document-scoped, not the pair-scoped `never-published`). Other Status filter options use internal APIs rather than public `publicationFilter` values.
