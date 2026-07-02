---
title: Using publicationFilter with the Document Service API
description: Use the publicationFilter parameter with Strapi's Document Service API to query documents by the relationship between their draft and published versions, such as never-published or modified documents.
displayed_sidebar: cmsSidebar
sidebar_label: Publication filter
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

The [`status`](/cms/api/document-service/status) parameter answers "do I want the draft or the published version?". The `publicationFilter` parameter answers a different question: "which documents do I want, based on how their draft and published versions relate?". For example: drafts that were never published, or entries whose draft has unsaved changes compared to what is live.

With Draft & Publish, Strapi stores each entry as up to 2 database rows per locale: a *draft row* and a *published row*. `publicationFilter` selects which documents to consider, based on how these 2 rows relate; `status` picks which of the 2 rows is returned for each of them. The Document Service API returns draft rows when `status` is omitted; REST and GraphQL return published rows instead, so the equivalent queries there need an explicit `status` (see [REST API: `publicationFilter`](/cms/api/rest/publication-filter)).

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
The next section lists all accepted values, [Use cases](#use-cases) maps common goals to the parameters to pass, and the rest of the page shows complete examples.

## Available values {#values}

`publicationFilter` accepts one of the following kebab-case values. GraphQL exposes the same set through the [`PublicationFilter` enum](/cms/api/graphql#publication-filter). Unknown values raise a validation error (REST returns HTTP `400`; GraphQL fails at query validation).

| Value | Selects |
| ----- | ------- |
| `never-published` | Documents never published in a given locale |
| `has-published-version` | Documents that have both a draft and a published version |
| `modified` | Documents whose draft was edited since it was last published |
| `unmodified` | Documents whose draft has not changed since it was last published |
| `published-without-draft` | Published entries with no draft counterpart |
| `published-with-draft` | Published entries that also have a draft |
| `never-published-document` | Documents never published in any locale |
| `has-published-version-document` | Documents published in at least one locale |

:::note Values ending in -document and localization
Values ending in `-document` consider all locales of a document, which matters when [Internationalization (i18n)](/cms/features/internationalization) is enabled: for example, `never-published-document` excludes a document as soon as one of its locales is published. All other values consider one locale at a time. Without i18n, both variants behave the same.
:::

:::info "Cohorts"?
Strapi internals refer to the groups of documents these values select as *publication cohorts*, but you never need that term to use the API.
:::

## Possible use cases {#use-cases}

| I want to… | Use `status` with… | Use `publicationFilter` with… | Full example |
| ---------- | ------------------ | ----------------------------- | ------------ |
| Find drafts never published in a given locale | `draft` | `never-published` | [Quick example](#quick-example) |
| Find the newer drafts of entries modified since their last publication | `draft` | `modified` | [Modified documents](#modified) |
| Find the currently-live version of those same modified entries | `published` | `modified` | [Modified documents](#modified) |
| Find drafts that have not changed since their last publication | `draft` or `published` | `unmodified` | – |
| Find entries that have both a draft and a published version | `draft` or `published` | `has-published-version` | – |
| Find published entries that have no draft counterpart | `published` | `published-without-draft` | [Published entries without a draft](#published-without-draft-example) |
| Find published entries that also have a draft | `published` | `published-with-draft` | [Published entries with a draft](#published-with-draft-example) |
| Find drafts of documents never published in any locale | `draft` | `never-published-document` | [Documents never published in any locale](#document-scoped) |
| Find documents published in at least one locale | `draft` or `published` | `has-published-version-document` | – |
| Check whether one specific document matches a value (with `findOne()` or `findFirst()`) | `draft` or `published` | any value | [Use with findOne() and findFirst()](#find-one-find-first) |
| Count only the documents that match a value (with `count()`) | `draft` or `published` | any value | [Count only matching documents](#count) |

:::caution
Pairing a value with the opposite `status` from the table above is valid but returns no rows rather than an error: for example, `never-published` with `status: 'published'` returns an empty result, because these documents have no published row yet.
:::

## Examples

### Modified documents {#modified}

`modified` selects documents whose draft was edited since it was last published (the draft row's `updatedAt` is more recent than the published row's). The example below returns their newer draft rows; pass `status: 'published'` instead to return the currently-live published version of those same documents.

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

### Documents never published in any locale {#document-scoped}

`never-published-document` considers all locales, so a multi-locale document with even one published locale is excluded entirely, including its draft-only locales:

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

### Published entries without a draft {#published-without-draft-example}

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

### Published entries with a draft {#published-with-draft-example}

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

### Use with `findOne()` and `findFirst()` {#find-one-find-first}

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

### Count only matching documents {#count}

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

:::note Note: Content Manager mapping
In the Content Manager, the **Draft (never published)** list filter maps to `status: 'draft'` and `publicationFilter: 'never-published-document'` (document-scoped, not the per-locale `never-published`). 
:::
