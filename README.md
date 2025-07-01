# XXX - [deps-edn] Incorrect resolution of S3 repository URL when there are query params

## Current behavior

On Clojure's [tools.deps](https://github.com/clojure/tools.deps), configuring the AWS region for the maven repository S3 bucket through query params is an acceptable approach (as seen [here](https://github.com/clojure/tools.deps/blob/87a170f36da934a8ab2af8cebc1dbf2b65e6dc4d/src/main/clojure/clojure/tools/deps/util/s3_transporter.clj#L65) and [here](https://www.reddit.com/r/Clojure/comments/eputbv/toolsdepsalpha_08264_clj_1101496_now_available/)). This region is used for downloading the project dependencies.

However, when the region query param is present, Renovate solves the URL incorrectly. In this case, for example, it searches for the packages on the `s3://ipim-test-maven/ipim/commons/dynamodb-local/` path instead of the `s3://ipim-test-maven/release/ipim/commons/dynamodb-local/` one as it should.

Reading the Renovate source code, it is possible to see that the problem happens exactly at the [`getMavenUrl`](https://github.com/renovatebot/renovate/blob/main/lib/modules/datasource/maven/util.ts#L279-L288) function:

```ts
> new URL('ipim/commons/dynamodb-local/maven-metadata.xml', 's3://ipim-test-maven/release?region=us-east-2/') // Renovate adds a trailing slash because URL requires it
URL {
  href: 's3://ipim-test-maven/ipim/commons/dynamodb-local/maven-metadata.xml',
  origin: 'null',
  protocol: 's3:',
  username: '',
  password: '',
  host: 'ipim-test-maven',
  hostname: 'ipim-test-maven',
  port: '',
  pathname: '/ipim/commons/dynamodb-local/maven-metadata.xml',
  search: '',
  searchParams: URLSearchParams {},
  hash: ''
}
```

### Debug Logs

The Renovate debug logs show the incorrect path being used.

```
DEBUG: Looking up ipim.commons:dynamodb-local in repository s3://ipim-test-maven/release?region=us-east-2/ (repository=isapim/renovate-deps-edn)
DEBUG: Maven S3 lookup error: unknown error (repository=isapim/renovate-deps-edn)
       "failedUrl": "s3://ipim-test-maven/ipim/commons/dynamodb-local/maven-metadata.xml",
       "err": {
         "name": "NoSuchKey",
         "$fault": "client",
         "$metadata": {
           "httpStatusCode": 404,
           "requestId": "CNETZVCK79V86ZC8",
           "extendedRequestId": "E3rD0p7gwQlWvQmJa36S53uFKEKXhVdoW055gYqDc5WyGU7+afy0UiXz35Gii7yCXnEFoeCnlq5n4zoEYEExC/Qy99wZNXyTXUW0DFBMO/Q=",
           "attempts": 1,
           "totalRetryDelay": 0
         },
         "Code": "NoSuchKey",
         "Key": "ipim/commons/dynamodb-local/maven-metadata.xml",
         "RequestId": "CNETZVCK79V86ZC8",
         "HostId": "E3rD0p7gwQlWvQmJa36S53uFKEKXhVdoW055gYqDc5WyGU7+afy0UiXz35Gii7yCXnEFoeCnlq5n4zoEYEExC/Qy99wZNXyTXUW0DFBMO/Q=",
         "message": "The specified key does not exist.",
         "stack": "NoSuchKey: The specified key does not exist.
```


### Self-hosted Renovate Config

For test purposes, we're running self-hosted Renovate with the following config:

```
module.exports = {
  branchPrefix: 'renovate/',
  username: 'renovate-release',
  gitAuthor: 'Renovate Bot <bot@renovateapp.com>',
  onboarding: true,
  dryRun: 'full',
  platform: 'github',
  repositories: ['isapim/renovate-deps-edn'],
};
```

And the Github action step is configured as such:

```
- name: Run Renovate
  uses: renovatebot/github-action@v43.0.1
  with:
    renovate-version: 41.6.2
    configurationFile: renovate-config.js
    token: ${{ secrets.RENOVATE_TOKEN }}
    env-regex: ^(?:RENOVATE_\w+|LOG_LEVEL|AWS_\w+)$
  env:
    AWS_ACCESS_KEY_ID: ${{ env.AWS_ACCESS_KEY_ID }}
    AWS_SECRET_ACCESS_KEY: ${{ env.AWS_SECRET_ACCESS_KEY }}
    AWS_SESSION_TOKEN: ${{ env.AWS_SESSION_TOKEN }}
    AWS_REGION: ${{ env.AWS_REGION }}
    LOG_LEVEL: debug
```

## Expected behavior

Look for the packages in the correct S3 path even when the repository region is configured via query param in the `deps.edn`.

## Disclaimers

The bucket `ipim-test-maven` is public, but AWS credentials are still necessary to run Renovate and to download the clojure dependencies.

The dependency `dynamodb-local` is not of my authorship, it is from https://github.com/dmcgillen/clj-dynamodb-local. I merely put it into a public s3 with another group to illustrate the pathing problem and will delete it as soon as possible.

## Link to the Renovate issue or Discussion

Put your link to the Renovate issue or Discussion here.
