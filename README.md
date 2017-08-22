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

### 5 Minute Demo

* If you have access to an AWS account with full administrative privileges you can create a stack containing a VPC and 
    optionally an EC2 instance
* If you have access to an account with limited capabilities and cannot create a VPC you can launch an instance into an
    existing VPC
    
#### Demo Setup

* Ensure Java is installed
* [Install the AWS CLI](http://docs.aws.amazon.com/cli/latest/userguide/installing.html) and then [configure a named profile](http://docs.aws.amazon.com/cli/latest/userguide/cli-multiple-profiles.html). 
    We'll supply the profile name as a parameter to avoid passing around and possibly committing AWS credentials.
    
#### VPC and Instance Creation

1. Run the Gradle wrapper command along with the createStack task and some parameters 
    `./gradlew updateStack -Pprofile=your-named-aws-profile` would be the minimal amount of parameters and would create 
    the stack named app-example in us-east-2 although those values could be overridden using either a local-config.groovy file
    or by passing more parameters to the task ex: `./gradlew createStack -Pprofile=your-named-aws-profile -Pregion=us-west-1 -Pstack=vpc-only -Pinstance=false` 
    the list of possible params to be set can be seen in the ext.params section towards the top of the build.gradle file.
1. The command will return immediately and if you look in the web console you'll see the stack getting provisioned. Unless 
    you changed the instance parameter to true the stack will initially just be the VPC without any instances.
1. We could rerun the command creating the instance this time `./gradlew updateStack -Pprofile=your-named-aws-profile -Pinstance=true` 
    and in short order you'll see an EC2 instance created within the VPC
1. Once completed with this test run the `./gradlew deleteStack -Pprofile=your-named-aws-profile` to terminate all resources 
    prevent the accumulation of charges.

If you find yourself constantly using the same parameter values add a local-config.groovy file to the project and add your 
desired values. Any parameters passed from the command line will override config file values if you need to change them.

----

_local-config.groovy example content_

    project.params.with {
        profile = 'my-aws-cli-profile-name'
        stack = 'test-stack'
    }
    
#### Instance Creation Within an Existing VPC

If you look at the default app-main.yaml template it's actually just calling two other templates, the VPC is created from 
a publicly available VPC template and the application is from the app-main.yml file from this same project. We can execute 
the app template separately to just create the app within an existing VPC.

1. Locate the subnet ID of an existing subnet within a VPC in your account. AWS Console -> VPC -> Subnets and you should 
    see a list of subnets with IDs that look like subnet-11111111. We'll pass this in as an argument to the app.yaml template. 
1. Run the updateStack task again but with parameters to use a different template and set of stack parameters 
    `./gradlew updateStack -Pprofile=your-named-aws-profile -Ptemplate=app.yaml -PstackParamSet=appOnly -Psubnet=SUBNET_ID`
1. Once created we'll test out the [Change Set capability](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-updating-stacks-changesets.html)
    to preview updates before you actually change any resources.
    * Go into the src/main/cloudformation/app.yaml file and change the instance type from a nano to a small `InstanceType: 't2.small'`
    * Run the `./gradlew createChangeSet -Pprofile=your-named-aws-profile -Ptemplate=app.yaml -PstackParamSet=appOnly -Psubnet=SUBNET_ID` and then look in the console to [inspect the change set](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-updating-stacks-changesets-view.html)
    * In this case CloudFormation would modify the existing instance by shutting it down and changing the instance type. 
        This would cause a momentary outage but if this is acceptable you could execute the change set to apply the 
        update or you could discard the change set without affecting any resources. This is a really helpful capability that
        aids in understanding the impact of updates before they occur. Every CloudFormation resource has the impact of updates
        listed at the individual property level. Our example of an [EC2 instance type property update](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-ec2-instance.html#cfn-ec2-instance-instancetype) states 
        that EBS backed instances will require an interruption but the older instance store types would require the full replacement
        of the instance.
1. Once completed with this test run the `./gradlew deleteStack -Pprofile=your-named-aws-profile` to terminate all resources 
    prevent the accumulation of charges.
    
### Demo Wrap-Up

This demo shows how you can interact with components that have already been created such as the VPC component in the 
app-main.yaml template. Notice how the version of the component is preserved within the URL so you can be assured it 
will not change if you're using a release version. The example project itself is still versioned as a SNAPSHOT as seen 
in the gradle.properties file. Once you have a working component you'd use the [release](https://cfn-stacks.com/docs/artifacts3-plugin/latest/index.html#release) 
plugin task to create a tagged release version which would be published to your configured [ArtifactS3 repository](https://cfn-stacks.com/docs/artifacts3-repo/latest/index.html).