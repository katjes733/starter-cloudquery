# starter-cloudquery

- [starter-cloudquery](#starter-cloudquery)
  - [Prerequisites](#prerequisites)
    - [CloudQuery](#cloudquery)
    - [Cloud Provider Setup](#cloud-provider-setup)
      - [AWS](#aws)
        - [SSO](#sso)
          - [Single account](#single-account)
          - [Multiple accounts](#multiple-accounts)
    - [Environment variables](#environment-variables)
  - [Migration](#migration)
    - [Local migration](#local-migration)
  - [Sync](#sync)
    - [Local sync](#local-sync)
    - [Docker sync](#docker-sync)
  - [FAQ](#faq)
    - [Why is migration separate from sync?](#why-is-migration-separate-from-sync)
    - [Why are there errors synching data?](#why-are-there-errors-synching-data)

starter project for cloudquery configuration and sync to destination

## Prerequisites

### CloudQuery

This prerequisite can be skipped if you intend to use docker for CloudQuery operations.

1. Install CloudQuery with:

   ```sh
   brew install cloudquery/tap/cloudquery
   ```

### Cloud Provider Setup

#### AWS

To configure AWS, there are multiple ways as outlined below.
Chose one or a combination of multiple based on your use case.

##### SSO

1. Make sure you have the AWS CLI installed (see [here](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)).
2. Configure a profile using `aws configure sso` (for details, see [here](https://docs.aws.amazon.com/cli/latest/userguide/sso-configure-profile-token.html)).

###### Single account

1. In `/configuration/source/aws.yml`, the following entry must exist in section `spec`:

   ```yaml
   accounts:
      - id: "111122223333"                     # id of the account
        local_profile: "profile_name_with_sso" # profile name as per ~/.aws/config
   ```

2. Before running any [Sync](#sync) operation, make sure you are signed in to SSO:

   ```sh
   aws sso login --profile profile_name_with_sso
   ```

   :warning: **NOTE**: replace `profile_name_with_sso` with the actual profile name as per `~/.aws/config`

###### Multiple accounts

This requires an organizational setup in AWS where there is an Organization established with a master account and a number of child accounts. This also requires that each account to be accessed by CloudQuery has an IAM role that can be assumed from the master account with permissions to read AWS resources.

:warning: **NOTE**: if you wish to include the organization master account, then the IAM role is available in that account as well. Otherwise

1. In `/configuration/source/aws.yml`, the following entry must exist in section `spec`:

   ```yaml
   org:
      admin_account:
        local_profile: "profile_name_with_sso" # profile name as per ~/.aws/config
      member_role_name: "member_role_name"     # name of member role in child accounts
   ```

2. Before running any [Sync](#sync) operation, make sure you are signed in to SSO:

   ```sh
   aws sso login --profile profile_name_with_sso
   ```

   :warning: **NOTE**: replace `profile_name_with_sso` with the actual profile name as per `~/.aws/config`

### Environment variables

1. Set the following environment variables:

   - PGUSER
   - PGPASSWORD
   - PGHOST
   - PGPORT
   - PGDBNAME

   You can use the following in your shell:

   ```sh
   export PGUSER=
   export PGPASSWORD=
   export PGHOST=
   export PGPORT=
   export PGDBNAME=
   ```

## Migration

### Local migration

[CloudQuery](#cloudquery) prerequisite required.

1. Run in your terminal:

   ```sh
   cloudquery migrate ./configuration/migration ./configuration/destination/postgres.yml --no-log-file --log-format json --log-console --telemetry-level none --log-level info
   ```

   This will create tables as per the migration configurations in `/configuration/migration`.

## Sync

### Local sync

[CloudQuery](#cloudquery) prerequisite required.

1. Run in your terminal:

   ```sh
   cloudquery sync ./configuration/source ./configuration/destination/postgres.yml --no-log-file --log-format json --log-console --telemetry-level none --log-level info --no-migrate
   ```

   :warning: **NOTE**: this command requires that a migration was executed prior (see [here](#migration))

### Docker sync

## FAQ

### Why is migration separate from sync?

ClouQuery supports running migration and sync within the same operation. The reason it is separated in this documentation is because when you build applications on top of the data lake, often time you wish to maintain the schema using an ORM tool. For that, you typically need the schema first, introspect and only then build out your production DB.
This also gives one better control and separation between schema and data in general.

### Why are there errors synching data?

There are mamnuy reasongs why you might see errors during sync, especially in multi account environments. Oftentimes the actual error provide sufficient insight into its root cause.
Possible error scenarios are:

- a member role in a child account is not permitted to be assumed. Check if that is on purpose (e.g. the account is more premissive) or if the member role was not deployed correctly
- a member role in a child account cannot access a resource. Check if there is an explicity deny in the corresponding member role or if the permissions do not allow the operation. There might also be SCPs impacting certain operations.
