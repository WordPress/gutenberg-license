# Readme

This repository holds the information relevant to the requests for consent from [Gutenberg's](https://github.com/WordPress/gutenberg) contributors to dual-license their past contributions to Gutenberg under the GPL and MPL (more information available here). For more information, you can look at the [initial discussion issue on GitHub](https://github.com/WordPress/gutenberg/issues/23651) as well as the [initial proposal post](https://make.wordpress.org/core/2020/09/21/proposal-dual-licensing-gutenberg-under-gpl-v2-0-and-mpl-v2-0/) on 21 September 2020, and [updated proposal post](https://make.wordpress.org/core/2021/03/05/dual-licensing-gutenberg-next-steps/) on Make/Core.

# Format of `dual-license-responses.json`

The [`dual-license-responses.json`](data/dual-license-responses.json) file records responses relevant to the work to update Gutenberg to be dual-licensed from (#31893)[https://github.com/WordPress/gutenberg/issues/31893] and (#31913)[https://github.com/WordPress/gutenberg/issues/31913].

The json has two top level nodes. The `claimedEmails` node includes an object in `claimedEmails.responses` for each email address that has contributed to Gutenberg's `trunk` branch (i.e., by being an author or coauther of a commit on that branch) but was not associated with any GitHub account. 

The `gitHubUserContributors` node includes an object in `gitHubUserContributors.responses` for each GitHub user who has contributed to Gutenberg's `trunk` branch. Once a grant or denial of consent is received for a particular contributor, the relevant `gitHubUserContributors.responses` object will be updated with a `consent` boolean, with `true` representing a grant of consent to dual-licensing the user's past contributions and `false` representing a denial of consent. If there is no `consent` field for a particular `gitHubUserContributions.responses` object, that means we have not received either a grant or denial of consent.

When a relevant comment is noted for either the `claimedEmails` or `gitHubUserContributors`, information about that comment is recorded in the `comment` field on the relevant node.

# Checking the Latest Responses

Execute `node scripts/update-dual-license-responses.js` to update [`dual-license-responses.json`](data/dual-license-responses.json) with the latest comments on the relevant GitHub issues. This script has been tested with node v14.16.0. In order to run this script, you must have the [GitHub CLI](https://cli.github.com/) installed and configured (the GitHub CLI handled pagination of the search results). 

Running that script will download all the comments on each issue, allow the user to process each new comment individually (previously processed comments are skipped), and update the [`dual-license-responses.json`](data/dual-license-responses.json) file appropriately based on the user's input.

This script will automatically run [`summarize-dual-license-responses.js`](scripts/summarize-dual-license-responses.js), which prints out some current status of the requests for consent and updates the [`status.md`](status.md) file.

# How Was `dual-license-responses.json` Initialized?

The [`dual-license-responses.json`](dual-license-responses.json) json file was initialized with:
1. All the GitHub users who have contributed to Gutenberg; and 
2. All of the email addresses that authored a commit on Gutenberg's `trunk` branch but was not associated with any GitHub account. We want to identify the GitHub users behind these commits and ask those GitHub users to consent to dual-license their past contributions.

## How Gutenberg’s Contributors Were Identified

This data was collected on April 16, 2021

The easiest way to execute the GraphQL queries listed below is by using the [GitHub CLI](https://cli.github.com/) to execute a command along these lines `gh api graphql --paginate -f query='query(...'`, which will handle authentication and pagination for you.

### Contributors With GitHub Logins

#### All Merged and Open PRs

Saved as [2021-04-16_merged-or-open-prs.json](data/2021-04-16_merged-or-open-prs.json)

<details>
  <summary>GraphQL query</summary>
  
```
query($endCursor: String) {
  repository(owner: "WordPress", name: "gutenberg") {
    pullRequests(states:[MERGED, OPEN] first: 100, after: $endCursor) {
      totalCount
      pageInfo {
        endCursor
        hasNextPage
      }
      nodes {
        state
        number
        author {
          login
        }
        mergedBy {
          login
        }
      }
    }
  }
}'
```

Note that this query returns _all_ PRs through the date the script is run.
</details>

Although this query captures logins of both the "author" and the person that merged each PR, the data shows that there is no one who has merged a PR who has not also authored at least one PR. In other words, including logins from the "mergedBy" field does not add any additional contributors beyond those captured by the "author" field.

The authors of the PRs were extracted by running 

```
cat 2021-04-16_merged-or-open-prs.json \
  | jq --raw-output '.data.repository.pullRequests.nodes[].author.login' \
  | sort \
  | uniq
```

#### All Authors and Co-Authors of commits on trunk

Saved as [all_commits_before_2021-04-16.json](data/all_commits_before_2021-04-16.json) with 743 unique logins.

<details>
<summary>GraphQL query</summary>
  
```
query($endCursor: String) {
  repository(owner: "WordPress", name: "gutenberg") {
    object(expression:"trunk") {
      ... on Commit {
        history(first:100 after:$endCursor, until:"2021-04-16T00:00:00") {
          pageInfo {
            hasNextPage
            endCursor
          }
          nodes {
            committedDate
            oid
            author {
              email
              name
              user {
                login
              }
            }
            authors(first:100) {
              pageInfo {
                hasNextPage
              }
              nodes {
                email
                name
                user {
                  login
                }
              }
            }
          }
        }
      }
    }
  }
}'
``` 
</details>

Note that although that query only retrieves the first 100 "authors" on each commit (and does not paginate on that if there are more), I confirmed that there were not more than 100 authors on any commit (the highest was actually 8):

```
$ cat all_commits_before_2021-04-16.json \
    | jq '.data.repository.object.history.nodes[].authors.pageInfo.hasNextPage' \
    | sort \
    | uniq
false
```

The list of all "authors" on all these PR included every "author" returned here, so we are fine just including everyone listed as in the "authors" object in the above response:

```
cat all_commits_before_2021-04-16.json \
  | jq --raw-output '.data.repository.object.history.nodes[].authors.nodes[].user.login' \
  | sort \
  | uniq
```

#### Resulting data

All contributors for which we have GitHub logins were determined by combining the users with commits on `trunk` with the users who had opened or merged any Gutenberg PR.

These users were used to initialize the `gitHubUserContributors.responses` array in [`dual-license-responses.json`](data/dual-license-responses.json).

### Contributors Without An Associated GitHub Account

These users were exacted from the "co-authors" of any PRs on trunk (every commit "author" was coverered by the following "coauthors" query), where the email did not have an associated github account (the "user" was null).

```
cat 2021-04-16_authors-and-coauthors-on-trunk.json \
  | jq '.data.repository.object.history.nodes[].authors.nodes[]
         | select(.user.login == null)
         | .email' --raw-output \
  | sort \
  | uniq
```

These emails were used to initialize the `claimedEmails.responses` array in [`dual-license-responses.json`](data/dual-license-responses.json).
