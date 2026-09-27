# Authentication, hosts, and safe transport

Checked against official GitHub documentation on 2026-09-27. Recheck the endpoint page and host-specific docs before an unusual write or after an API/version change.

## Resolve only a variable name

```bash
compgen -e | LC_ALL=C sort -u | grep -Ei '^(GH_TOKEN|GH_ENTERPRISE_TOKEN|GITHUB_([A-Z0-9]+_)*TOKEN)$'
```

`compgen -e` enumerates exported names without obtaining their values. The pattern intentionally excludes unrelated variables such as `GITHUB_REPOSITORY`, `GITHUB_API_URL`, and `GITHUB_TOKEN_EXPIRY`. An explicitly named nonmatching environment variable can be used if the user directs it; validate its shell name before access. Zero matches: stop and ask the user to export a least-privilege token into this session (for example, `GITHUB_TOKEN`) without pasting it into chat or a command. In an interactive Bash session, `read -r -s -p 'GitHub token: ' GITHUB_TOKEN; printf '\n'; export GITHUB_TOKEN` accepts it without echoing or putting the value in shell history. Do not persist it to a plaintext profile. One match: set `GH_TOKEN_VAR` to its name. Multiple matches: ask for the name unless the conversation already selects one. Never test several tokens in turn to pick a working one.

The resolved variable's **value** is read only at request time. These examples use `GH_TOKEN_VAR` for its name, `GH_API` for the REST base, `GH_GRAPHQL` for the GraphQL URL, and `GH_OWNER`/`GH_REPO` for the target. Do not assign the secret to a fixed `GITHUB_TOKEN` placeholder, print it, pass it on a command line, or save it. The request helper below keeps the Authorization header in a transient file descriptor. Its byte check rejects whitespace, control bytes, DEL, and non-ASCII bytes before embedding the token in a quoted curl-config value; quote and backslash escaping then follows [curl's config syntax](https://curl.se/docs/manpage.html#-K). This protects the transport boundary and does not identify a token type. Stop on an unsupported value rather than weakening that boundary. The helper returns one JSON object with numeric `status`, selected `headers`, and `body_b64` (base64 of the response body, so binary bodies remain representable). HTTP error responses still return this object; only transport or local failures return a nonzero shell status. Keep shell tracing off in the caller, and do not display the full object when the body or a signed download `Location` may be sensitive.

```bash
gh_request() (
  set +x
  set -o pipefail
  local LC_ALL=C
  local request_url=${1:?supply a vetted API URL}; shift
  [[ ${GH_API-} == https://* ]] || return 2
  local expected_graphql
  if [[ $GH_API == */api/v3 ]]; then
    expected_graphql=${GH_API%/v3}/graphql
  else
    expected_graphql=$GH_API/graphql
  fi
  [[ ${GH_GRAPHQL-} == "$expected_graphql" ]] || return 2
  [[ ${GH_API_VERSION-} =~ ^[0-9]{4}-[0-9]{2}-[0-9]{2}$ ]] || return 2
  [[ $request_url == "$GH_API/"* || $request_url == "${GH_GRAPHQL-}" ]] || return 2
  local accept='application/vnd.github+json'
  local -a request_args=()
  while (($#)); do
    case $1 in
      --request)
        [[ $# -ge 2 && $2 =~ ^(GET|HEAD|POST|PUT|PATCH|DELETE)$ ]] || return 2
        request_args+=(--request "$2"); shift 2 ;;
      --header)
        [[ $# -ge 2 && ! $2 =~ [^[:print:]] ]] || return 2
        case $2 in
          'Accept: '*) accept=${2#Accept: } ;;
          'Content-Type: '*|'If-None-Match: '*|'If-Modified-Since: '*)
            request_args+=(--header "$2") ;;
          *) return 2 ;;
        esac
        shift 2 ;;
      --data-binary)
        [[ $# -ge 2 && $2 == '@-' ]] || return 2
        request_args+=(--data-binary '@-'); shift 2 ;;
      *) return 2 ;;
    esac
  done
  local token_name=${GH_TOKEN_VAR:?select token variable name first}
  [[ $token_name =~ ^[A-Za-z_][A-Za-z0-9_]*$ ]] || return 2
  local token=${!token_name-}
  if [[ -z $token || $token =~ [^[:graph:]] ]]; then
    printf '%s\n' 'Selected GitHub token variable is empty or contains unsupported transport bytes.' >&2
    return 2
  fi
  # Under the C locale, graph is ASCII 0x21-0x7e. This checks the transport
  # boundary, not the token type; Bash variables cannot contain NUL.
  # Curl's quoted config values require backslash and quote escaping.
  local config_token=${token//\\/\\\\}
  config_token=${config_token//\"/\\\"}
  umask 077
  local response_dir status
  response_dir=$(mktemp -d) || return 1
  trap 'rm -f -- "$response_dir/headers" "$response_dir/body"; rmdir -- "$response_dir"' EXIT
  status=$(curl -q --silent --show-error --globoff --config <(
    printf 'header = "Authorization: Bearer %s"\n' "$config_token"
  ) --header "Accept: $accept" \
    --header "X-GitHub-Api-Version: ${GH_API_VERSION:?select version first}" \
    --dump-header "$response_dir/headers" --output "$response_dir/body" \
    --write-out '%{http_code}' "${request_args[@]}" --url "$request_url") || return 1
  [[ $status =~ ^[0-9]{3}$ ]] || return 1
  base64 < "$response_dir/body" | tr -d '\n' |
    jq -Rsc --argjson status "$status" \
      --rawfile response_headers "$response_dir/headers" '
        def h($name):
          ([$response_headers | split("\n")[]
            | select((ascii_downcase | startswith(($name | ascii_downcase) + ":")))
            | sub("^[^:]+:[ \t]*"; "") | sub("\r$"; "")][-1] // null);
        {status:$status,
         headers:{link:h("Link"), location:h("Location"), etag:h("ETag"), retry_after:h("Retry-After"),
           rate_remaining:h("X-RateLimit-Remaining"),
           rate_reset:h("X-RateLimit-Reset"),
           accepted_permissions:h("X-Accepted-GitHub-Permissions"),
           sso:h("X-GitHub-SSO"),
           enterprise_version:h("X-GitHub-Enterprise-Version")},
         body_b64:.}'
)

gh_json_body() (
  set -o pipefail
  jq -er 'if .status >= 200 and .status < 300 then .body_b64
          else error("GitHub HTTP " + (.status | tostring)) end' | base64 -d
)

gh_rest_pages() (
  set -o pipefail
  local next=${1:?supply a REST list URL} max_pages=${2:-1000} count=0 response status
  if [[ ! $max_pages =~ ^[1-9][0-9]*$ ]]; then
    printf '%s\n' '{"error":"invalid_input"}' >&2
    return 2
  fi
  if [[ ${GH_API-} != https://* ]]; then
    printf '%s\n' '{"error":"invalid_input"}' >&2
    return 2
  fi
  while [[ -n $next ]]; do
    if [[ $next != "$GH_API/"* ]]; then
      printf '%s\n' '{"error":"invalid_next_url"}' >&2
      return 2
    fi
    if ! ((++count <= max_pages)); then
      printf '%s\n' '{"error":"page_limit"}' >&2
      return 4
    fi
    response=$(gh_request "$next" 2>/dev/null) || {
      printf '%s\n' '{"error":"transport"}' >&2
      return 1
    }
    status=$(jq -er 'select((.headers | type) == "object") |
      .status | select(type == "number" and . >= 100 and . <= 599 and . == floor)' \
      <<< "$response" 2>/dev/null) || {
      printf '%s\n' '{"error":"parse"}' >&2
      return 5
    }
    if ((status < 200 || status >= 300)); then
      jq -c '{error:"http", status, headers:{retry_after:.headers.retry_after,
        rate_remaining:.headers.rate_remaining, rate_reset:.headers.rate_reset,
        accepted_permissions:.headers.accepted_permissions, sso:.headers.sso}}' \
        <<< "$response" >&2
      return 3
    fi
    printf '%s\n' "$response"
    next=$(jq -er '
      .headers.link as $link |
      if $link == null then ""
      elif ($link | type) != "string" then error("invalid Link header")
      else try ($link | capture("<(?<url>[^>]*)>;[[:space:]]*rel=\"next\"").url) catch ""
      end' <<< "$response" 2>/dev/null) || {
      printf '%s\n' '{"error":"parse"}' >&2
      return 5
    }
  done
)

gh_actions_download() (
  set +x
  set -o pipefail
  local LC_ALL=C
  local api_url=${1:?supply an Actions log or artifact URL}
  local action_path=${api_url#"${GH_API:?select API base first}/repos/${GH_OWNER:?select owner first}/${GH_REPO:?select repo first}/actions/"}
  local item_id response download_url authority config_url output download_status
  case $action_path in
    runs/*/logs) item_id=${action_path#runs/}; item_id=${item_id%/logs} ;;
    artifacts/*/zip) item_id=${action_path#artifacts/}; item_id=${item_id%/zip} ;;
    *) return 2 ;;
  esac
  [[ $api_url == "$GH_API/repos/$GH_OWNER/$GH_REPO/actions/"* && $item_id =~ ^[0-9]+$ ]] || return 2
  response=$(gh_request "$api_url") || return 1
  download_url=$(jq -er '
    if .status == 302 and (.headers.location | type == "string") and (.headers.location | length > 0)
    then .headers.location
    else error("Expected a signed Actions download redirect") end
  ' <<< "$response") || return 1
  [[ $download_url == https://* && ! $download_url =~ [^[:graph:]] ]] || return 2
  authority=${download_url#https://}
  authority=${authority%%[/?#]*}
  [[ -n $authority && $authority != *'@'* ]] || return 2
  # Keep the signed URL out of process arguments and diagnostic output.
  config_url=${download_url//\\/\\\\}
  config_url=${config_url//\"/\\\"}
  umask 077
  output=$(mktemp "${TMPDIR:-/tmp}/github-actions-download.XXXXXXXX") || return 1
  download_status=$(curl -q --silent --globoff --proto '=https' --max-redirs 0 \
    --config <(printf 'url = "%s"\n' "$config_url") \
    --output "$output" --write-out '%{http_code}' 2>/dev/null) || {
      rm -f -- "$output"
      printf '%s\n' 'Actions download failed; signed URL may have expired.' >&2
      return 1
    }
  if [[ ! $download_status =~ ^2[0-9]{2}$ ]]; then
    rm -f -- "$output"
    printf '%s\n' 'Actions download returned a non-success status.' >&2
    return 1
  fi
  printf '%s\n' "$output"
)
```

Use Bash, `curl`, `jq`, and `base64`. Set `set -o pipefail` in the calling shell before piping results. `gh_request` accepts only the listed request/header/body options, disallows authenticated redirects, and checks every URL against the selected API base. It also checks that `GH_GRAPHQL` is the documented endpoint derived from `GH_API`; a caller cannot silently point it at a different host. It creates a private temporary directory for response body and raw headers and removes it on exit; the token is never written there. The selected header fields are parsed inside `jq` and never passed as process arguments. The `-q` first curl option disables implicit curl configuration. Do not use `curl -v`, `--trace`, or a debug wrapper. A signed `Location` is sensitive: never display or persist the envelope from an Actions download redirect.

`gh_actions_download` handles the two-stage Actions download for the exact selected repository and a numeric run or artifact ID. It captures the authenticated `302` response privately, validates the signed HTTPS URL and its printable ASCII transport bytes, and fetches it immediately into a mode-600 temporary file using a separate curl invocation with **no Authorization header, API version header, redirect following, implicit curl configuration, or URL globbing**. The signed URL stays in memory and a transient file descriptor, not process arguments. A second redirect or non-2xx response fails closed; do not print the URL while diagnosing it. The function returns only the temporary file path; inspect the archive carefully because logs and artifacts can contain secrets, then delete that file. GitHub documents that these redirect URLs expire after one minute. If the download fails after expiry, request a fresh authenticated redirect and retry only after checking the failure; do not replay arbitrary writes.

```bash
if archive=$(gh_actions_download "$GH_API/repos/$GH_OWNER/$GH_REPO/actions/runs/$RUN_ID/logs"); then
  # Inspect only the needed entries without printing raw logs.
  unzip -Z -1 "$archive"
  rm -f -- "$archive"
fi

if archive=$(gh_actions_download "$GH_API/repos/$GH_OWNER/$GH_REPO/actions/artifacts/$ARTIFACT_ID/zip"); then
  # Inspect or extract into a private directory, then remove the archive.
  rm -f -- "$archive"
fi
```

Example after setting the nonsecret variables and defining `gh_request`:

```bash
set -o pipefail
gh_request "$GH_API/user" | gh_json_body | jq '{login, id, type}'
gh_request "$GH_API/repos/$GH_OWNER/$GH_REPO" | gh_json_body |
  jq '{full_name, private, default_branch, html_url, permissions}'
```

For a failed call, keep the envelope in the shell and inspect only `status`, the needed allowlisted headers, and a deliberately filtered error message. For example, `jq '{status, headers:{retry_after:.headers.retry_after, rate_remaining:.headers.rate_remaining, accepted_permissions:.headers.accepted_permissions, sso:.headers.sso, enterprise_version:.headers.enterprise_version}}'` selects useful diagnostics without showing the body or raw headers. A `304` is not an ordinary body success: use the previously cached body only when its ETag, URL, media type, host, and selected credential match. For pagination, `gh_rest_pages` emits one envelope per page. This example displays only issue identifiers and titles after every page succeeds:

```bash
set -o pipefail
if pages=$(gh_rest_pages "$GH_API/repos/$GH_OWNER/$GH_REPO/issues?per_page=100"); then
  while IFS= read -r page; do
    printf '%s\n' "$page" | gh_json_body | jq -c '.[] | {number, title}'
  done <<< "$pages"
else
  page_rc=$?
  printf 'Issue list is incomplete (pagination exit %d); see the sanitized error above.\n' "$page_rc" >&2
fi
```

The pagination helper emits one sanitized JSON error on stderr: `invalid_input` or `invalid_next_url` (exit 2), `transport` (exit 1), `http` with status and selected diagnostic headers (exit 3), `page_limit` (exit 4), or `parse` (exit 5). It never includes the response body, signed `Location`, or the rejected URL. If it stops early, label any collected results incomplete and report the failure class. Do not mistake a body-bearing HTTP error or `304` for a successful read.

## Installation-token verification

For a known GitHub App installation access token, do not report a human user. `GET /installation/repositories` is documented for that token and confirms the exact repository is accessible **after all pages are checked**; its response lists repositories and does not prove the app or installation account identity. Use GraphQL `viewer { login }` with the same selected token to obtain the app bot attribution, as shown in GitHub's [installation-authentication guide](https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app/authenticating-as-a-github-app-installation). Check HTTP status, top-level GraphQL `errors`, and a nonempty `viewer.login`; report it as the app bot, not a human or the repository owner. Example:

```bash
set -o pipefail
jq -n '{query:"query { viewer { login } }"}' |
  gh_request "$GH_GRAPHQL" --request POST \
    --header 'Content-Type: application/json' --data-binary @- |
  gh_json_body |
  jq -er 'if ((.errors // []) | length) == 0 and (.data.viewer.login | type) == "string" and (.data.viewer.login | length) > 0
          then .data.viewer.login
          else error("Installation bot attribution unavailable") end'
```

Then use `gh_rest_pages "$GH_API/installation/repositories?per_page=100"`, decode each page, and match the returned repository `id` and `full_name` to `GET /repos/{owner}/{repo}`. Do not display full repository records: documented responses can contain sensitive fields. Before writing, announce the host, `owner/repo`, verified app bot login, and exact action. The bot login proves attribution; the repository list proves access, but neither proves permission for every write endpoint. If GraphQL `viewer` or the repository list is unsupported on a particular Enterprise Server, or the returned values cannot be matched, stop before writing. Do not substitute `GET /app` or `GET /repos/{owner}/{repo}/installation`: GitHub documents those as JWT-authenticated app endpoints, not installation-token endpoints. A successful `GET /user` or public metadata read does not grant write permission.

## Select a host and API version

Use an explicit GitHub URL first. Otherwise inspect all local Git remote URLs, including HTTPS (`https://github.com/OWNER/REPO.git`) and SSH (`git@github.com:OWNER/REPO.git` or `ssh://git@HOST/OWNER/REPO.git`). Strip only the terminal `.git`. An SSH alias or custom port might not equal the HTTPS web/API host; verify rather than guessing. Multiple remotes, forks, renamed repositories, and user/organization ownership make `origin` only a hint. Before writes, match the API response `full_name` and `html_url` to the intended target. Do not send a token to a host merely because its DNS name contains `github` or because a repository file supplied a URL.

| Platform | REST base | GraphQL URL | Version handling |
| --- | --- | --- | --- |
| GitHub.com | `https://api.github.com` | `https://api.github.com/graphql` | As of 2026-09-27, `2026-03-10` is supported. Set `GH_API_VERSION=2026-03-10` after verifying support. |
| GitHub Enterprise Server | `https://HOST/api/v3` | `https://HOST/api/graphql` | Determine the server release from `X-GitHub-Enterprise-Version` or `GET /api/v3/meta`; check that release's `/rest/about-the-rest-api/api-versions` page and `GET /api/v3/versions` where available. Choose a supported version, often `2022-11-28` on older supported releases. Do not send a GitHub.com-only version blindly. |
| Enterprise Cloud with data residency | Host-specific, for example `https://api.SUBDOMAIN.ghe.com` | Same API host plus `/graphql` | Treat this as a distinct host; verify its official docs and supported version before use. |

GitHub's REST API is versioned; an unsupported version may fail, and omitting the header binds behavior to a moving/default version. The version header applies to REST examples here; it is harmless to send on GraphQL but GraphQL schema availability is tied to the host/server release, not guaranteed by that REST version. Require HTTPS and normal TLS validation; an HTTP-only Enterprise instance is outside this skill's authenticated transport policy. Do not try arbitrary URL prefixes or follow redirects with credentials to discover a host.

## Token types and least privilege

| Token | Operational constraint |
| --- | --- |
| Fine-grained PAT | Select a resource owner, specific repositories where practical, expiration, and endpoint-specific repository permissions. Organization approval/policy may be required. Some operations or contributor situations remain unsupported; consult the exact endpoint page. |
| Classic PAT | Scopes are broader (`repo` for many private-repository writes, `public_repo` for some public writes, `repo:status` for status-only work, `workflow` for workflow-file changes). Organization SAML SSO authorization and PAT policy may still block access. Avoid `repo` unless required. |
| GitHub App installation token | Restricted to installation repositories and permissions; expires after about one hour. It may identify an installation/bot rather than a user. This skill consumes an already exported token and does not create private keys/JWTs or rotate it. |
| GitHub App user token / OAuth token | Access is limited by both app grant and user/resource access; endpoint support varies. Verify identity and endpoint support. |
| Actions `GITHUB_TOKEN` | Exists only when exported into the task environment. Its permissions are controlled by workflow/repository policy and it normally serves the workflow repository; do not assume cross-repository access. |

Do not classify a token by its prefix, length, or environment variable name. Formats change; a token-shaped string proves neither identity nor authorization. Test against the chosen host with the selected token and inspect only safe fields. `X-OAuth-Scopes` and `X-Accepted-OAuth-Scopes` may help diagnose classic scopes; `X-Accepted-GitHub-Permissions` identifies accepted fine-grained permission sets for a REST endpoint. Treat these as diagnostics, not proof of effective access. `X-GitHub-SSO` can indicate a classic PAT needs SAML authorization. A `404` on private content may conceal lack of access; a `403` can also be policy or rate limiting. Never switch to another token or a stored `gh` login as a fallback.

## Official sources

- [Authenticating to the REST API](https://docs.github.com/en/rest/authentication/authenticating-to-the-rest-api)
- [Managing personal access tokens](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens)
- [Fine-grained token permissions by endpoint](https://docs.github.com/en/rest/authentication/permissions-required-for-fine-grained-personal-access-tokens)
- [GitHub App authentication](https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app/about-authentication-with-a-github-app)
- [GitHub App installation token generation and expiry](https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app/generating-an-installation-access-token-for-a-github-app)
- [GitHub App installation authentication and GraphQL viewer](https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app/authenticating-as-a-github-app-installation)
- [Installation repository list](https://docs.github.com/en/rest/apps/installations)
- [App endpoints requiring a JWT](https://docs.github.com/en/rest/apps/apps)
- [Use `GITHUB_TOKEN` in workflows](https://docs.github.com/en/actions/tutorials/authenticate-with-github_token)
- [REST API versions](https://docs.github.com/en/rest/about-the-rest-api/api-versions)
- [Enterprise Server REST versions](https://docs.github.com/en/enterprise-server@3.21/rest/about-the-rest-api/api-versions)
- [Enterprise Server GraphQL endpoint](https://docs.github.com/en/enterprise-server@3.18/graphql/guides/forming-calls-with-graphql)
- [Enterprise Server REST base and release header](https://docs.github.com/en/enterprise-server@3.18/rest/enterprise-admin)
- [REST troubleshooting](https://docs.github.com/en/rest/using-the-rest-api/troubleshooting-the-rest-api)
- [GitHub CLI environment variable names, including `GH_ENTERPRISE_TOKEN`](https://cli.github.com/manual/gh_help_environment)
- [curl config quoting and default config behavior](https://curl.se/docs/manpage.html#-K)
