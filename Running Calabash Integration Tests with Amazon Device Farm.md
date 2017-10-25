## Running Calabash Integration Tests with Amazon Device Farm

In part one of this series you learned how to write Calabash scripts for your UI-level Android testing. In this entry we’ll cover how to run these tests as part of a continuous or nightly build cycle. To do so we’ll use a service called Amazon Device Farm (ADF) and the Jenkins automation server. 

## Amazon Device Farm

[Amazon Device Farm][1] is a service that hosts thousands of physical Android and iOS devices and exposes endpoints to interact with these devices. One of these endpoints accepts your Calabash test suite and application APK, runs your tests on a physical device, and then returns the result of this run. Conveniently, Device Farm returns not only a pass or failure boolean but also the complete Calabash test result and optionally screenshots and videos of the run and failure point. ADF provides a web UI for running tests and you can also call the [API directly][2]; however, in order to achieve truly continuous testing we’ll need to run these builds regularly while, with various build triggers and the ability to store the run results and notify stakeholders of failures. For this, we turn to Jenkins.

## Jenkins Continuous Integration Server

[Jenkins CI Server][3] (henceforth Jenkins) is a powerful swiss army knife of a tool that we use for a wide array of purposes at Dimagi. For example:

- Building and hosting our Android APK files
- Running unit tests and linters on our GitHub pull requests
- Building our web application [Formplayer][4]

In short, any process with well-defined inputs and outputs (which should be all of your processes!) that you want to run regularly could be run on Jenkins. Further, Jenkins provides a *ton* of steps and hooks for interacting with builds and there’re thousands of plugins for just about any action. I could write an entire post about Jenkins (and started to do just that) but for our purposes we just need to know that Jenkins can pull source code, run scripts, and publish files (called Artifacts in Jenkins parlance).

## Defining our process

1. Pull source code from GitHub
2. Copy our under-test application APK from another Jenkins job
3. Invoke a script to package our Calabash files into the `feature.zip` format required by ADF
4. Call the Device Farm API with these features, our APK, and the run configuration
5. When the test completes, store and publish the test results (Calabash output, screenshots, videos)

## The setup

## Jenkins Pipeline

Jenkins Pipeline (henceforth Pipeline) is a domain-specific scripting language based on Groovy that allows you to configure your Jenkins builds in code. Previously, all Jenkins builds needed to be configured using the Web UI. Pipeline allows you to write the configuration as code so that you can check it into source control, track the changes, and revert changes when needed. With Pipeline all you need to do within Jenkins is create the job and direct it to the GitHub repository - from there, your script takes over.

## The Code
`Jenkinsfile`:
	node {
	    checkout scm
	
	    sh 'chmod a+x scripts/make_features'
	    sh 'bash scripts/make_features'
	
	    step ([$class: 'CopyArtifact',
	                  projectName: params.cc_android_job,
	                  filter: '**/app-commcare-debug.apk',
	                  fingerprintArtifacts: true,
	                  flatten: true]);
	
	    step([$class: 'AWSDeviceFarmRecorder',
	                        projectName: 'commcare-odk',
	                        devicePoolName: params.pool,
	                        runName: params.stageString,
	                        appArtifact: 'app-commcare-debug.apk',
	                        testToRun: 'CALABASH',
	                        calabashFeatures: 'aws/features.zip',
	                        calabashTags: params.tag,
	                        isRunUnmetered: true,
	                        storeResults: true,
	                        ignoreRunError: false,
	                    ])
	}

and `scripts/make_features`:

	rm -r aws
	mkdir aws
	zip -r aws/features.zip features/*

Let’s go through this line by line.

First, we checkout the code from source control. 
Second, we run a short bash script that makes a new `features.zip` file containing our Calabash features, step definition, and resources
Next, we copy the most recent artifact APK file from the Jenkins job building our Android application.
Finally, we invoke the [AWS Device Farm Plugin][5] for Jenkins - this allows us to make our call to the Device Farm API as a step in Pipeline. We pass in a few configuration parameters here, in particular:

- `projectName` - the name of the AWS project configured earlier
- `devicePoolName` - the name of the device pool to use
- `appArtifact` and `calabashFeatures` - clear from context
- `storeResults` - whether we should save the artifacts in Jenkins
- `ignoreRunError` - whether a Calabash failure should result in a build failure
- `calbashTags` allows you to pass in a set of tags so that ADF will only run a subset of your complete test set. This is a powerful feature that I hope to cover more completely in a future post.

Now we’re set! Once the run completes we can view the results on the run within Jenkins or click through to view directly on ADF.

Now we have a basic UI test runner setup with Jenkins, Pipeline, Calabash, and Amazon Device Farm; however, we’ve not begun to tap into the full set of features provided by these tools. As a few examples, we can:

- Run subsets of our tests using tags
- Retry failed tests using Pipeline’s `retry`
- Run our tests in smaller batches using Pipeline’s `step`
- Setup build triggers and email reports
- Add wildcards for easier release testing

… the list goes on and I hope to cover some of these topics in future posts.   

## Words of Caution

You now have all the information needed to setup your own continuous, integrated testing server on real devices. However, in the spirit of full disclose, getting all of this running was difficult for us and we still have occasional issues with stability. While Calabash and Jenkins in particular are established tools, Pipeline and Amazon Device Farm are very cutting edge and still have weakness in documentation, support, and churn. We frequently submit support tickets, pull requests, or simply use workarounds in order to get full value out of these tools. 

## Conclusion

With that said, at Dimagi we feel that the value we derive from this setup was well worth the upfront development cost and continued maintenance chores. As Clark elaborated on in the previous post, we were able to massively cut down on our QA resource consumption and release turnaround time. We catch bugs as soon as they’re merged into our mobile codebase instead of waiting for a release. We catch many *server* bugs almost as soon as they are deployed because of failures from our Calabash integration tests and nightly builds. This setup is not for everyone, particularly those without the capacity to make the upfront investment and head space for periodic minor bug fixes. However, if you find yourself bogged down by QA cycles, afraid to merge large Android refactors, or ensnared by server-mobile API failures, then I highly recommend this stack. 

[1]:	https://aws.amazon.com/device-farm/
[2]:	http://docs.aws.amazon.com/devicefarm/latest/APIReference/API_ScheduleRun.html
[3]:	https://jenkins.io/
[4]:	https://github.com/dimagi/formplayer
[5]:	%20https://wiki.jenkins.io/display/JENKINS/AWS+Device+Farm+Plugin