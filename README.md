## CloudFormation Stacks Example Application

* [Basic Gradle Project Build Overview](https://docs.gradle.org/current/userguide/tutorial_using_tasks.html)
* [CloudFormation Stacks Process Overview](https://cfn-stacks.com/docs/artifacts3-plugin/latest/index.html)

### Project Directory Structure

----
    ├── gradle
    │   └── wrapper
    └── src
        ├── docs
        │   └── asciidoc
        │       └── stylesheets
        └── main
            └── cloudformation
----

* **gradle/wrapper:** [Gradle bootstrap components](https://docs.gradle.org/current/userguide/gradle_wrapper.html). 
    No tool install required, Java is the only prereq.
* **src/docs:** Application documentation generated from this markup using the `docs` task
* **src/main/cloudformation:** One or more CloudFormation templates to be included in the component

### Usage

The project is preconfigured to use the latest version of the ArtifactS3 Gradle plugin which provides
a workflow for authoring and publishing CloudFormation template components. A [full listing of tasks](https://cfn-stacks.com/docs/artifacts3-plugin/latest/index.html#plugin-tasks) 
describes the available features.

### 15 Minute Demo

* If you have access to an AWS account with full administrative privileges you can create a stack containing a VPC and 
    optionally an EC2 instance
* If you have access to an account with limited capabilities and cannot create a VPC you can launch an instance into an
    existing VPC
    
#### Demo Setup

* Ensure Java is installed
* [Install the AWS CLI](http://docs.aws.amazon.com/cli/latest/userguide/installing.html) and then [configure a named profile](http://docs.aws.amazon.com/cli/latest/userguide/cli-multiple-profiles.html). 
    We'll supply the profile name as a parameter to avoid passing around and possibly committing AWS credentials.
    
*Note:* Two items of note with the commands used in the examples below. 

1. The commands all use [Gradle Wrapper](https://docs.gradle.org/current/userguide/gradle_wrapper.html) which will 
download a specific version of Gradle to the local system and install in a temp area. Depending on your OS use 
`./gradlew` for Linux and Mac or `gradlew` on Windows using the gradlew.bat file. 
1. All of the examples use the updateStack task, even when creating a new stack. The plugin I've wrapped has only one 
task but to give the familiar Create/Update/Delete actions I created an aliased createStack task but it's the same as 
calling updateStack.

#### VPC and Instance Creation

1. Run the Gradle wrapper command along with the createStack or updateStack task and some parameters 
    `./gradlew updateStack -Pprofile=AWS_PROFILE_NAME` would be the minimal amount of parameters and would create 
    the stack named app-example in us-east-2 although those values could be overridden using either a local-config.groovy file
    or by passing more parameters to the task ex: `./gradlew createStack -Pprofile=AWS_PROFILE_NAME -Pregion=us-west-1 -Pstack=vpc-only -Pinstance=false` 
    the list of possible params to be set can be seen in the [ext.params section of the build.gradle](build.gradle#L19) file.
1. The command will return immediately and if you look in the web console you'll see the stack getting provisioned. Unless 
    you changed the instance parameter to true the stack will initially just be a VPC without any instances.
1. We could re-run the command creating the instance this time `./gradlew updateStack -Pprofile=AWS_PROFILE_NAME -Pinstance=true` 
    and short thereafter you'll see an EC2 instance created within the VPC
1. Once completed with this test run the `./gradlew deleteStack -Pprofile=AWS_PROFILE_NAME` to terminate all resources 
    prevent the accumulation of charges.

If you find yourself constantly using the same parameter values add a local-config.groovy file to the project and add your 
desired values. The file has been .gitignore'd so you wont accidentally check it in to your repo. Any parameters passed 
from the command line will still override config file values if you need to change them.

_local-config.groovy example content_

    project.params.with {
        profile = 'my-aws-cli-profile-name'
        stack = 'test-stack'
    }
    
#### Instance Creation Within an Existing VPC

If you look at the default [app-main.yaml template](src/main/cloudformation/app-main.yaml) it calls two other templates, 
the VPC is created from a publicly available template and the application is from the [app.yml](src/main/cloudformation/app.yaml) 
file from this same project. An output of the VPC template containing a newly created subnet is passed to the application 
template as a parameter and is used to provision an EC2 instance. We can run just the app.yml file as long as we pass in
the needed subnet ID parameter.

1. Locate the subnet ID of an existing subnet within a VPC in your account. AWS Console -> VPC -> Subnets and you should 
    see a list of subnets with IDs that look like subnet-11111111. We'll pass this in as an argument to the app.yaml template. 
1. Run the updateStack task again but with parameters to use a different template and set of stack parameters 
    `./gradlew updateStack -Pprofile=AWS_PROFILE_NAME -Ptemplate=app.yaml -PstackParamSet=appOnly -Psubnet=SUBNET_ID`
1. Once created we'll test out the [Change Set capability](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-updating-stacks-changesets.html)
    to preview updates before you actually change any resources.
    * Go into the src/main/cloudformation/app.yaml file and change the instance type from a nano to a small `InstanceType: 't2.small'`
    * Run the `./gradlew createChangeSet -Pprofile=AWS_PROFILE_NAME -Ptemplate=app.yaml -PstackParamSet=appOnly -Psubnet=SUBNET_ID` and then look in the console to [inspect the change set](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-updating-stacks-changesets-view.html)
    * In this case CloudFormation would modify the existing instance by shutting it down and changing the instance type. 
        This would cause a momentary outage but if this is acceptable you could execute the change set to apply the 
        update or you could discard the change set without affecting any resources. This is a really helpful capability that
        aids in understanding the impact of updates before they occur. Every CloudFormation resource has the impact of updates
        listed at the individual property level. Our example of an [EC2 instance type property update](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-ec2-instance.html#cfn-ec2-instance-instancetype) states 
        that EBS backed instances will require an interruption but the older instance store types would require the full replacement
        of the instance.
1. Once completed with this test run the `./gradlew deleteStack -Pprofile=AWS_PROFILE_NAME` to terminate all resources 
    prevent the accumulation of charges.

#### Create a Private Template Repo

1. We've already taken care of the prerequisite steps needed to deploy an ArtifactS3 template repository. Clone the project
and run the command listed in the [deployment docs](https://cfn-stacks.com/docs/artifacts3-repo/latest/index.html#deployment)
 (from the artifacts3-repo directory) `./gradlew updateStack -Pprofile=YOUR_PROFILE_NAME -Pstack=STACK_NAME -PDomainName=UNIQUE_S3_BUCKET_NAME` 
1. Once the stack reaches the CREATE_COMPLETE status you'll be able to reference the fully qualified bucket name as your repo
1. At this point you and your account administrator will be the only people able to access templates within the bucket but
    you can later [expand access](http://docs.aws.amazon.com/AmazonS3/latest/dev/example-policies-s3.html) as needed.

#### Publish a Component to a Repo

1. If you've already created a local-config.groovy file add a repo key with a value of the fully qualified bucket name backing 
    the repo we just created `bucket = 'UNIQUE_S3_BUCKET_NAME.s3.us-east-2.amazonaws.com'` there is also a region component in the 
    name that might need adjusting if you're working in a region other than us-east-2. You can also provide the value as a 
    command line param -Prepo=UNIQUE_S3_BUCKET_NAME.s3.us-east-2.amazonaws.com
1. You can use the example-app as is for this test but you're really want to make a copy, remove the .git folder, update 
    the project name in settings.gradle and the group name in build.gradle before reinitializing git to create a component 
    of your own. Once you publish you'll notice that the namespace and version are preserved in the artifact key name which 
    is why you'll want to update to your own settings. Example publishing command: `./gradlew publish -Pprofile=YOUR_PROFILE_NAME -Prepo=UNIQUE_S3_BUCKET_NAME.s3.us-east-2.amazonaws.com`
1. If you look in the S3 bucket shortly thereafter you'd see snapshot and templates keys with the snapshot containing a jar
    file containing the templates with the rest of the keyname preserving the namespace and version information. The templates
    directory also has keynames that preserve the namespace and version but contain the templates which have been unzipped 
    from the jar file by a Lambda function.
1. The path to the unzipped file in the templates directory is what you're referencing when you want to use a published template

#### Version a Component

1. Snapshots are fine for development, you can redeploy as often as needed and downstream users will always pick up the newest
    version. At the point you want to make your templates available for use by others you'll want to publish a final version
    so they can be assured the content won't change without them explicitly upgrading the version in their template.
1. The artifacts3-plugin uses the [Gradle Release Plugin](https://github.com/researchgate/gradle-release) to automate the 
    release process. The process of cutting a release involves the tagging of the final version in your SCM system so to 
    do this you'll need to fork or copy and update the example-app repo as you won't have the rights needed to push updates.
1. The process to cut a release involves checking in and pushing any outstanding changes and then using the `./gradlew release -Pprofile=YOUR_PROFILE_NAME` 
    command to perform the [steps to cut a release](https://cfn-stacks.com/docs/artifacts3-plugin/latest/index.html#release)
1. The release task automatically triggers the publish task so shortly after the release has completed the templates will 
    be available from the release/ key prefix within the S3 bucket which you can then make available to others for use in 
    their templates. *Note:* The default settings of the artifacts3-repo is to delete snapshots after 30 days. This can be 
    disabled or changed to another setting but the idea of a snapshot is that it's just for development use.

### Demo Wrap-Up

This demo covered most of the workflow steps to produce templates and an example application which consumes them. Ensure you've
terminated any running stacks to release resources at the completion of this demo.