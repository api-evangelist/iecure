---
name: iecure-resolve-team-and-leadership
description: >-
  Resolve iECURE's people — leadership, board of directors, advisors and staff — from the company's
  WordPress REST API, where they are carried by a `portfolio` post type rather than a person type.
api: iecure:iecure-team-api
operations: [listTeamGroups, listTeamMembers, getTeamMember, listAuthors]
---

# Resolve iECURE people

The single non-obvious fact about this deployment: **iECURE's team members live in the `portfolio`
custom post type.** The Avada theme's portfolio feature was repurposed to carry people, so the REST
collection is `/wp/v2/portfolio` and the classifying taxonomy is `/wp/v2/portfolio_entries`, whose
terms resolve to public URLs under `/team_members/`. An agent looking for `/wp/v2/people` or
`/wp/v2/team` will find nothing and wrongly conclude the data is absent.

Base URL: `https://iecure.com/wp-json`. No credentials.

## 1. Get the groups

    GET /wp/v2/portfolio_entries?per_page=100

Terms observed 2026-08-22, with counts: `team` (35), `teammembers` (21), `bod` (6), `leadership` (6),
`advisors` (3). The taxonomy is hierarchical and the terms overlap — `team` is the umbrella, so a
person appears under both `team` and their specific group. Do not sum the counts to get a headcount.

## 2. List people in a group

    GET /wp/v2/portfolio?portfolio_entries={term_id}&per_page=100&orderby=menu_order&order=asc&_fields=id,slug,link,title,portfolio_entries,featured_media

`menu_order` is the ordering the site itself renders, which for leadership rosters is usually
seniority rather than alphabetical. Prefer it over `date`.

## 3. Fetch a person with their headshot

    GET /wp/v2/portfolio/{id}?_embed

`title.rendered` is the person's name and credentials as displayed; `content.rendered` is their bio
as HTML. `_embed` inlines the featured media so you get the headshot `source_url` without a second
call to `/wp/v2/media/{id}`.

## 4. Cross-check against the About page

    GET /wp/v2/pages?slug=about&_fields=id,link,content

The public `/about/` page carries anchors for `#leadership`, `#board-of-directors`, `#advisors`,
`#team`, `#investors` and `#our_story`. The `#investors` section is rendered page content only — it
is **not** exposed as structured data anywhere in the API, so investor names must be read from the
page HTML and attributed to it, not to a collection.

## 5. Authors are a different, smaller set

    GET /wp/v2/users?per_page=100

Only two accounts have published content, and they are site editors — not the leadership roster.
Never conflate `/wp/v2/users` with `/wp/v2/portfolio`. (This collection being anonymously readable at
all is unusual for WordPress; most deployments return 401 here.)

## Rules

- **Read only.** No public write path exists.
- **`per_page` max 100**, else `400 rest_invalid_param`.
- **Titles and bios are HTML.** `title.rendered` contains entities (`&amp;`, `&#8217;`) and bios
  contain markup. Decode and strip before use.
- **People data is personal data.** These are named individuals on a company website. Use it to
  understand the company; do not build a contact list from it. Per API Evangelist's enrichment PII
  guardrail, do not extract or store personal contact details.
- **Counts are a snapshot** taken 2026-08-22, not a contract.
