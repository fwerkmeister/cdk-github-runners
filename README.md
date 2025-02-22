# GitHub Self-Hosted Runners CDK Constructs

[![NPM](https://img.shields.io/npm/v/@cloudsnorkel/cdk-github-runners?label=npm&logo=npm)][7]
[![PyPI](https://img.shields.io/pypi/v/cloudsnorkel.cdk-github-runners?label=pypi&logo=pypi)][6]
[![Maven Central](https://img.shields.io/maven-central/v/com.cloudsnorkel/cdk.github.runners.svg?label=Maven%20Central&logo=java)][8]
[![Go](https://img.shields.io/github/v/tag/CloudSnorkel/cdk-github-runners?color=red&label=go&logo=go)][11]
[![Nuget](https://img.shields.io/nuget/v/CloudSnorkel.Cdk.Github.Runners?color=red&&logo=nuget)][12]
[![Release](https://github.com/CloudSnorkel/cdk-github-runners/actions/workflows/release.yml/badge.svg)](https://github.com/CloudSnorkel/cdk-github-runners/actions/workflows/release.yml)
[![License](https://img.shields.io/badge/license-Apache--2.0-blue)](https://github.com/CloudSnorkel/cdk-github-runners/blob/main/LICENSE)

Use this CDK construct to create ephemeral [self-hosted GitHub runners][1] on-demand inside your AWS account.

* Easy to configure GitHub integration with a web-based interface
* Customizable runners with decent defaults
* Multiple runner configurations controlled by labels
* Everything fully hosted in your account
* Automatically updated build environment with latest runner version

Self-hosted runners in AWS are useful when:

* You need easy access to internal resources in your actions
* You want to pre-install some software for your actions
* You want to provide some basic AWS API access (but [aws-actions/configure-aws-credentials][2] has more security controls)

Ephemeral (or on-demand) runners are the [recommended way by GitHub][14] for auto-scaling, and they make sure all jobs run with a clean image. Runners are started on-demand. You don't pay unless a job is running.

## API

The best way to browse API documentation is on [Constructs Hub][13]. It is available in all supported programming languages.

## Providers

A runner provider creates compute resources on-demand and uses [actions/runner][5] to start a runner.

|                  | EC2               | CodeBuild                  | Fargate        | Lambda        |
|------------------|-------------------|----------------------------|----------------|---------------|
| **Time limit**   | Unlimited         | 8 hours                    | Unlimited      | 15 minutes    |
| **vCPUs**        | Unlimited         | 2, 4, 8, or 72             | 0.25 to 4      | 1 to 6        |
| **RAM**          | Unlimited         | 3gb, 7gb, 15gb, or 145gb   | 512mb to 30gb  | 128mb to 10gb |
| **Storage**      | Unlimited         | 50gb to 824gb              | 20gb to 200gb  | Up to 10gb    |
| **Architecture** | x86_64, ARM64     | x86_64, ARM64              | x86_64, ARM64  | x86_64, ARM64 |
| **sudo**         | ✔                 | ✔                         | ✔              | ❌           |
| **Docker**       | ✔                 | ✔ (Linux only)            | ❌              | ❌           |
| **Spot pricing** | ✔                 | ❌                         | ✔              | ❌           |
| **OS**           | Linux, Windows    | Linux, Windows             | Linux, Windows | Linux         |

The best provider to use mostly depends on your current infrastructure. When in doubt, CodeBuild is always a good choice. Execution history and logs are easy to view, and it has no restrictive limits unless you need to run for more than 8 hours.

You can also create your own provider by implementing `IRunnerProvider`.

## Installation

1. Confirm you're using CDK v2
2. Install the appropriate package
   1. [Python][6]
      ```
      pip install cloudsnorkel.cdk-github-runners
      ```
   2. [TypeScript or JavaScript][7]
      ```
      npm i @cloudsnorkel/cdk-github-runners
      ```
   3. [Java][8]
      ```xml
      <dependency>
      <groupId>com.cloudsnorkel</groupId>
      <artifactId>cdk.github.runners</artifactId>
      </dependency>
      ```
   4. [Go][11]
      ```
      go get github.com/CloudSnorkel/cdk-github-runners-go/cloudsnorkelcdkgithubrunners
      ```
   5. [.NET][12]
      ```
      dotnet add package CloudSnorkel.Cdk.Github.Runners
      ```
3. Use `GitHubRunners` construct in your code (starting with default arguments is fine)
4. Deploy your stack
5. Look for the status command output similar to `aws --region us-east-1 lambda invoke --function-name status-XYZ123 status.json`
6. Execute the status command (you may need to specify `--profile` too) and open the resulting `status.json` file
7. Open the URL in `github.setup.url` from `status.json` or [manually setup GitHub](SETUP_GITHUB.md) integration as an app or with personal access token
8. Run status command again to confirm `github.auth.status` and `github.webhook.status` are OK
9. Trigger a GitHub action that has a `self-hosted` label with `runs-on: [self-hosted, linux, codebuild]` or similar
10. If the action is not successful, see [troubleshooting](#Troubleshooting)

[![Demo](demo-thumbnail.jpg)](https://youtu.be/wlyv_3V8lIw)

## Customizing

The default providers configured by `GitHubRunners` are useful for testing but probably not too much for actual production work. They run in the default VPC or no VPC and have no added IAM permissions. You would usually want to configure the providers yourself.

For example:

```typescript
let vpc: ec2.Vpc;
let runnerSg: ec2.SecurityGroup;
let dbSg: ec2.SecurityGroup;
let bucket: s3.Bucket;

// create a custom CodeBuild provider
const myProvider = new CodeBuildRunner(this, 'codebuild runner', { 
  label: 'my-codebuild',
  vpc: vpc,
  securityGroup: runnerSg,
});
// grant some permissions to the provider
bucket.grantReadWrite(myProvider);
dbSg.connections.allowFrom(runnerSg, ec2.Port.tcp(3306), 'allow runners to connect to MySQL database');

// create the runner infrastructure
new GitHubRunners(this, 'runners', {
   providers: [myProvider],
});
```

Another way to customize runners is by modifying the image used to spin them up. The image contains the [runner][5], any required dependencies, and integration code with the provider. You may choose to customize this image by adding more packages, for example.

```typescript
const myBuilder = new CodeBuildImageBuilder(this, 'image builder', {
  dockerfilePath: FargateRunner.LINUX_X64_DOCKERFILE_PATH,
  runnerVersion: RunnerVersion.specific('2.291.0'),
  rebuildInterval: Duration.days(14),    
});
myBuilder.setBuildArg('EXTRA_PACKAGES', 'nginx xz-utils');

const myProvider = new FargateRunner(this, 'fargate runner', {
  label: 'customized-fargate',
  vpc: vpc,
  securityGroup: runnerSg,
  imageBuilder: myBuilder,
});

// create the runner infrastructure
new GitHubRunners(stack, 'runners', {
  providers: [myProvider],
});
```

Your workflow will then look like:

```yaml
name: self-hosted example
on: push
jobs:
  self-hosted:
    runs-on: [self-hosted, customized-fargate]
    steps:
      - run: echo hello world
```

Windows images must be built with AWS Image Builder.

```typescript
const myWindowsBuilder = new ContainerImageBuilder(this, 'Windows image builder', {
  architecture: Architecture.X86_64,
  os: Os.WINDOWS,
  runnerVersion: RunnerVersion.specific('2.291.0'),
  rebuildInterval: Duration.days(14),    
});
myWindowsBuilder.addComponent(new ImageBuilderComponent(this, 'Ninja Component',
  {
    displayName: 'Ninja',
    description: 'Download and install Ninja build system',
    platform: 'Windows',
    commands: [
      'Invoke-WebRequest -UseBasicParsing -Uri "https://github.com/ninja-build/ninja/releases/download/v1.11.1/ninja-win.zip" -OutFile ninja.zip',
      'Expand-Archive ninja.zip -DestinationPath C:\\actions',
      'del ninja.zip',
    ],
  }
));

const myProvider = new FargateRunner(this, 'fargate runner', {
  label: 'customized-windows-fargate',
  vpc: vpc,
  securityGroup: runnerSg,
  imageBuiler: myWindowsBuilder,
});

// create the runner infrastructure
new GitHubRunners(stack, 'runners', {
  providers: [myProvider],
});
```

## Architecture

![Architecture diagram](architecture.svg)

## Troubleshooting

1. Always start with the status function, make sure no errors are reported, and confirm all status codes are OK
2. If jobs are stuck on pending:
   1. Make sure `runs-on` in the workflow matches the expected labels set in the runner provider
   2. If it happens every time, cancel the job and start it again
4. Confirm the webhook Lambda was called by visiting the URL in `troubleshooting.webhookHandlerUrl` from `status.json`
   1. If it's not called or logs errors, confirm the webhook settings on the GitHub side
   2. If you see too many errors, make sure you're only sending `workflow_job` events
5. When using GitHub app, make sure there are active installation in `github.auth.app.installations`
6. Check execution details of the orchestrator step function by visiting the URL in `troubleshooting.stepFunctionUrl` from `status.json`
   1. Use the details tab to find the specific execution of the provider (Lambda, CodeBuild, Fargate, etc.)
   2. Every step function execution should be successful, even if the runner action inside it failed

## Other Options

1. [philips-labs/terraform-aws-github-runner][3] if you're using Terraform
2. [actions-runner-controller/actions-runner-controller][4] if you're using Kubernetes


[1]: https://docs.github.com/en/actions/hosting-your-own-runners/about-self-hosted-runners
[2]: https://github.com/marketplace/actions/configure-aws-credentials-action-for-github-actions
[3]: https://github.com/philips-labs/terraform-aws-github-runner
[4]: https://github.com/actions-runner-controller/actions-runner-controller
[5]: https://github.com/actions/runner
[6]: https://pypi.org/project/cloudsnorkel.cdk-github-runners
[7]: https://www.npmjs.com/package/@cloudsnorkel/cdk-github-runners
[8]: https://search.maven.org/search?q=g:%22com.cloudsnorkel%22%20AND%20a:%22cdk.github.runners%22
[9]: https://docs.github.com/en/developers/apps/getting-started-with-apps/about-apps
[10]: https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token
[11]: https://pkg.go.dev/github.com/CloudSnorkel/cdk-github-runners-go/cloudsnorkelcdkgithubrunners
[12]: https://www.nuget.org/packages/CloudSnorkel.Cdk.Github.Runners/
[13]: https://constructs.dev/packages/@cloudsnorkel/cdk-github-runners/
[14]: https://docs.github.com/en/actions/hosting-your-own-runners/autoscaling-with-self-hosted-runners#using-ephemeral-runners-for-autoscaling
