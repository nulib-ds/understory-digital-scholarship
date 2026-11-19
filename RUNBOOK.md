# Runbook: replacing production-understory

`understory-digital-scholarship` replaces `production-understory`, in the same
AWS account (production). It is restored from production's 2026-11-17 backup,
then takes over production's three hostnames:

| Host | Serves |
|---|---|
| `iiif-understory.dc.library.northwestern.edu` | IIIF documents and public search |
| `images-understory.dc.library.northwestern.edu` | the Image API |
| `admin-understory.dc.library.northwestern.edu` | the admin UI |

Every IIIF id embeds its host, so with the same hosts the restored documents
need no rewriting, and every Canopy site's links keep working.

Run each step from this repo's folder. Steps 1 to 5 change nothing public.
Step 6 takes the public hostnames down for about half an hour.

## 0. Before you start

- **AWSAdministratorAccess in the production account.** The stack creates IAM
  roles, which PowerUser cannot. Use that profile in every command below
  (`export AWS_PROFILE=<your production admin profile>`, after `aws sso login`).
- **The SAM CLI.** No Docker or Node: nothing is built here.
- **Releases are private to NU's accounts for now,** so read them with your
  credentials:

  ```bash
  export UNDERSTORY_RELEASES='s3://understory-releases-{region}'
  ```

- **The backup's location.** Mat has it; it is not written down here.

## 1. Check the release

```bash
scripts/verify
```

It should say `understory.yml is Understory v1.0.1, unmodified.`

## 2. Name yourself the first admin

Set `AdminEmail` in `samconfig.yaml` to your own address, locally, and **do
not commit it**: a person's address does not belong in the repo, which may be
public. Cognito emails it a temporary password when step 3 creates the stack.
`AdminEmail` is read only then, so set it back to `""` right after step 3.

## 3. Create the stack, without a domain

```bash
sam deploy
```

Review the change set and accept it. It takes 20 to 30 minutes, mostly
CloudFront. The stack starts on generated `*.cloudfront.net` names, because
production still holds the real ones.

**Do not edit anything in it yet.** Until step 6 its IIIF base is its
`cloudfront.net` name, and a save would write that into the documents.

## 4. Restore the backup: IIIF bucket first

```bash
BACKUP='<the backup prefix, ending in /understory/2026-11-17>'
NEW=understory-digital-scholarship

aws s3 sync "$BACKUP/production-understory-iiif/" "s3://$NEW-iiif/" \
  --exclude 'internal/publish/*' --exclude 'internal/collection-import/*'

aws s3 sync "$BACKUP/production-understory-source/" "s3://$NEW-source/"
```

- **The order matters.** Copying an original into the source bucket fires the
  converters, which skip any image whose TIFF, and any video whose `ready`
  `media.json`, is already in the IIIF bucket. IIIF first, so nothing is
  converted or transcoded again.
- **The two excludes** are production's publish runs and its search aliases.
  They name indexes this stack's search collection does not have.

Then check nothing was converted. Both commands should print nothing:

```bash
aws logs filter-log-events --log-group-name "/aws/lambda/$NEW-iiif-image" \
  --filter-pattern '"Done:"' --query 'events[].message' --output text
aws logs filter-log-events --log-group-name "/aws/lambda/$NEW-av-transcode" \
  --filter-pattern '"Submitted MediaConvert job"' --query 'events[].message' --output text
```

## 5. Prepare the swap, while production is still live

1. Agree a window, and ask curators to stop editing in production.
2. Copy anything changed since the backup, straight from production:

   ```bash
   aws s3 sync s3://production-understory-iiif/ "s3://$NEW-iiif/" \
     --exclude 'internal/publish/*' --exclude 'internal/collection-import/*'
   aws s3 sync s3://production-understory-source/ "s3://$NEW-source/"
   ```

3. List production's users and their groups, to invite them again in step 7.
   Keep the list out of this repo: it is personal data.

   ```bash
   POOL=$(aws cloudformation describe-stacks --stack-name production-understory \
     --query "Stacks[0].Outputs[?OutputKey=='CognitoUserPoolId'].OutputValue" --output text)
   for g in $(aws cognito-idp list-groups --user-pool-id "$POOL" --query 'Groups[].GroupName' --output text); do
     echo "$g:"; aws cognito-idp list-users-in-group --user-pool-id "$POOL" --group-name "$g" \
       --query 'Users[].Attributes[?Name==`email`].Value' --output text
   done
   ```

4. Note production's `CertificateArn` and `HostedZoneId`, for step 6:

   ```bash
   aws cloudformation describe-stacks --stack-name production-understory \
     --query "Stacks[0].Parameters[?ParameterKey=='CertificateArn' || ParameterKey=='HostedZoneId']" --output table
   ```

## 6. The swap

Public downtime starts here.

1. **Release production's hostnames.** Update production in place, with its own
   template, clearing only its domain. Its distributions drop the names, and
   its DNS records and Amplify domain go:

   ```bash
   aws cloudformation describe-stacks --stack-name production-understory \
     --query 'Stacks[0].Parameters[].ParameterKey' --output json \
   | python3 -c 'import json,sys; print(json.dumps([{"ParameterKey": k, "ParameterValue": ""} if k == "BaseDomainName" else {"ParameterKey": k, "UsePreviousValue": True} for k in json.load(sys.stdin)]))' \
     > /tmp/production-params.json

   aws cloudformation update-stack --stack-name production-understory --use-previous-template \
     --capabilities CAPABILITY_IAM CAPABILITY_AUTO_EXPAND --parameters file:///tmp/production-params.json
   aws cloudformation wait stack-update-complete --stack-name production-understory
   ```

2. **Check the names are free.** None of the three hostnames should have a
   record left in the zone:

   ```bash
   aws route53 list-resource-record-sets --hosted-zone-id '<HostedZoneId from step 5>' \
     --query "ResourceRecordSets[?contains(Name, '-understory.dc.')].[Name, Type]" --output text
   ```

3. **Take them.** Deploy with a temporary config that adds the domain.
   `samconfig.swap.yaml` is gitignored; delete it afterwards.

   ```bash
   sed -e 's|- BaseDomainName=""|- BaseDomainName=dc.library.northwestern.edu|' \
       -e 's|- CertificateArn=""|- CertificateArn=<CertificateArn from step 5>|' \
       -e 's|- HostedZoneId=""|- HostedZoneId=<HostedZoneId from step 5>|' \
       samconfig.yaml > samconfig.swap.yaml
   sam deploy --config-file samconfig.swap.yaml
   rm samconfig.swap.yaml
   ```

4. **Check.** The documents, an image's `info.json` and the admin UI:

   ```bash
   curl -s https://iiif-understory.dc.library.northwestern.edu/working/presentation/collection/index/collection.json | head -c 300
   curl -s -o /dev/null -w '%{http_code}\n' https://admin-understory.dc.library.northwestern.edu/
   ```

## 7. After the swap

1. **Commit the domain.** In `samconfig.yaml`, set
   `BaseDomainName=dc.library.northwestern.edu` and delete the `CertificateArn`
   and `HostedZoneId` lines: the stack keeps the values step 6 gave it, and
   they identify the account, which this repo does not.
2. **Sign in** at https://admin-understory.dc.library.northwestern.edu with
   step 2's temporary password.
3. **Rebuild search.** For each collection, open Share: **Publish IIIF
   assets**, then **Publish search index**. Until then that collection's public
   search answers 404.
4. **Invite everyone** from step 5's list on the Users screen, and give back
   their roles and collection grants.

## 8. Retire production-understory

When the new instance has run for a while:

```bash
aws cloudformation delete-stack --stack-name production-understory
```

Its two data buckets are retained by design, as one more copy of the data.
Delete them deliberately, later. The `understory-github-actions-production`
stack, its deploy role, can go with it.

## 9. Optional: deploy from GitHub

1. Deploy `github-deploy-role.yml` in the production account, with
   `GitHubRepository=nulib-ds/understory-digital-scholarship`.
2. In this repo, create a `production` environment allowing only `main`. If
   one exists already, made by an old run of the workflow, it has no branch
   rule: add the rule before going on.
3. Set the repository secret `AWS_DEPLOY_ROLE_ARN` (that stack's output; a
   secret because it carries the account id), and the variable
   `AWS_REGION=us-east-1`. `UNDERSTORY_RELEASES=s3://understory-releases-{region}`
   is set already, so that pull requests skip the anonymous release check
   while the release bucket is private.

Every push to `main` then deploys, after `scripts/verify`. Until `AWS_REGION`
is set, the deploy job is skipped.
