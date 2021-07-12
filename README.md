<!-- note marker start -->
**NOTE**: This repo contains only the documentation for the private BoltsOps Pro repo code.
Original file: https://github.com/boltopspro/session-manager-cli/blob/master/README.md
The docs are publish so they are available for interested customers.
For access to the source code, you must be a paying BoltOps Pro subscriber.
If are interested, you can contact us at contact@boltops.com or https://www.boltops.com

<!-- note marker end -->

# Session Manager

[![Support](https://img.shields.io/badge/get-support-blue.svg)](https://boltops.com?utm_source=badge&utm_medium=badge&utm_campaign=session-manager)

[![BoltOps Badge](https://img.boltops.com/boltops/badges/boltops-badge.png)](https://www.boltops.com)

Sets up [AWS session manager](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager.html). This is useful if you are setting it up repeatedly on multiple AWS accounts or multiple regions.

The tool does the following:

    1. Creates a sessions CloudWatch log group
    2. Sets up the Session Manager preference for the region to use that CloudWatch log group.

Though this script can run multiple times, it is meant to be used as a one-off script. Changes to session manager preferences, KMS key policy, CloudWatch log outside of this script will not be reflected.

Nevertheless, this script is useful to get rid of some mundane boilerplate tasks.

## Why Use Session Manager?

[AWS session manager](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager.html) provides SSH access to EC2 instances in a super-secure way.  Some benefits:

* **secure access**: Access is established across a secure tunnel. No need to open any inbound ports at all.
* **centralized access control**: You can use IAM users and policies to control access.
* **compliance and auditability**: Commands are logged to Amazon CloudWatch and or S3.
* **low management maintenance**: No need to managed a bastion host. No need to set up individual ssh keys on every instance.

The benefits are pretty meaningful. With this setup, **zero** ports are required to be open.

If you required your IAM users to log into the AWS Console with MFA, then you essentially get MFA ssh for free.

The ssh session can be either in a browser or a terminal shell.  If you are on the go and need to debug an issue, all you need is a browser to start a shell session.

## How Session Manager Works?

Here's a short summary of how Session Manager works.

* You run the ssm-agent on your instances.
* You use either the AWS console or the AWS CLI with the Session Manager Plugin to start a session.
* An encrypted tunnel originating from the host is started by the ssm-agent.
* The communication between your client and the tunnel is done via a WebSocket.

## Install

    git clone git@github.com:boltopspro/session-manager
    cd session-manager
    bundle

## Usage

    exe/session-manager setup

Example output on first run:

    $ exe/session-manager setup
    Creating CloudWatch log group: sessions
    Creating SSM Document SSM-SessionManagerRunShell with Session Manager preferences
    $

Example output on additional runs:

    $ exe/session-manager setup
    Creating CloudWatch log group: sessions
    CloudWatch log group sessions already exists
    Creating SSM Document SSM-SessionManagerRunShell with Session Manager preferences
    WARN: SSM Document SSM-SessionManagerRunShell already exists
    $

## Noop Mode

The `--noop` option is useful to see what the `session-manager setup` script will do.  Example:

    $ exe/session-manager setup --noop
    NOOP: Creating CloudWatch log group: sessions
    NOOP: Creating SSM Document SSM-SessionManagerRunShell with Session Manager preferences
    $

## Start Session

Here's an example of you start a shell session with AWS session manager.

### CLI

For the CLI, you'll need install the Session Manager Plugin. Docs: [Install the Session Manager Plugin for the AWS CLI](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-install-plugin.html). Then you can:

    aws ssm start-session --target i-055de3ac214e88fc6 # replace with your own instance id

Example with output:

    $ aws ssm start-session --target i-055de3ac214e88fc6
    Starting session with SessionId: botocore-session-1565553907-04abdc1e2eb9c3652
    sh-4.2$ pwd
    /usr/bin
    sh-4.2$ exit
    Exiting session with sessionId: botocore-session-1565553907-04abdc1e2eb9c3652.
    $

### Console

1. Go to the **Systems Manager** console
2. Click on **Session Manager** on the left-hand side menu
3. Make sure you're on the first tab: **Sessions**
4. Click **Start session** on the right-hand side
5. Select the target instance
6. Click **Start session**

## SSH Support

Session Manager also supports ssh access. See: [Enable SSH Connections Through Session Manager](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-getting-started-enable-ssh-connections.html).  Essentially you configure your `~/.ssh/config` with something like this:

    # SSH over Session Manager
    host i-* mi-*
        ProxyCommand sh -c "aws ssm start-session --target %h --document-name AWS-StartSSHSession --parameters 'portNumber=%p'"

Then you can use ssh with the instance id to access the instance. Example:

    ssh i-065d6916EXAMPLE

## Keeping SSM Agent Updated

It is recommended to keep the ssm agent on the instances up-to-date. This can be pretty easily accomplished with State Manager Associations: [Automate Updates to SSM Agent
](https://docs.aws.amazon.com/systems-manager/latest/userguide/ssm-agent-automatic-updates.html).

Here's an example of the AWS CLI command that will work with instances that have StateManager tag is set to yes.

    aws ssm create-association --name SsmAgent --targets Key=tag:StateManager,Values=yes --name AWS-UpdateSSMAgent --schedule-expression "cron(0 2 ? * * *)"

And if you decide to update the association later. Note, we'll use `ac977c83-180e-4b5c-a639-c22d49ef29e3` as an example association id. You'll need to replace it with your own.

    aws ssm update-association  --name SsmAgent --targets Key=tag:StateManager,Values=yes --name AWS-UpdateSSMAgent --schedule-expression "cron(0 1 ? * * *)" --association-id ac977c83-180e-4b5c-a639-c22d49ef29e3

## KMS Encrypted CloudWatch Log

The script can optionally create a KMS key used to encrypt the CloudWatch Log group also.

    1. Creates a kms key if it doesn't exist. The default alias name is `session`.
    2. Creates a sessions CloudWatch log group encrypted with that key
    3. Sets up the Session Manager preference for the region to use that CloudWatch log group.

**Cavaets:**

Be aware that doing encrypted the CloudWatch Log group will mean instances using Session Manager require access to the KMS key. If the instance does not have KMS access, Session Manager will simply close upon trying to launch.

AWS changed the recommended IAM Managed Policy from `AmazonEC2RoleForSSM` to `AmazonSSMManagedInstanceCore` in [Aug 2019](https://aws.amazon.com/blogs/aws/new-session-manager/):

> Update (August 2019) – The original version of this blog post referenced the now-deprecated AmazonEC2RoleForSSM IAM policy. It has been updated to reference the AmazonSSMManagedInstanceCore policy instead.

Switching over to the latest recommended `AmazonSSMManagedInstanceCore` policy could result in Session Manager not working if the Session Preferences are configured with an encrypted CloudWatch Log group. This is because the Session Manager and the CloudWatch Log Group require access to the KMS key and the `AmazonSSMManagedInstanceCore` policy does not grant KMS access.

## Cleanup: Delete Default Session Preferences

Session Manager Preferences can be updated in the System Managers Console under Session Manager > Preferences:

![](https://img.boltops.com/boltopspro/blueprints/session-manager/session-preferences.png)

Underneath the hood, Session Manager stores these Session Preferences an [SSM Document](https://docs.aws.amazon.com/systems-manager/latest/userguide/sysman-ssm-docs.html) called `SSM-SessionManagerRunShell`:

![](https://img.boltops.com/boltopspro/blueprints/session-manager/SSM-SessionManagerRunShell.png)

Session Manager creates this SSM Document if you have edited the Preferences.  If this document already exist, this script will not override it. You have to delete it first. Here's the command:

    aws ssm delete-document --name SSM-SessionManagerRunShell

Here's also the command to delete the CloudWatch Log group if you want to delete that also.

    aws logs delete-log-group --log-group-name sessions

Also you can quickly delete both CloudWatch Log Group and SSM Document with the `session-manager delete` command.

## References

* AWS Blog Post: [New – AWS Systems Manager Session Manager for Shell Access to EC2 Instances](https://aws.amazon.com/blogs/aws/new-session-manager/)