# GitHub GraphQL guide

Use GraphQL when one selection of related PR data, review threads, participants, or Projects v2 data avoids several REST calls. REST remains simpler for most single-resource writes and is the documented surface for Contents, Git objects, Actions, releases, checks, and many administrative operations. A GitHub Enterprise Server schema follows the server release and may lack GitHub.com fields or mutations. Recheck that host's schema before relying on a newer field; do not assume REST API version headers make GraphQL features available.

GitHub.com GraphQL: `https://api.github.com/graphql`. Enterprise Server: `https://HOST/api/graphql`. Enterprise Cloud data-residency hosts use their documented API hostname plus `/graphql`. Use `gh_request` and `gh_json_body` from [authentication and transport](authentication-and-transport.md), with the same chosen environment token. The envelope exposes HTTP status and selected headers; decoding a 2xx body is still only the first success check. Pass user values as GraphQL **variables**, never interpolate them into query source. REST `node_id` corresponds to GraphQL `id` for many resources; do not manufacture global IDs or assume their encoded form.

## Query and check a page

The following example returns one page of a PR's reviews. `GH_OWNER`, `GH_REPO`, `PR_NUMBER` (a validated integer), and `AFTER_CURSOR` are nonsecret input variables. Supply `null` for the first cursor and repeat while `hasNextPage` is true, passing the returned `endCursor` as the next `after` value. For multiple connections, paginate each independently or simplify the query.

```bash
query=$(cat <<'GQL'
query($owner: String!, $repo: String!, $number: Int!, $after: String) {
  repository(owner: $owner, name: $repo) {
    pullRequest(number: $number) {
      number
      url
      reviews(first: 50, after: $after) {
        nodes { id state author { login } submittedAt }
        pageInfo { hasNextPage endCursor }
      }
    }
  }
  rateLimit { cost remaining resetAt }
}
GQL
)
set -o pipefail
jq -n --arg query "$query" --arg owner "$GH_OWNER" --arg repo "$GH_REPO" \
  --argjson number "$PR_NUMBER" --arg after "${AFTER_CURSOR:-}" \
  '{query:$query, variables:{owner:$owner, repo:$repo, number:$number,
    after:(if $after == "" then null else $after end)}}' |
  gh_request "$GH_GRAPHQL" --request POST \
    --header 'Content-Type: application/json' --data-binary @- |
  gh_json_body |
  jq -e 'if ((.errors // []) | length) > 0 or .data == null
         then error("GraphQL request failed; inspect a sanitized error summary")
         else {pullRequest:.data.repository.pullRequest, rateLimit:.data.rateLimit}
         end'
```

`AFTER_CURSOR` is opaque: never decode or synthesize it. The example passes it as data with `jq --arg`; an unset or empty cursor becomes JSON `null` for the first page.

Connections require `first` or `last` from 1 to 100. Use smaller pages for expensive nested selections. Check `pageInfo.hasNextPage` and `endCursor` before declaring the list complete. A query can return partial `data` with top-level `errors` on HTTP 200; fail or explicitly label partial data rather than silently treating it as complete. For mutations, also inspect any mutation-specific error/result fields and confirm the changed resource. A missing `repository` or `pullRequest` may be lack of permission as well as absence. A GraphQL error message may contain user-provided text; summarize safely.

Monitor `rateLimit { cost remaining resetAt }` and response rate-limit headers. GraphQL primary limits are point-based, and secondary limits may also apply. A GraphQL rate-limit failure can arrive with HTTP 200 and top-level errors; use the same bounded backoff rules as REST. Limit query depth and node count rather than requesting many connections at maximum size.

Projects v2 has a useful GraphQL API for items, fields, and field values. Treat project writes as distinct, potentially organization-visible operations; check Projects permissions and the specific mutation schema for the host. Search through GraphQL is still subject to result and cost limits; the REST search cap and dedicated rate bucket make it unsuitable as a substitute for a complete repository listing. Auto-merge and merge-queue operations have GraphQL support on some hosts, but are elevated and version dependent; check the current schema and repository policy before use.

## Official sources

- [Forming GraphQL calls](https://docs.github.com/en/graphql/guides/forming-calls-with-graphql)
- [Enterprise Server GraphQL endpoint](https://docs.github.com/en/enterprise-server@3.18/graphql/guides/forming-calls-with-graphql)
- [GraphQL pagination](https://docs.github.com/en/graphql/guides/using-pagination-in-the-graphql-api)
- [GraphQL rate and query limits](https://docs.github.com/en/graphql/overview/rate-limits-and-query-limits-for-the-graphql-api)
- [Global node IDs](https://docs.github.com/en/graphql/guides/using-global-node-ids)
- [GraphQL schema reference](https://docs.github.com/en/graphql/reference)
- [Projects v2 API guide](https://docs.github.com/en/issues/planning-and-tracking-with-projects/automating-your-project/using-the-api-to-manage-projects)
