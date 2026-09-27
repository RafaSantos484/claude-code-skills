# Repository contents and Git objects

Use `GH_API`, `GH_OWNER`, `GH_REPO`, and `gh_request` from [authentication and transport](authentication-and-transport.md). Choose a branch, tag, or full commit SHA explicitly when reproducibility matters. Require Contents read to inspect private code and Contents write for content mutations. Writing workflow files can also require Workflows write (or the classic PAT `workflow` scope); check the exact endpoint and token type. Repository rulesets, branch protection, and required reviews may reject otherwise authorized writes.

## Read

`GET /repos/{owner}/{repo}/contents/{path}?ref={branch|tag|sha}` returns file or directory metadata and, for ordinary small files, base64 content. Percent-encode path and `ref` as URL components; do not use raw user text as a path or query. The default ref is the repository default branch, which can change. Directory responses are limited to 1,000 entries; use the Trees API for larger directories. `GET /repos/{owner}/{repo}/git/trees/{sha}?recursive=1` can itself be `truncated`; if so, traverse subtrees instead of reporting a complete inventory.

For files up to 1 MB, normal Contents responses support base64. From 1 to 100 MB, use the documented raw media type or object media type (whose `content` is empty); above 100 MB the Contents endpoint is unsupported. `download_url` expires and is not a durable link. A symlink may resolve to a normal file or return a symlink object. A submodule may be represented with `type: "file"` for compatibility; inspect `submodule_git_url` and gitlink mode rather than assuming it is a file. Git LFS content in a Git tree is normally a pointer, not the large object itself; do not claim to have inspected the LFS payload. Avoid writing an LFS pointer accidentally when the requested change concerns the binary object.

## One file: Contents API

`PUT /repos/{owner}/{repo}/contents/{path}` accepts a commit `message`, base64 `content`, optional `branch`, and for **update** the current file blob `sha`. A new file omits `sha`. `DELETE` requires the current file `sha`, message, and optional branch. Read the file and branch tip first; preserve path, content encoding, line endings, and intended branch. After success, check the returned `content`/`commit` SHA and URL. Serializing these operations is necessary; GitHub documents conflicts when Contents create/update and delete are run concurrently. If the SHA is stale or the branch changed, stop, re-read, compare, and rebuild the intended change. Do not blindly retry. File deletion is elevated when hard to reverse or part of a broader destructive action.

## Several files: one Git commit

When a change should be atomic, do not issue independent Contents writes. The [Git database guide](https://docs.github.com/en/rest/guides/using-the-rest-api-to-interact-with-your-git-database) documents this sequence:

1. Read `GET /git/ref/heads/{branch}` to record the current tip SHA. Read that commit and its tree SHA.
2. Create changed blobs with `POST /git/blobs` (or supply documented inline content to tree entries). Do not use this API for large binary/LFS payloads without a separate, researched LFS workflow.
3. Create **one** tree with `POST /git/trees`, using the current `base_tree` and all intended entries. Set correct path, mode (`100644`, `100755`, `120000` for symlink, `160000` for submodule), type, and blob/commit SHA or content. A `null` SHA deletes an entry. Avoid conflicting parent and nested paths; the Trees API can overwrite nested structure in that case.
4. Create **one** commit with `POST /git/commits`, using the new tree and the recorded tip as its sole parent. Inspect the commit object before moving a ref.
5. Re-read the branch tip. If it changed, stop and rebuild on the new parent/tree after reviewing the concurrent change. Otherwise `PATCH /git/refs/heads/{branch}` to the new commit SHA with `force:false`. GitHub's default non-force behavior rejects non-fast-forward movement. Verify the ref now points to the expected commit and return a commit URL.

The objects created in steps 2–4 are not visible as branch history until the single ref update; this gives one atomic multi-file commit at the branch pointer. It is not a multi-request transaction or a compare-and-swap guarantee. Re-reading before update plus `force:false` prevents overwriting divergent work; a concurrent fast-forward can still require a fresh comparison. Never use `force:true` for convenience. Force-updating a ref is elevated and requires explicit authorization of the exact branch, old/new SHAs, and lost-history impact. Deleting a branch or tag ref is also elevated. For a new branch, create `refs/heads/{branch}` from a verified base commit; make sure it does not already exist and report the returned ref. For tags, distinguish lightweight refs from annotated tag objects and releases.

## Official sources

- [Repository Contents API](https://docs.github.com/en/rest/repos/contents)
- [Git blobs](https://docs.github.com/en/rest/git/blobs), [trees](https://docs.github.com/en/rest/git/trees), [commits](https://docs.github.com/en/rest/git/commits), [references](https://docs.github.com/en/rest/git/refs), [tags](https://docs.github.com/en/rest/git/tags)
- [Git database workflow](https://docs.github.com/en/rest/guides/using-the-rest-api-to-interact-with-your-git-database)
- [Git LFS overview](https://docs.github.com/en/repositories/working-with-files/managing-large-files/about-git-large-file-storage)
- [Repository rulesets](https://docs.github.com/en/rest/repos/rules), [branch protection](https://docs.github.com/en/rest/branches/branch-protection)
