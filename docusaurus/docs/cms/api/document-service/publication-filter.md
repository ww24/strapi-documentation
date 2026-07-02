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

The `publicationFilter` is a parameter that, combined with [the `status` parameter](/cms/api/document-service/status), can help you cover complex queries to find exactly what you need with the [Document Service API](/cms/api/document-service).

While `status` answers "do I want the draft or the published version?", the `publicationFilter` parameter answers a different question: "which documents do I want, based on how their draft and published versions relate?". This is useful for example to find drafts that were never published, or entries whose draft has unsaved changes compared to what is live.

:::caution Caution: Different default behaviors for different APIs
The Document Service API returns draft versions of documents when `status` is omitted, while REST and GraphQL return the published ones instead, so REST API queries need an explicit `status` (see [REST API: `publicationFilter`](/cms/api/rest/publication-filter)).
:::

:::prerequisites
The [Draft & Publish](/cms/features/draft-and-publish) feature must be enabled on the content-type. If Draft & Publish is disabled, `publicationFilter` has no effect.
:::

## Available values {#values}

`publicationFilter` accepts one of the following kebab-case values.

| Value | Selects |
| ----- | ------- |
| `never-published` | Documents never published in a given locale |
| `has-published-version` | Documents that have both a draft and a published version |
| `modified` | Documents whose draft was edited since it was last published |
| `unmodified` | Documents whose draft has not changed since it was last published |
| `published-without-draft` | Published entries with no draft counterpart |
| `published-with-draft` | Published entries that also have a draft |
| `never-published-document` | Documents never published in any locale |
| `has-published-version-document` | Documents published in at least one locale<br/>(useful when [i18n](/cms/features/internationalization) is enabled) |

:::info "Cohorts"?
Strapi internals refer to the groups of documents these values select as *publication cohorts*, but you never need that term to use the API.
:::

:::note Notes
*  GraphQL exposes the same set through the [`PublicationFilter` enum](/cms/api/graphql#publication-filter).
* Unknown values raise a validation error (REST returns HTTP `400`; GraphQL fails at query validation).
* Values ending in `-document` consider all locales of a document, which matters when [Internationalization (i18n)](/cms/features/internationalization) is enabled: for example, `never-published-document` excludes a document as soon as one of its locales is published. All other values consider one locale at a time. Without i18n, both variants behave the same.
:::

## Possible use cases {#use-cases}

The following table lists many possible use cases, illustrating how the `status` and `publicationFilter` parameters can be combined to find exactly what you need with the Document Service API:

| I want to… | Use `status` as… | Use `publicationFilter` as… | Full example |
| ---------- | ------------------ | ----------------------------- | ------------ |
| Find drafts never published in a given locale | `draft` | `never-published` | [Find never published drafts](#never-published-example) |
| Find the newer drafts of entries modified since their last publication | `draft` | `modified` | [Modified documents](#modified) |
| Find drafts of documents never published in any locale | `draft` | `never-published-document` | [Documents never published in any locale](#document-scoped) |
| Find the currently-live version of those same modified entries | `published` | `modified` | [Modified documents](#modified) |
| Find published entries that have no draft counterpart | `published` | `published-without-draft` | [Published entries without a draft](#published-without-draft-example) |
| Find published entries that also have a draft | `published` | `published-with-draft` | [Published entries with a draft](#published-with-draft-example) |
| Find drafts that have not changed since their last publication | `draft` or `published` | `unmodified` | [Unmodified entries](#unmodified-example) |
| Find entries that have both a draft and a published version | `draft` or `published` | `has-published-version` | [Entries with a published version](#has-published-version-example) |
| Find documents published in at least one locale | `draft` or `published` | `has-published-version-document` | [Documents published in at least one locale](#has-published-version-document-example) |
| Check whether one specific document matches a value (with `findOne()` or `findFirst()`) | `draft` or `published` | any value | [Use with findOne() and findFirst()](#find-one-find-first) |
| Count only the documents that match a value (with `count()`) | `draft` or `published` | any value | [Count only matching documents](#count) |

:::caution
Pairing a value with the opposite `status` from the table above is valid but returns no rows rather than an error: for example, `never-published` with `status: 'published'` returns an empty result, because these documents have no published row yet.
:::

## Examples

The following section lists the most common use cases summed up in the [table](#use-cases) above.

### Find never published drafts {#never-published-example}

One of the most common use cases is to find the drafts that have never been published. To do so, pass:

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

### Find modified documents {#modified}

`publicationFilter: modified` selects documents whose draft has unpublished changes. `status` then decides which version of those documents you get back.

With `status: 'draft'`, the query returns the pending draft rows:

<Endpoint
  kind="js"
  path="strapi.documents().findMany()"
  title="findMany() with publicationFilter: 'modified' and status: 'draft'"
  description="Return the pending draft rows of documents with unpublished changes."
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

With `status: 'published'`, the same query returns the currently live version of those documents instead:

<Endpoint
  kind="js"
  path="strapi.documents().findMany()"
  title="findMany() with publicationFilter: 'modified' and status: 'published'"
  description="Return the currently live rows of documents with unpublished changes."
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

### Find documents never published in any locale {#document-scoped}

`publicationFilter: never-published-document` considers all locales, so a multi-locale document with even one published locale is excluded entirely, including its draft-only locales:

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

### Find published entries without a draft {#published-without-draft-example}

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

### Find published entries with a draft {#published-with-draft-example}

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

### Find entries with a published version {#has-published-version-example}

`has-published-version` selects documents that have both a draft and a published version for the same locale (it excludes published entries that have no draft counterpart). It returns rows with either `status`; the example below returns the draft rows.

<Endpoint
  kind="js"
  path="strapi.documents().findMany()"
  title="findMany() with publicationFilter: 'has-published-version'"
  description="Return the draft rows of documents that also have a published version for the same locale."
  codeTabs={[
    {
      label: 'JavaScript',
      code: `await strapi.documents('api::restaurant.restaurant').findMany({
    status: 'draft',
    publicationFilter: 'has-published-version',
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
    publishedAt: null,
    locale: "en", // default locale
    // …
  }
  // …
]`
    }
  ]}
/>

### Find unmodified entries {#unmodified-example}

`unmodified` selects documents whose draft has not changed since it was last published (the draft row's `updatedAt` is not more recent than the published row's). It returns rows with either `status`; the example below returns the draft rows.

<Endpoint
  kind="js"
  path="strapi.documents().findMany()"
  title="findMany() with publicationFilter: 'unmodified'"
  description="Return the draft rows of documents unchanged since their last publication."
  codeTabs={[
    {
      label: 'JavaScript',
      code: `await strapi.documents('api::restaurant.restaurant').findMany({
    status: 'draft',
    publicationFilter: 'unmodified',
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
    publishedAt: null,
    locale: "en", // default locale
    // …
  }
  // …
]`
    }
  ]}
/>

### Find documents published in at least one locale {#has-published-version-document-example}

`has-published-version-document` considers all locales, so it matches a document as soon as one of its locales is published. With `status: 'draft'`, it returns the draft rows of every locale of those documents, including locales that were never published themselves:

<Endpoint
  kind="js"
  path="strapi.documents().findMany()"
  title="findMany() with publicationFilter: 'has-published-version-document'"
  description="Return the draft rows of documents published in at least one locale."
  codeTabs={[
    {
      label: 'JavaScript',
      code: `await strapi.documents('api::restaurant.restaurant').findMany({
    status: 'draft',
    publicationFilter: 'has-published-version-document',
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
    publishedAt: null,
    locale: "en", // published in at least one locale
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
