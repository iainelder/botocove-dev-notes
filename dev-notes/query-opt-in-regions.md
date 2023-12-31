# Query opt-in regions

Alexandre Alencar points out in [issue 74](https://github.com/connelldave/botocove/issues/74) that Botocove can't access any opt-in region.

For a client project I need to query opt-in regions, so I want to solve this too.

## Set up

Use the sandbox organization management account.

```bash
.env aws-sandbox
export AWS_PROFILE=sandbox.Organization-Management-Account.480783779961.AdministratorAccess
```

Install `cove_cli`:

```bash
tmp="$(mktemp --dir)"
git clone https://github.com/iainelder/cove-cli.git "$tmp/cove-cli"
python3 -m venv "$tmp/venv"
source "$tmp/venv/bin/activate"
python3 -m pip install --upgrade pip
python3 -m pip install -e "$tmp/cove-cli"
git -C "$tmp/cove-cli" switch add-options
cove_cli --help
```

## Solve the problem in boto3

The Sandbox member accounts have no enabled opt-in regions.

```bash
cove_cli '
s.client("account").list_regions(
    MaxResults=50, RegionOptStatusContains=["ENABLED"]
)["Regions"]
' \
| jq -s -c 'sort_by(.Name) | .[] | {Id,Name,Result}'
```

```json
{"Id":"393170312900","Name":"AWS IQ","Result":[]}
{"Id":"274835020608","Name":"BayBE API","Result":[]}
{"Id":"973820050801","Name":"CodeArtifact","Result":[]}
{"Id":"726356392388","Name":"Desktop","Result":[]}
{"Id":"429218284006","Name":"Macie Adminstrator","Result":[]}
{"Id":"482035687842","Name":"Macie Workshop","Result":[]}
{"Id":"483535468253","Name":"Sandbox-B-1","Result":[]}
{"Id":"620591330564","Name":"Sandbox-B-2","Result":[]}
{"Id":"219424528675","Name":"Sandbox-B-3","Result":[]}
{"Id":"867192178045","Name":"Sandbox-B-4","Result":[]}
{"Id":"269424375644","Name":"Sandbox-B-5","Result":[]}
```

The Sandbox management account has no enabled opt-in regions.

```bash
python -c '
import boto3, json, sys
regions = boto3.Session().client("account").list_regions(
    MaxResults=50, RegionOptStatusContains=["ENABLED"]
)["Regions"]
json.dump(regions, fp=sys.stdout)
print()
'
```

```json
[]
```

List the available opt-in regions.

```bash
python -c '
import boto3, json, sys
regions = boto3.Session().client("account").list_regions(
    MaxResults=50, RegionOptStatusContains=["DISABLED"]
)["Regions"]
for r in regions:
    json.dump(r, fp=sys.stdout)
    print()
'
```

```json
{"RegionName": "af-south-1", "RegionOptStatus": "DISABLED"}
{"RegionName": "ap-east-1", "RegionOptStatus": "DISABLED"}
{"RegionName": "ap-south-2", "RegionOptStatus": "DISABLED"}
{"RegionName": "ap-southeast-3", "RegionOptStatus": "DISABLED"}
{"RegionName": "ap-southeast-4", "RegionOptStatus": "DISABLED"}
{"RegionName": "eu-central-2", "RegionOptStatus": "DISABLED"}
{"RegionName": "eu-south-1", "RegionOptStatus": "DISABLED"}
{"RegionName": "eu-south-2", "RegionOptStatus": "DISABLED"}
{"RegionName": "il-central-1", "RegionOptStatus": "DISABLED"}
{"RegionName": "me-central-1", "RegionOptStatus": "DISABLED"}
{"RegionName": "me-south-1", "RegionOptStatus": "DISABLED"}
```

Set a profile for Sandbox-B-1.

```bash
aws configure set "source_profile" "sandbox.Organization-Management-Account.480783779961.AdministratorAccess" --profile "483535468253.OAAR"
aws configure set "role_arn" "arn:aws:iam::483535468253:role/OrganizationAccountAccessRole" --profile "483535468253.OAAR"
aws sts get-caller-identity --profile 483535468253.OAAR
```

```json
{
    "UserId": "AROAXBFHVW3O7ENM3BJWQ:botocore-session-1695513573",
    "Account": "483535468253",
    "Arn": "arn:aws:sts::483535468253:assumed-role/OrganizationAccountAccessRole/botocore-session-1695513573"
}
```

Enable an opt-in region in Sandbox-B-1. The command is mute.

```bash
aws account enable-region --profile 483535468253.OAAR --region-name eu-central-2
```

Run this loop until the RegionOptStatus is ENABLED.

```bash
while true; do
    aws account get-region-opt-status --profile 483535468253.OAAR --region-name eu-central-2 \
    | jq --arg ts "$(date --utc --iso-8601=sec)" -c '{ts: $ts} + .'
    sleep 10
done
```

```json
{"ts":"2023-09-24T00:18:38+00:00","RegionName":"eu-central-2","RegionOptStatus":"ENABLING"}
{"ts":"2023-09-24T00:18:50+00:00","RegionName":"eu-central-2","RegionOptStatus":"ENABLING"}
{"ts":"2023-09-24T00:19:02+00:00","RegionName":"eu-central-2","RegionOptStatus":"ENABLED"}
```

List the availability zones in Sandbox-B-1 in eu-central-1 and eu-central-2.

```bash
cove_cli \
--target-ids 483535468253 \
--regions eu-central-1 eu-central-2 \
-- 's.client("ec2").describe_availability_zones()["AvailabilityZones"][0]["RegionName"]' \
| jq -s -c 'sort_by(.Name) | .[] | {Id,Name,Result,ExceptionDetails}'
```

Both regions fail. I expected eu-central-2 to fail, but not eu-central-1. eu-central-1 failed because of an SCP that blocks most regions.

```json
{"Id":"483535468253","Name":"Sandbox-B-1","Result":null,"ExceptionDetails":"ClientError('An error occurred (UnauthorizedOperation) when calling the DescribeAvailabilityZones operation: You are not authorized to perform this operation.')"}
{"Id":"483535468253","Name":"Sandbox-B-1","Result":null,"ExceptionDetails":"ClientError('An error occurred (AuthFailure) when calling the DescribeAvailabilityZones operation: AWS was not able to validate the provided access credentials')"}
```

Disable the SCP.

```bash
aws organizations detach-policy --policy-id p-6gyzbwaw --target-id r-auh0
```

Now eu-central-1 works and eu-central-2 fails.

```json
{"Id":"483535468253","Name":"Sandbox-B-1","Result":null,"ExceptionDetails":"ClientError('An error occurred (AuthFailure) when calling the DescribeAvailabilityZones operation: AWS was not able to validate the provided access credentials')"}
{"Id":"483535468253","Name":"Sandbox-B-1","Result":"eu-central-1","ExceptionDetails":null}
```

Try to reproduce the error using the basic CLI.

This matches because it gives `eu-central-1`.

```bash
aws ec2 describe-availability-zones --profile 483535468253.OAAR --region eu-central-1 \
| jq -r '.AvailabilityZones[0].RegionName'
```

This doesn't match because it gives `eu-central-2`. Botocove gives an error.

```bash
aws ec2 describe-availability-zones --profile 483535468253.OAAR --region eu-central-2 \
| jq -r '.AvailabilityZones[0].RegionName'
```

Try to reproduce the error using Boto3.

This doesn't match because it gives a InvalidClientTokenId error. Botocove gives an AuthFailure error.

```python
import boto3
(
    boto3.Session(
        profile_name="483535468253.OAAR", region_name="eu-central-2"
    ).client("ec2")
    .describe_availability_zones()
    ["AvailabilityZones"][0]["RegionName"]
)
```

```text
ClientError: An error occurred (InvalidClientTokenId) when calling the AssumeRole operation: The security token included in the request is invalid
```

This matches the Botocove behavior. It gives the same AuthFailure error.

```python
import boto3

creds = boto3.Session().client("sts").assume_role(
    RoleSessionName="Test",
    RoleArn="arn:aws:iam::483535468253:role/OrganizationAccountAccessRole"
)["Credentials"]

session = boto3.Session(
    aws_access_key_id=creds["AccessKeyId"],
    aws_secret_access_key=creds["SecretAccessKey"],
    aws_session_token=creds["SessionToken"],
    region_name="eu-central-2"
)

session.client("ec2").describe_availability_zones()["AvailabilityZones"][0]["RegionName"]
```

```text
ClientError: An error occurred (AuthFailure) when calling the DescribeAvailabilityZones operation: AWS was not able to validate the provided access credentials
```

Read a Knowledge Center article to get the answer: [Why did I receive the IAM error "AWS was not able to validate the provided access credentials" in some AWS Regions?](https://repost.aws/knowledge-center/iam-validate-access-credentials). It shows how to set the endpoint URL of the STS client to work with opt-in regions.

This works as the CLI does. It gives `eu-central-2`.

This is how Botocove needs to work too.

```python
import boto3

mgmt = boto3.Session()

creds = mgmt.client(
    "sts",
    region_name=mgmt.region_name,
    endpoint_url=f"https://sts.{mgmt.region_name}.amazonaws.com",
).assume_role(
    RoleSessionName="Test",
    RoleArn="arn:aws:iam::483535468253:role/OrganizationAccountAccessRole"
)["Credentials"]

member = boto3.Session(
    aws_access_key_id=creds["AccessKeyId"],
    aws_secret_access_key=creds["SecretAccessKey"],
    aws_session_token=creds["SessionToken"],
    region_name="eu-central-2"
)

member.client("ec2").describe_availability_zones()["AvailabilityZones"][0]["RegionName"]
```

## Solve the problem in botocove

Before I create a GitHub issue in the Botocove repo I need to create repro using the Botocove library.

```python
from botocove import cove
from itertools import chain

response = cove(
    lambda s: (
        s.client("ec2").describe_availability_zones()
        ["AvailabilityZones"][0]["RegionName"]
    ),
    target_ids=["483535468253"],
    regions=["eu-central-1", "eu-central-2"]
)()

for result in chain(
            response["FailedAssumeRole"],
            response["Exceptions"],
            response["Results"],
        ):
    print(
        repr(
            {
                k: v
                for k, v in result.items()
                if k in {"Id", "Region", "Result", "ExceptionDetails"}
            }
        )
    )
```

Region `eu-central-1` echoes its name and region `eu-central-2` gives an `AuthFailure` error.

```python
{'Id': '483535468253', 'Region': 'eu-central-2', 'ExceptionDetails': ClientError('An error occurred (AuthFailure) when calling the DescribeAvailabilityZones operation: AWS was not able to validate the provided access credentials')}
{'Id': '483535468253', 'Region': 'eu-central-1', 'Result': 'eu-central-1'}
```

Use botocove as a convenient way to check the opt-in status of the target regions. You can use the `Accounts.ListRegions` API but you first need to enable trusted access in the organization.

```python
response = cove(
    lambda s: s.client("account").list_regions(), target_ids=["483535468253"]
)()

[
    r
    for r in response["Results"][0]["Result"]["Regions"]
    if r["RegionName"] in {"eu-central-1", "eu-central-2"}
]
```

```python
[{'RegionName': 'eu-central-1', 'RegionOptStatus': 'ENABLED_BY_DEFAULT'},
 {'RegionName': 'eu-central-2', 'RegionOptStatus': 'ENABLED'}]
```

## Create an issue

Create [issue 77](https://github.com/connelldave/botocove/issues/77) in the botocove repo.

## Fix it in botocove

Now use the botocove repo for the tests so that I can fix the code directly.

Stay in the Sandbox organization management account. Leave the virtualenv and go to the botocove working copy.

I refactor the session and client initializers to make them easier to read. Then I set the `endpoint_url` in the STS client.

The repro program now gives the expected result.

```python
{'Id': '483535468253', 'Region': 'eu-central-2', 'Result': 'eu-central-2'}
{'Id': '483535468253', 'Region': 'eu-central-1', 'Result': 'eu-central-1'}
```

I commit 02a0567a58ea8a00cdc39e829821c73a61863626 with the fix.

Pytest passes locally, but fails in the latest CI run ([#103](https://github.com/iainelder/botocove/actions/runs/6448602367/job/17506051971)).

The first failure is in `test_when_two_regions_are_passed_then_output_has_one_result_per_account_per_region` and is because of a `NoRegionError`.

I'm not sure why pytest passed locally the first time around in pre-commit. When I run it again it fails in the same way. Use `pytest --ff -x` to rerun failures first and exit on the first failure.

When I remove the `endpoint_url` parameter, all the tests pass.

## Test regional endpoint support in moto

Does moto support STS regional endpoints?

Use the aws-sandbox environment but don't set a default profile.

I think moto fails when the `AWS_DEFAULT_REGION` environment variable is unset.

Set up functions to get the identity using default and regional endpoints.

```python
from boto3.session import Session
from moto import mock_sts
from unittest.mock import patch
import os

def default_identity():
    sts_client = Session().client("sts")
    return sts_client.get_caller_identity()

def regional_identity():
    sts_client = Session().client("sts", endpoint_url="https://sts.eu-west-1.amazonaws.com")
    return sts_client.get_caller_identity()
```

These work except number 4. The mocked calls in 5 and 6 work because a region is set either from the profile or from the environment variable.

```python
with patch.dict(os.environ, {"AWS_PROFILE": "sandbox.Organization-Management-Account.480783779961.AdministratorAccess"}):
    print(default_identity())

with patch.dict(os.environ, {"AWS_PROFILE": "sandbox.Organization-Management-Account.480783779961.AdministratorAccess"}):
    print(regional_identity())

with patch.dict(os.environ, {}, clear=True), mock_sts():
    print(default_identity())

with patch.dict(os.environ, {}, clear=True), mock_sts():
    print(regional_identity())

with patch.dict(os.environ, {"AWS_DEFAULT_REGION": "eu-west-1"}, clear=True), mock_sts():
    print(regional_identity())

with patch.dict(os.environ, {"AWS_PROFILE": "sandbox.Organization-Management-Account.480783779961.AdministratorAccess"}), mock_sts():
    print(regional_identity())
```

Number 4 fails with this error:

```text
NoRegionError: You must specify a region.
```

I can reproduce the error in number 2 if I delete the `region` key from the management profile and unset the `AWS_DEFAULT_REGION` environment variable.

That suggests it's not because of moto at all but because of how boto3 3 behaves.

If I set the `endpoint_url` in the STS client do I also have to set the `region_name`? In fact that's what the repost example shows.

## Dig into the NoRegion error

```python
from boto3.session import Session
from unittest.mock import patch
import os
from moto import mock_sts
from contextlib import contextmanager
from typing import Dict


@contextmanager
def tmpenv(env: Dict[str, str]) -> None:
    yield patch.dict(os.environ, {}, clear=True)


def identity(*args, **kwargs):
    sts = Session().client("sts", *args, **kwargs)
    print(sts.get_caller_identity()["Account"])


# NoCredentialsError: Unable to locate credentials.
with tmpenv({}):
    identity()


# NoRegionError: You must specify a region.
with tmpenv({}):
    identity(endpoint_url="https://sts.us-east-1.amazonaws.com")


# NoCredentialsError: Unable to locate credentials
with tmpenv({}):
   identity(
        endpoint_url="https://sts.us-east-1.amazonaws.com",
        region_name="us-east-1",
    )

# 123456789012
with tmpenv({}), mock_sts():
    identity(
        endpoint_url="https://sts.us-east-1.amazonaws.com",
        region_name="us-east-1",
    )

# NoRegionError: You must specify a region.
with tmpenv({}), mock_sts():
    identity(endpoint_url="https://sts.us-east-1.amazonaws.com")


# NoRegionError: You must specify a region.
with tmpenv({}), mock_sts():
    identity(
        endpoint_url=f"https://sts.{None}.amazonaws.com",
        region_name=None,
    )
```

I think it's the case with `None` that triggers the strange behavior in the unit tests.

## Review branch to fix assuming session bug

I seem to have known about an "assuming session bug" since 2022-04 (see VWAG 2022-03 work diary).

Review the almost-forgotten branch `fix-assuming-session-region`. It appears to address this bug, but it doesn't include a repro so I'm not sure what it is supposed to fix. Maybe the unit tests cover it.

I created the branch around 2023-06. My `Homegate-Organization.md` note from that time says to "Set the assuming session for botocove using environment variables to avoid a bug in single-region mode."

It suggests to set the assuming session like this in Python code:

```python
os.environ["AWS_PROFILE"] = "Profile1"
os.environ["AWS_DEFAULT_REGION"] = "eu-central-1"

@cove()
def check(session: Session):
    return session.client("sts").get_caller_identity()
```

Rather than something like this:

```python
    @cove(
        assuming_session=Session(
            profile_name="Profile1",
            region_name="eu-central-1",
        ),
        rolename=role_name
    )
    def check(session):
        return session.client("sts").get_caller_identity()
```

I set `assuming_session` a lot in my analyses before 2023. I might find some error examples if I dig through old notes.

On insight from that branch is that the `do_nothing` function in the unit tests is ineffective because it, well, does nothing to exercise the session or client.

Replace it with a function that calls a regional endpoint. In the branch I used an SQS API because it's a simple regional API.

Revive the `test_assuming_session.py` test module from the branch.

The tests expose two bugs. I had to mark the tests as `xfail` to commit them.

Before I add the regional STS feature, I need to fix these bugs.

## Fix assuming region bug

2023-10-21.

The "bugs" exposed by my tests around the SQS API were in fact caused by my failure to mock the SQS service! I fixed it by adding `@mock_sqs` to `_call_regional_api`.

(Later I swap out SQS for EC2, but the point is the same: I need to mock all the services.)

Fix the assuming session bug! Now when the assuming session has a region and the default region is unset, cove uses the assuming session's region.

## Add support for opt-in regions

Now I am ready to add support for opt-in regions.

How can I test the opt-in region behavior using Moto?

Check the regions in the service model and count the availability zones.

```python
from moto import mock_sts, mock_ec2
from boto3.session import Session
from botocore.exceptions import ClientError
from pprint import pprint


def count_AZs(session, region):
    try:
        AZs = session.client("ec2", region_name=region).describe_availability_zones()[
            "AvailabilityZones"
        ]
        return len(AZs)
    except ClientError as error:
        if error.response["Error"]["Code"] == "AuthFailure":
            return None
        raise


def count_AZs_for_all_regions(session):
    regions = session.get_available_regions(service_name="ec2")
    return {r: count_AZs(session, r) for r in regions}


def display(name, azs_per_region):
    pprint(name)
    pprint(len(azs_per_region))
    pprint(azs_per_region)


with mock_sts(), mock_ec2():
    display("Moto", count_AZs_for_all_regions(Session()))

display("Sandbox", count_AZs_for_all_regions(Session()))
```

My Boto3 version is 1.28.68. Its service model has 28 regions. The [EC2 Regions and Zones](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-regions-availability-zones.html) page lists also 28 regions. (It sorts the regions in americabetical order.)

The [AWS global infrastructure](https://aws.amazon.com/about-aws/global-infrastructure/) advertises 32 launched regions. The other 4 are 2 China regions and 2 US GovCloud regions.

Moto doesn't model opt-in regions. All regions act enabled and give a count of availability zones. Some regions have 0 availability zones.

```python
'Moto'
28
{'af-south-1': 3,
 'ap-east-1': 3,
 'ap-northeast-1': 3,
 'ap-northeast-2': 4,
 'ap-northeast-3': 3,
 'ap-south-1': 3,
 'ap-south-2': 0,
 'ap-southeast-1': 3,
 'ap-southeast-2': 3,
 'ap-southeast-3': 3,
 'ap-southeast-4': 0,
 'ca-central-1': 3,
 'eu-central-1': 3,
 'eu-central-2': 0,
 'eu-north-1': 3,
 'eu-south-1': 3,
 'eu-south-2': 0,
 'eu-west-1': 3,
 'eu-west-2': 3,
 'eu-west-3': 3,
 'il-central-1': 0,
 'me-central-1': 0,
 'me-south-1': 3,
 'sa-east-1': 3,
 'us-east-1': 6,
 'us-east-2': 3,
 'us-west-1': 2,
 'us-west-2': 4}
```

The Sandbox management account has no enabled opt-in regions. `None` indicates a disabled opt-in region.

```python
'Sandbox'
28
{'af-south-1': None,
 'ap-east-1': None,
 'ap-northeast-1': 3,
 'ap-northeast-2': 4,
 'ap-northeast-3': 3,
 'ap-south-1': 3,
 'ap-south-2': None,
 'ap-southeast-1': 3,
 'ap-southeast-2': 3,
 'ap-southeast-3': None,
 'ap-southeast-4': None,
 'ca-central-1': 3,
 'eu-central-1': 3,
 'eu-central-2': None,
 'eu-north-1': 3,
 'eu-south-1': None,
 'eu-south-2': None,
 'eu-west-1': 3,
 'eu-west-2': 3,
 'eu-west-3': 3,
 'il-central-1': None,
 'me-central-1': None,
 'me-south-1': None,
 'sa-east-1': 3,
 'us-east-1': 6,
 'us-east-2': 3,
 'us-west-1': 2,
 'us-west-2': 4}
```

In the sandbox this `Account.ListRegions` call returns `[]`.

Moto fails with this error: `ClientError: An error occurred (UnrecognizedClientException) when calling the ListRegions operation: The security token included in the request is invalid.`

(Moto's [IMPLEMENTATION_COVERAGE.md](https://github.com/getmoto/moto/blob/531fdc7851ad1fa259c6298596343d6589f206ad/IMPLEMENTATION_COVERAGE.md) doesn't even list the `account` service yet, so the support seems a long way off.)

```python
with mock_sts(), mock_ec2():
    enabled_regions = Session().client("account").list_regions(
        RegionOptStatusContains=["ENABLED"], MaxResults=50
    )["Regions"]
    pprint(enabled_regions)

enabled_regions = Session().client("account").list_regions(
    RegionOptStatusContains=["ENABLED"], MaxResults=50
)["Regions"]
pprint(enabled_regions)
```

For now I accept that there is no unit test coverage for the opt-in regions. It's good enough that the opt-in region support passes the existing unit tests and passes the test in a real organization described in the GitHub issue (see above section "[Solve the problem in Botocove](#solve-the-problem-in-botocove)").

Use the solution from the linked repost article.

Before fix:

```python
{'Id': '483535468253', 'Region': 'eu-central-2', 'ExceptionDetails': ClientError('An error occurred (AuthFailure) when calling the DescribeAvailabilityZones operation: AWS was not able to validate the provided access credentials')}
{'Id': '483535468253', 'Region': 'eu-central-1', 'Result': 'eu-central-1'}
```

After fix:

```python
{'Id': '483535468253', 'Region': 'eu-central-1', 'Result': 'eu-central-1'}
{'Id': '483535468253', 'Region': 'eu-central-2', 'Result': 'eu-central-2'}
```

I need to put a branch on the assuming session region name to make the existing tests pass. I don't like this. I don't think the branch is necessary for real use cases. In fact it breaks the real single-region use case because now cove outputs the `Region` key. None of the existing tests catch that. Run the GitHub issue code and comment out the `regions` parameter to see.

It outputs this:

```python
{'Id': '483535468253', 'Region': 'eu-central-1', 'Result': 'eu-central-1'}
```

But it should output this:

```python
{'Id': '483535468253', 'Result': 'eu-central-1'}
```

What are the older tests doing to mess it up like this?

## Check how moto works without AWS context

On my local computer, there are no `AWS` environment variables. The following returns `{}`.

```python
import os
{k: v for k, v in os.environ.items() if "AWS" in k}
```

The CoveHostAccount invokes STS and Organizations before running the cove function. Those services both are "global" services in the sense that they work without setting a region in the context. I use the EC2 service in new tests as a simple regional API.

```python
from boto3.session import Session
from moto import mock_sts, mock_organizations, mock_ec2
```

Moto's version of `STS.GetCallerIdentity` succeeds.

```python
mm = mock_sts().__enter__()
Session().client("sts").get_caller_identity()
```

```python
{'UserId': 'AKIAIOSFODNN7EXAMPLE',
 'Account': '123456789012',
 'Arn': 'arn:aws:sts::123456789012:user/moto',
 'ResponseMetadata': {'RequestId': 'c6104cbe-af31-11e0-8154-cbc7ccf896c7',
  'HTTPStatusCode': 200,
  'HTTPHeaders': {'server': 'amazon.com',
   'date': 'Sun, 22 Oct 2023 11:11:35 GMT'},
  'RetryAttempts': 0}}
```

The unpatched API raises a `NoCredentialsError`.

```python
mm.__exit__()
Session().client("sts").get_caller_identity()
```

```text
NoCredentialsError: Unable to locate credentials
```

Moto's version of `Organizations.DescribeOrganization`, although it raises `AWSOrganizationsNotInUseException`, succeeds because that service would respond with that error when no organization exists.

```python
mm = mock_organizations().__enter__()
Session().client("organizations").describe_organization()
```

```text
AWSOrganizationsNotInUseException: An error occurred (AWSOrganizationsNotInUseException) when calling the DescribeOrganization operation: Your account is not a member of an organization.
```

The unpatched SDK raises `NoCredentialsError`.

```python
mm.__exit__()
Session().client("organizations").describe_organization()
```

```text
NoCredentialsError: Unable to locate credentials
```

Moto's version of `EC2.DescribeAvailabilityZones` raises a `NoRegionError`.

```python
mm = mock_ec2().__enter__()
Session().client("ec2").describe_availability_zones()
```

```text
NoRegionError: You must specify a region.
```

The unpatched SDK matches Moto's behavior.

```python
mm.__exit__()
Session().client("ec2").describe_availability_zones()
```

```text
NoRegionError: You must specify a region.
```

So Moto doesn't care about credentials but does care about regions.

## Find functions that call `do_nothing`

I suspect the tests that call `do_nothing` are why I haven't noticed this till now.

`ack` doesn't match line ranges, so use `awk`.

Had to learn some new tricks to do this:

* [Print range if start and end match](https://unix.stackexchange.com/questions/456843/print-data-between-two-lines-only-if-range-end-exists-from-a-text-file)
* [Assign a here-document to a variable](https://stackoverflow.com/questions/1167746/how-to-assign-a-heredoc-value-to-a-variable-in-bash)
* [Assign `printf` output to variable](https://stackoverflow.com/questions/48953008/assigning-printf-output-to-a-variable-in-awk-command)

```bash
read -r -d $'\0' prog <<"EOF"
function present_as_ack() { return sprintf("%s:%s:%s", FILENAME, NR, $0) }
/^def test_/       { hold = present_as_ack(); next }
                   { hold = hold ORS present_as_ack() }
/def do_nothing\(/ { print hold }
EOF

find tests -type f -name '*.py' -exec awk "$prog" '{}' +
```

The output looks like `ack` for a range match.

```text
tests/test_regions.py:724:def test_when_region_is_unspecified_then_result_has_no_region_key(
tests/test_regions.py:725:    mock_small_org: SmallOrg,
tests/test_regions.py:726:) -> None:
tests/test_regions.py:727:    @cove()
tests/test_regions.py:728:    def do_nothing(session: Session) -> None:
tests/test_regions.py:737:def test_when_region_is_unspecified_then_output_has_one_result_per_account(
tests/test_regions.py:738:    mock_session: Session, mock_small_org: SmallOrg
tests/test_regions.py:739:) -> None:
tests/test_regions.py:740:    @cove()
tests/test_regions.py:741:    def do_nothing(session: Session) -> None:
```

Change the `awk` program to output just the first line of the matched range. That's all I need to find the test functions. Just remove the block that adds to the hold space.

```bash
read -r -d $'\0' prog <<"EOF"
function present_as_ack() { return sprintf("%s:%s:%s", FILENAME, NR, $0) }
/^def test_/       { hold = present_as_ack(); next }
/def do_nothing\(/ { print hold }
EOF

find tests -type f -name '*.py' -exec awk "$prog" '{}' +
```

Now I can see all the test functions at a glance.

```text
tests/test_regions.py:724:def test_when_region_is_unspecified_then_result_has_no_region_key(
tests/test_regions.py:737:def test_when_region_is_unspecified_then_output_has_one_result_per_account(
tests/test_regions.py:752:def test_when_region_is_str_then_raises_type_error(mock_small_org: SmallOrg) -> None:
tests/test_regions.py:763:def test_when_region_is_empty_then_raises_value_error(mock_small_org: SmallOrg) -> None:
tests/test_regions.py:774:def test_when_any_region_is_passed_then_result_has_region_key(
tests/test_regions.py:787:def test_when_two_regions_are_passed_then_output_has_one_result_per_account_per_region(
tests/test_target_ids.py:817:def test_when_unspecified_then_output_has_a_result_for_each_org_account(
tests/test_target_ids.py:828:def test_when_accounts_in_org_then_output_has_result_for_each_target_id(
tests/test_target_ids.py:844:def test_when_target_id_is_str_then_raises_type_error(mock_small_org: SmallOrg) -> None:
tests/test_target_ids.py:855:def test_when_account_not_in_org_raises_value_error(
tests/test_target_ids.py:871:def test_when_empty_sequence_raises_value_error(
```

## Fix all the tests

I updated all those tests above.

The version 2 tokens had knock-on effects on all the test code. I had to add at least a default region fixture to most of  the modules.

I took the opportunity to delete broken testing of internals. Tests should cover the public interface and behavior.

It's quite a big change now, so sleep on it and review and refactor later.

```console
$ git diff --shortstat query-opt-in-regions master
 14 files changed, 330 insertions(+), 181 deletions(-)
```

Consider splitting the PR into two. It may actually be the prerequisite bug fix that requires those test changes rather than the version 2 token support itself.

## Submit PR

Create [PR 79](https://github.com/connelldave/botocove/pull/79).

Offer to split it into two PRs if it makes the changes clearer.

Await feedback from Dave Connell.

## Catch up

2023-12-27.

I closed PR 79 because it was too big to review and I didn't even understand all the changes. Dave Connell suggested that I break it up into multiple PRs to make it easier to follow.

Dave Connell merged [PR 80](https://github.com/connelldave/botocove/pull/80), which simplifies the host account initialization.

Now I work on fixing the elusive assuming session bug.

Push new branch fix-assuming-session-bugs to start to address this. It's been so long I've forgotten what I wrote above.

## Revisit NoRegionError

2023-12-29.

Reread this sections.

> So Moto doesn't care about credentials but does care about regions.

The tests that `do_nothing` hide the buggy region handling.

Don't change that yet. It needs a deeper refactor.

## Submit PR for assuming region

2023-12-29.

https://github.com/connelldave/botocove/pull/84

## Next steps:

Moto can't test opt-in regions. What about LocalStack?

If LocalStack can't test this, what about a dedicated test organization?

Refactor or delete all the tests that check complete outputs instead of just a feature. These break every time a feature changes. It's unclear what they cover.

* `test_decorator_no_args`
* tests/test_decorator.py::test_decorated_simple_func_passed_args
* tests/test_session.py::test_session_result_formatter
* tests/test_session.py::test_session_result_formatter_with_policy_arn
* tests/test_session.py::test_session_result_formatter_with_policy
* tests/test_exceptions.py::test_handled_exception_in_wrapped_func
