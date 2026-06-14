# Query Index Background & Troubleshooting

## How the EDS Query Index Works

The query index is the primary mechanism for blocks and components to discover and list content in an EDS site. It is configured via a `helix-query.yaml` file in the GitHub repository and served as JSON at `/query-index.json`.

### Key Concepts

- **helix-query.yaml** — Lives in the GitHub repo root. Defines which properties to index and how they are sourced (from metadata, headings, or content).
- **query-index.json** — The live JSON endpoint. Returns an array of page entries with the indexed properties.
- **Consumers** — Blocks and components that fetch `query-index.json` to build dynamic lists: navigation, footer, card lists, search results, recent posts, tag-filtered collections.
- **Default limit** — The index returns a maximum of 500 entries by default. Sites with more pages need to paginate or increase the limit.
- **Index freshness** — The index updates when pages are previewed or published via Sidekick. Unpublished pages remain in the index until explicitly removed.

### Common Properties

| Property | Source | Typical Consumers |
|----------|--------|-------------------|
| `path` | automatic | All consumers |
| `title` | metadata | Nav, cards, search |
| `description` | metadata | Cards, search |
| `image` | metadata | Cards, hero blocks |
| `lastModified` | automatic | Freshness sorting |
| `template` | metadata | Filtered collections |
| `tags` | metadata | Tag-filtered blocks |
| `author` | metadata | Blog cards |

### Example helix-query.yaml

```yaml
indices:
  all:
    include:
      - /**
    properties:
      title:
        select: head > meta[property="og:title"]
        value: attribute(el, "content")
      description:
        select: head > meta[name="description"]
        value: attribute(el, "content")
      image:
        select: head > meta[property="og:image"]
        value: attribute(el, "content")
      tags:
        select: head > meta[property="article:tag"]
        values: attribute(el, "content")
  blog:
    include:
      - /blog/**
    properties:
      title:
        select: head > meta[property="og:title"]
        value: attribute(el, "content")
      author:
        select: head > meta[name="author"]
        value: attribute(el, "content")
```

## Troubleshooting

| Problem | Cause | Solution |
|---------|-------|----------|
| Query index returns 404 | No `helix-query.yaml` in the repo | Create a `helix-query.yaml` with the desired property configuration |
| Pages missing from index | Pages not previewed/published after index was configured | Open each missing page with Sidekick and click Preview, then Publish |
| Stale entries persist after deletion | Index caches entries until the path is re-published | Preview and publish a blank document at the old path, or wait for cache expiry |
| Properties appear empty in index | The `select` or `value` expression in `helix-query.yaml` does not match the metadata key | Verify the property name in the metadata table matches the YAML configuration exactly (case-sensitive) |
| Index returns fewer pages than expected | Default limit of 500 is too low | Add `?limit=1000` or implement pagination in consuming blocks |
| Named index not found | Index name in URL does not match `helix-query.yaml` key | Verify the index name — access it at `/query-index.json?sheet=<name>` |
