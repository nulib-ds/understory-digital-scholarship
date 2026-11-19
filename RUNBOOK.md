# Runbook: replacing production-understory

> **Done on 2026-11-19.** `understory-digital-scholarship`, on v1.0.1, took
> over production's three hostnames that day. Steps 1 to 7 are written as
> they were run, with what the run taught; step 8 is still to come.

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
- **A version manager may refuse `sam` and `python3` in this folder.** asdf
  looks for a `.tool-versions`, this repo has none, and it answers "No version
  is set for command sam". Name the version for the shell instead
  (`export ASDF_AWS_SAM_CLI_VERSION=<an installed version>`), and call
  `/usr/bin/python3` in step 6.
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
not commit it**: a person's address does not belong in the repo, which is
public. Cognito emails it a temporary password when step 3 creates the stack.
`AdminEmail` is read only then, so set it back to `""` right after step 3
(`git checkout samconfig.yaml`).

The temporary password lasts seven days. If step 7 comes later than that,
send a new one:

```bash
POOL=$(aws cloudformation describe-stacks --stack-name understory-digital-scholarship \
  --query "Stacks[0].Outputs[?OutputKey=='CognitoUserPoolId'].OutputValue" --output text)
aws cognito-idp admin-create-user --user-pool-id "$POOL" --username '<your address>' --message-action RESEND
```

## 3. Create the stack, without a domain

```bash
sam deploy
```

Review the change set: every row is an Add, and none names a DNS record.
Accept it. It takes 20 to 30 minutes, mostly CloudFront. The stack starts on
generated `*.cloudfront.net` names, because production still holds the real
ones.

- **Do not sign in yet, let alone edit.** Until step 6 the stack's IIIF base is
  its `cloudfront.net` name. A save writes that into the documents, and even
  the admin's home page, before step 4, creates the root collection with it.
- **A failed create leaves nothing behind.** The data buckets are
  `RetainExceptOnCreate`, so the rollback deletes them too. Delete the
  `ROLLBACK_COMPLETE` stack and deploy again.

## 4. Restore the backup: IIIF bucket first

```bash
BACKUP='<the backup prefix, ending in /understory/2026-11-17>'
NEW=understory-digital-scholarship

aws s3 sync "$BACKUP/production-understory-iiif/" "s3://$NEW-iiif/" \
  --exclude 'internal/publish/*' --exclude 'internal/collection-import/*' --only-show-errors

# Complete when this prints 0.
aws s3 sync --dryrun "$BACKUP/production-understory-iiif/" "s3://$NEW-iiif/" \
  --exclude 'internal/publish/*' --exclude 'internal/collection-import/*' | wc -l
```

- **The two excludes** are production's publish runs and its search aliases.
  They name indexes this stack's search collection does not have.
- **The source bucket is optional, and NU's run left it out.** It holds the
  original uploads. Serving reads only the IIIF bucket, which holds the
  converted TIFFs and A/V renditions, and NU keeps no originals: this is a
  presentation system, not a preservation one. Without them, an image cannot
  be converted again from its original, and an A/V upload never attached to a
  canvas drops off its work's recovery list.

To bring the originals over, now or later, copy them after the IIIF bucket,
never before:

```bash
aws s3 sync "$BACKUP/production-understory-source/" "s3://$NEW-source/" --only-show-errors
```

Copying an original into the source bucket fires the converters, which skip
any image whose TIFF, and any video whose `ready` `media.json`, is already in
the IIIF bucket. Then check nothing was converted. Both commands should print
nothing; a `ResourceNotFoundException` means that converter never ran, which
is as good:

```bash
aws logs filter-log-events --log-group-name "/aws/lambda/$NEW-iiif-image" \
  --filter-pattern '"Done:"' --query 'events[].message' --output text
aws logs filter-log-events --log-group-name "/aws/lambda/$NEW-av-transcode" \
  --filter-pattern '"Submitted MediaConvert job"' --query 'events[].message' --output text
```

## 5. Prepare the swap, while production is still live

1. Agree a window, and ask curators to stop editing in production. NU's
   curators stopped at midday on 2026-11-17, the day of the backup.
2. **Copy anything production changed since the backup.** Do not lean on
   `aws s3 sync` for this. It skips a same-sized file unless the source is
   newer, and step 4 stamped every copy with its own date, so a same-length
   edit would be missed; nor does it remove what production deleted. List
   what changed instead:

   ```bash
   aws s3api list-objects-v2 --bucket production-understory-iiif \
     --query "Contents[?LastModified>='2026-11-17'].[LastModified,Key]" --output text | sort
   ```

   Then print each changed document whose copy differs from production's, and
   `aws s3 cp` each one it names from production:

   ```bash
   aws s3api list-objects-v2 --bucket production-understory-iiif \
     --query "Contents[?LastModified>='2026-11-17'].[Key]" --output text | grep -v '^internal/' |
   while read -r key; do
     a=$(aws s3api head-object --bucket production-understory-iiif --key "$key" --query ETag --output text)
     b=$(aws s3api head-object --bucket "$NEW-iiif" --key "$key" --query ETag --output text 2>/dev/null)
     [ "$a" = "$b" ] || echo "DIFFERS: $key"
   done
   ```

   NU's list held one session from the morning of 2026-11-17, which spot
   checks found already in the backup.
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

4. Note production's `CertificateArn` and `HostedZoneId`, for step 6, and keep
   them out of this repo: they identify the account.

   ```bash
   aws cloudformation describe-stacks --stack-name production-understory \
     --query "Stacks[0].Parameters[?ParameterKey=='CertificateArn' || ParameterKey=='HostedZoneId']" --output table
   ```

## 6. The swap

1. **Release production's hostnames.** Update production in place, with its
   own template, clearing only its domain. Its distributions drop the names,
   and its DNS records and Amplify domain go. Make it a change set, so you see
   what it does to production before it does it:

   ```bash
   aws cloudformation describe-stacks --stack-name production-understory \
     --query 'Stacks[0].Parameters[].ParameterKey' --output json \
   | /usr/bin/python3 -c 'import json,sys; print(json.dumps([{"ParameterKey": k, "ParameterValue": ""} if k == "BaseDomainName" else {"ParameterKey": k, "UsePreviousValue": True} for k in json.load(sys.stdin)]))' \
     > /tmp/production-params.json

   aws cloudformation create-change-set --stack-name production-understory --change-set-name release-hostnames \
     --use-previous-template --capabilities CAPABILITY_IAM CAPABILITY_AUTO_EXPAND \
     --parameters file:///tmp/production-params.json
   aws cloudformation wait change-set-create-complete --stack-name production-understory --change-set-name release-hostnames
   aws cloudformation describe-change-set --stack-name production-understory --change-set-name release-hostnames \
     --query 'Changes[].ResourceChange.[Action,LogicalResourceId,ResourceType,Replacement]' --output table
   ```

   It should remove `AmplifyDomain`, `IIIFDnsRecord` and `ImagesDnsRecord`,
   and modify the two distributions, the functions, the Amplify app and
   branch, the API, `SourceBucket` (its CORS rule) and `IiifServer`. It must
   not replace or remove a bucket, the user pool or the search collection. If
   it does anything else, delete the change set: nothing has changed.

   **Public downtime starts here:**

   ```bash
   aws cloudformation execute-change-set --stack-name production-understory --change-set-name release-hostnames
   aws cloudformation wait stack-update-complete --stack-name production-understory
   ```

   Should it fail, production rolls back onto its names, and the downtime
   ends there.

2. **Check the names are free.** None of the three hostnames should have a
   record left in the zone. If `admin-` is still listed, Amplify may still be
   removing its record: try again after a few minutes.

   ```bash
   aws route53 list-resource-record-sets --hosted-zone-id '<HostedZoneId from step 5>' \
     --query "ResourceRecordSets[?contains(Name, '-understory.dc.')].[Name, Type]" --output text
   ```

3. **Take them.** Deploy with a temporary config that adds the domain to
   `samconfig.yaml` as it stands before step 7.1, with all three lines empty.
   `samconfig.swap.yaml` is gitignored; delete it afterwards.

   ```bash
   sed -e 's|- BaseDomainName=""|- BaseDomainName=dc.library.northwestern.edu|' \
       -e 's|- CertificateArn=""|- CertificateArn=<CertificateArn from step 5>|' \
       -e 's|- HostedZoneId=""|- HostedZoneId=<HostedZoneId from step 5>|' \
       samconfig.yaml > samconfig.swap.yaml
   sam deploy --config-file samconfig.swap.yaml
   rm samconfig.swap.yaml
   ```

   The change set adds the three DNS records and modifies the three
   distributions, the functions, `InstanceSetup`, the CORS rules and
   `IiifServer`. A `CNAMEAlreadyExists` on `AdminDistribution` would mean
   Amplify had not let go of `admin-` yet: the stack rolls back to no domain,
   so wait and deploy again. If it fails for any other reason, give
   production its names back by repeating 6.1 with `BaseDomainName` set to
   `dc.library.northwestern.edu`.

4. **Check.** The documents, an image's `info.json` and the admin UI, whose
   `config.json` names the release:

   ```bash
   curl -s https://iiif-understory.dc.library.northwestern.edu/working/presentation/collection/index/collection.json | head -c 300
   curl -s https://admin-understory.dc.library.northwestern.edu/config.json
   ```

   For an image, follow a published manifest's image service `id` and fetch
   its `info.json`. A visitor whose resolver looked a name up during the gap
   may see it missing for up to fifteen minutes more, Route 53's negative
   caching.

## 7. After the swap

1. **Commit the domain.** In `samconfig.yaml`, set
   `BaseDomainName=dc.library.northwestern.edu` and delete the `CertificateArn`
   and `HostedZoneId` lines: the stack keeps the values step 6 gave it, and
   they identify the account, which this repo does not. To check the file
   matches the stack, run `sam deploy` and answer N: it should list only
   `IiifServer`, `ImagesDistribution` and `ImagesDnsRecord`. SAM fetches the
   image server's template afresh on every deploy, so those three always show,
   and the other two follow from it.
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
Delete them deliberately, later. Its source bucket and the backup then hold
the only copies of the originals, which step 4 left out. The
`understory-github-actions-production` stack, its deploy role, can go with it.

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
