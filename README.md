# awsmegatests

CloudFormation templates to setup the necessary cloud infrastructure
for aws megatests.

## Process

When this GitHub Action workflow is triggered, it starts an AWS batch job, which has permissions to subsequently spawn more AWS batch jobs. 
The first batch job pulls the nf-core pipeline from GitHub and starts it. 
The running pipeline can access a previously created S3 bucket to store any output files, such as the work directory, the trace directory, and the results. 
The pipeline's progress can be monitored on nf-core's nextflow tower instance. The final results are provided to nf-co.re. 

![AWS_megatests](AWS_megatests.png)

## Prerequisites

- Access to nf-core AWS account

## Steps

This process was setup by following this [guide](https://docs.opendata.aws/genomics-workflows/quick-start/) and customizing a few steps.

### Set up permissions

### (Create an S3 bucket)

There is a central bucket setup for all wor
1. Log in to AWS
2. Navigate to `S3`
3. Create new bucket, remember the name, i.e.:  `nf-core-awsmegatests`

### Set up a Virtual Private Cloud (VPC)

1. ['Launch Quick Start'](https://eu-west-1.console.aws.amazon.com/cloudformation/home?region=eu-west-1#/stacks/create/template?stackName=Quick-Start-VPC&templateURL=https://aws-quickstart.s3.amazonaws.com/quickstart-aws-vpc/templates/aws-vpc.template) 

:warning: Check region

2. Select 'Template is ready'
3. Select 'Upload a template file'
4. Use [VPCsetup.yml](https://github.com/nf-core/awsmegatests/blob/master/templates/VPCsetup.yml)
5. Set 'Availability Zones' to `eu-west-1, eu-west-2, eu-west-3`
6. Set 'Number of Availability Zones' to `3`
7. ?

### Set up Core Environment
Follow the instructions in ['Option A: Full Stack'](https://docs.opendata.aws/genomics-workflows/quick-start/):
1. Press 'Launch Stack'
2. Select 'Template is ready'
3. Select 'Upload a template file'
4. Use [GWFcore.yml](https://github.com/nf-core/awsmegatests/blob/master/templates/GWFcore.yml) and press 'Next'
5. Set 'S3 bucket name' to `nf-core-awsmegatests` (previously created)
6. Set 'Existing bucket' to `true`
7. Set 'Workflow orchestrator' to `Nextflow`
8. Private subnet -> VPC created on previous step (check on previous step in resources tab), private subnet 1A/2A/3A

### Setup Nextflow resources


### Setup GitHub Actions
In order to set up GitHub Actions, the following script has to be added to the ./github/workflows folder. 
```
name: AWS Megatests
# This workflow is triggered on pushes and PRs to the repository.
# Currently the -profile 'test' is used on AWS batch. In the future 
# this will be replaced by using actual data sets as test data.

on:
  pull_request:
    branches:
    - 'master'
  release:
    types: [published]

jobs:
  run-awstest:
    name: Run AWS test
    runs-on: ubuntu-latest
    steps:
      - name: Setup Miniconda
        uses: goanpeca/setup-miniconda@v1.0.2
        with:
          auto-update-conda: true
          python-version: 3.7
      - name: Install awscli
        run: conda install -c conda-forge awscli
      - name: Start AWS batch job
        env:
          AWS_ACCESS_KEY_ID: ${{secrets.AWS_KEY_ID}}
          AWS_SECRET_ACCESS_KEY: ${{secrets.AWS_KEY_SECRET}}
          TOWER_ACCESS_TOKEN: ${{secrets.TOWER_ACCESS_TOKEN}}
        run: |
          aws batch submit-job --region eu-west-1 --job-name <name> --job-queue '<queue name>' --job-definition nextflow --container-overrides command=nf-core/<pipeline name>,"-r ${GITHUB_SHA} -profile <AWS test profile> --outdir s3://nf-core-awsmegatests/<pipeline name>/results-${GITHUB_SHA} -w s3://nf-core-awsmegatests/<pipeline name>/work-${GITHUB_SHA}" 
```
#### Description
`Miniconda` is needed to install up `awscli`. In order to use `Miniconda` the latest stable release of a GitHub Action offered from the [marketplace](https://github.com/marketplace/actions/setup-miniconda) is used. Subsequently, `awscli` is installed via the `conda-forge` channel. For accessing the nf-core AWS account as well the nextflow tower instance, secrets have to be set. This can only be done by one of the core members within the repository under Settings > Secrets > Add new secret.

