
# SpockTestSupport.groovy

## NMB-28395

# Intro
This document describes the contents of SpockTestSupport.groovy and gives some examples of how it might be used.

# Spock Feature Summary
Spock is a test framework for Groovy and Java; its documentation is available here: [https://spockframework.org/spock/docs/1.0/index.html](https://spockframework.org/spock/docs/1.0/index.html)

Spock is intended to make test creation easy AND to encourage easy to read documentation of behavior.  For that reason the groovy files that contain Spock tests are often called “specifications”.

Each Spock specification, a groovy class, should extend Spock’s “Specification” class.  And then tests within the specification are defined using groovy’s “def” statement.  Within Spock those tests are called “feature methods”, because each should define the behavior of a feature.  The names for feature methods should be very descriptive; almost any strings are allowed e.g. “def ‘a simple test with an assertion’”

“Labeled blocks” are another feature of Spock.  They are simple labels that describe familiar test chronology and enforce some simple rules:
* “setup” Must be the first block used, and it may not be repeated.  If you do NOT specify a “setup” block then everything up to the first block is assumed to be setup.
* “when” and “then” always occur together.  “when” describes or implements a required state and “then” describes the conditions that must hold true as a result.  When/then pairs may be repeated. No assert statement is needed in the “then” block, just supply a boolean expression.  For example:

    when:
    def aVariable = 10
    then:
     aVariable == 10
* “cleanup” may only be followed by a where block, and may not be repeated.  It is used as similarly named functions to free up resources etc.  It runs even if the feature method throws an exception!
* “where” always comes last and is used to describe the data that should be used to drive the feature method (test).  Spock where blocks provide for an easy to read tabular format to drive “sets” of variable values for testing multiple scenarios.
* “expect” can used where it makes sense syntactically to combine when/then into one e.g. “expect: Math.abs(-12) == 12”

A common testing problem is how to mimic the behavior of complex classes in order to test some other piece of code.  Spock can mock objects from concrete classes with a simple call to Mock().  For example: “def complexClassInstance = Mock(complexClass)”  With no other settings this would return a default value for any method called (false, 0, or null).

# Purpose of SpockTestSupport.groovy
SpockTestSupport.groovy provides tools  to make testing objects defined in the SDA Platform easier.   There are several utility functions defined such as “jobCompleted” to test if a job-id has completed.  Also SpockTestSupport makes use of setup(), setupSpec(), and cleanupSpec() routines to check the status of the SDA Platform service, login, and logout.
The function templates “doSetupSpec” and “doCleanupSpec” provide hooks to customize set up and clean up.

# Commonly Used Functions

## setupSpec / doSetupSpec
__setupSpec__: Spock runs setupSpec() before running any specification.
setupSpec records start time, sets some summary counters to zero, uses SpockTestSupport internal configuration to login to CD Server using a REST based API, and lastly it runs doSetupSpec() if it is defined.
Internally setupSpec uses
* waitForServer()
* _login()

__doSetupSpec__: In order to customize pre-specification setup, define doSetupSpec.

## setup
If the log level is set to INFO, setup will run runtime and memory stats routines and log the results.
Internally it uses:
* checkServerStats()
* checkSpecsClientStats()

## cleanupSpec / doCleanupSpec
__cleanupSpec__: Spock runs cleanupSpec() after running any specification.
cleanupSpec() runs doCleanupSpec(), if it is defined, logs out of the CD Server and logs how much time the specification took to run.
Internally cleanupSpec uses:
* logout()

__doCleanUpSpec__: In order to customize after specification cleanup, define doCleanupSpec.

## dsl

### dsl(dslScript, args, options)

### dsl(dslScript, args, overwrite, options)

Evaluate the DSL in dslScript with the CD Server.  This can be used to create, modify and delete objects in the CD Server’s database, as well as launch jobs, pipeline, etc. in the CD Server.  The dsl functions return a  JSON object.
* args - is a groovy map object used for variable replacement in the dslString
* options - is a groovy map object used to provide dsl options such as “timeout’
* overwrite - the default, overwrite is false, means that all dsl will be regarded as an update to existing objects when they already exist.  If overwrite is true then non-essential members of objects that are imported will be deleted prior to updating with the values from the import.

Internally, these functions will respond to SERVER_UNAVAILABLE errors by making another request that attempts to wait for the server.  The function also update some summary internal counters.

| `def result = dsl "application 'myapp', projectName: 'myproj'"` | 
| --- |

Internally the dsl functions call
* toJson(json)

## job

### job(outcome,  job-id)

### jobCompleted(job-id)

### jobCompletedWithWarning(job-id)

### jobFailed(job-id)

### jobSucceeded(job-id)

These functions are used to check job status.  Internally they use DSL and wait for the job (job id) to complete.  The specified outcome is compared to the job status and the functions return true or false.
The similarly named functions are short-hand for various outcomes:
* jobCompleted checks for a null outcome
* jobCompletedWithWarning checks for outcome = “warning”
* jobFailed checks for outcome = ‘error’
* jobSucceeded checks for outcome = ‘success’

# Pipeline Compliance Example
    package com.electriccloud.spec  
    
    import org.junit.jupiter.api.Tag  
  
    @Tag("com.electriccloud.spec.categories.CoreDeployTests")  
    class RunCatalogItemTransactionSpec  
        extends SpockTestSupport {  
  
        def "pipeline compliance check"(projectNameToTest, pipelineNameToTest) {  
        def pipelineToTest  
        def stagesToTest  
        def args = [projectName: projectNameToTest]  
        args << [pipelineName: pipelineNameToTest]  
        given: "One or more pipelines are already created"
        
        when: "pipeline is loaded"  
        def pipelineResult = dsl("""  
    getPipeline projectName: '$args.projectName', pipelineName: '$args.pipelineName'  
    """, args)  
  
        then: "pipeline loads"  
        waitUntil {  
            assert pipelineResult?.pipeline  
            pipelineToTest = pipelineResult.pipeline  
            assert pipelineToTest  
        }  
  
        and: "pipeline is compliant"  
        pipelineToTest.projectName ==~ /compliance.*/ // under proper project  
        pipelineToTest.pipelineName ==~ /test.*/ // name is test*  
        pipelineToTest.stageCount == "2" // it has 2 stages  
        pipelineToTest.stageCount.toInteger() == 2 // OR ... it has 2 stages  
        
        when: "stages are loaded"  
        def stagesResult = dsl( """  
    getStages projectName: '$args.projectName', pipelineName: '$args.pipelineName'  
    """, args)  
  
        then: "stages load"  
        waitUntil {  
            assert stagesResult?.stage  
            stagesToTest = stagesResult.stage  
            assert stagesToTest  
        }  
       and: "stages are compliant"  
       stagesToTest[0].stageName == "dev"  
       stagesToTest[0].taskCount == "2"  
       stagesToTest[0].gate[0].gateType == "PRE"  
       stagesToTest[0].gate[0].manualApproval == "1"  
       stagesToTest[0].gate[1].gateType == "POST"  
       stagesToTest[1].stageName == "qa"  
       stagesToTest[1].taskCount == "2"  
       stagesToTest[1].gate[0].gateType == "PRE"  
       stagesToTest[1].gate[1].gateType == "POST"  
  
       cleanup:  
       //remove any test only artifacts  
  
       where:  
       projectNameToTest     | pipelineNameToTest  
       "compliance project" | "testpipeline"  
      }  
    }



# System Example

    <<< a spock specification which describes the system>>>

# Other Examples

## doCleanupSpec, doSetupSpec
    class ACleanupAndSetupSpec extends SpockTestSupport {

    @Override
    def doSetupSpec() {
        // Runs once before this specification`
        def response = dsl "runProcedure projectName: 'projTest', procedureName: 'procTest'"
        waitUntil {
            assert jobStatus(response.jobId).status == 'completed'
        }
        assert jobStatus(response.jobId).outcome == 'success'
    }

    @Override
    def doCleanupSpec() {
        // Runs once after this specification
        dslFile 'folder/cleanup.dsl'
    }

    def "A Test"() {
    … (continued) …  

doSetupSpec is provided as a means to run initialization before any tests are run.  Spock is designed to run setupSpec before the tests that make up the specification.  SpockTestSupport uses setupSpec to create connections to the SDA platform server and login; do not override setupSpec.  doSetupSpec runs after SpockTestSupport’s setupSpec routine.

The counterpart that performs tear-down logic runs after all tests have run is doCleanupSpec.  It runs BEFORE Spock’s “native” cleanupSpec routine.  Similarly, SpockTestSupport uses cleanupSpec to logout and log “time taken”; do not override cleanupSpec.

To run your own code during spec setup and tear down (in conjunction with SpockTestSupport) use doSetupSpec and doCleanupSpec.  If you override setupSpec or cleanupSpec you will lose SDA platform setup logic.

## cleanup, setup
    class BCleanupAndSetupTests extends SpockTestSupport {

    def cleanup() {
        // This runs after every test
        //  and overwrites the cleanup procedure
        //  defined in SpockTestSupport

    }

    def setup() {
        // This runs before every test
        //  and overwrites the setup procedure
        //  defined in SpockTestSupport
    }
    def "A Test"() {
    … (continued) …


Spock runs setup before each test and cleanup after each test.  Feel free to override these to perform pre-test and post-test logic.

## deleteProjects, waitUntil
    def "waitUntil uses jobStatus & jobStep, end with deleteProjects"() {

        when: "procedure with script is invoked"
        def response = dsl "runProcedure projectName: '$projectName', procedureName: 'testProc'"

        then: "procedure is expected to complete successfully"
        def jobResult
        def jobStepResult
        waitUntil {
           jobResult = jobStatus(response.jobId)
           assert jobResult.status == 'completed'
           jobStepResult = jobStep (response.jobId, 'step1')
        }
        assert jobStepResult.outcome == 'success'
        assert jobResult.outcome == 'success'

        cleanup:
        deleteProjects([proj: projectName])

    }


deleteProjects is a convenience to delete a list of projects by means of dsl.  This is frequently used to remove temporary data.

waitUntil is a means to repeat a block of groovy code until the “assert” conditions within the block are met or waitUntil exceeds its timeout.  By default waitUntil’s timeout is 120 seconds.  You may use a different timeout by supplying seconds as the first parameter:  e.g. waitUntil 240, {assert true}

# Reference

## SpockTestSupport.groovy

### actualParams(objectType, objectId)
Use DSL to get actual parameters for an object by object type and id.  Return actual parameters as a linked hash map.

### actualParams(jobStepId)
Use DSL to get actual parameters for a job step by job step id.  Return actual parameters as a linked hash map.

### assertAdminLogin()
Use login() to assert that the administrator username logged in. Uses the default configuration for admin username value.

### assertError(def result, def errorCode, def errorMsg)
Assert that the result (from HTTP) contains an error code and an error message.

### assertLogin(userName, password)
Use login() to assert that the username logged in.

### assertResourceStep(allJobSteps, pathToStep)
Return the resource step from allJobSteps.

### assertSuccessfullJobSteps(allJobSteps)
Return true is allJobSteps were successful.

### assertUsedDifferentResources(allJobSteps)
Return true is allJobSteps uses 1 or more resources.

### checkServerStats()
Log runtime and memory statistics from the CD Server.

### checkSpecsClientStats()
Log runtime and memory statistics from the client running Spock tests.

### checkStats()
short hand to call checkServerStats and checkSpecsClientStats

### cleanupSpec()
Spock runs cleanupSpec() after running any specification.
cleanupSpec() runs doCleanupSpec(), if it is defined, logs out of the CD Server and logs how much time the specification took to run.
Internally cleanupSpec uses:
* logout()

### createUser(userName, email, fullUserName, password)
Use DSL to create a userName.

### deleteCIBuilds(builds)
Use DSL to delete CI jobs where builds is an array of maps as follows:

    [jobname: “the job name”, buildName: “the build name”],


### deleteDataRetentionPolicies(policies)
Use DSL to delete data retention policies where policies is an array of policy names.

### deleteProjects(projectsMap, foreground = false)
Use DSL to delete projects where projectsMap is an array of maps as follows:


    [projectName: “the project name”],

### deleteResources = { resourcesMap }
Use DSL to delete resources where resourcesMap is an array of maps as follows:


    [resourceName: “the resource name”],


### deleteSearchFilter = {objectType, searchFIlterName, foreground)
Use DSL to delete a search filter by object type and search filter name.

### deleteUser(userName)
Wait until the userName is deleted.  Uses waitUntil()

### deleteWorkspaces(workspaces)
Use DSL to delete work spaces where workspaces is an array of maps as follows:


    [workspaceName: “the work space name”],


### deleteZone(zoneName)
Use DSL to delete the zone.

### doCleanupSpec()
A stub so that users may code their own after specification setup.

### doPublishArtifact(artifactInfo)
Use file “artifact/publish_artifact.dsl” to publish artifactInfo.artifactName with version artifactInfo.artifactVersion to ...

### doSetUpSpec()
A stub so that users may code their own before specification setup.

### dsl(String script, args, Closure<?>options)
Evaluate DSL with the CD Server.  This can be used to create, modify and delete objects in the CD Server’s database, as well as launch jobs, pipeline, etc. in the CD Server.  Calls dsl(script, args, false, options).  Returns a JSON object.

    dsl "deleteProject 'myproj'"

### dsl(String script, args, boolean overwrite, Closure<?>options)
Evaluate DSL with the CD Server.  This can be used to create, modify and delete objects in the CD Server’s database, as well as launch jobs, pipeline, etc. in the CD Server.  Calls  dslWithRawResponse(dslScript, args, overwrite, options). Returns a JSON object.

### dslWithRawResponse(String script, args, Closure<?>options)
Evaluate DSL with the CD Server.  This can be used to create, modify and delete objects in the CD Server’s database, as well as launch jobs, pipeline, etc. in the CD Server.  Calls  dslWithRawResponse(dslScript, args, overwrite, options). Returns a String.

### dslWithRawResponse(String script, args, boolean overwrite, Closure<?>options)
Evaluate DSL with the CD Server.  This can be used to create, modify and delete objects in the CD Server’s database, as well as launch jobs, pipeline, etc. in the CD Server. Returns a String.

### dslWithXmlResponse(String dslScript, args, LinkedHashMap<Object, Object>options)
Evaluate dslScript on the CD Server and return XML.  Calls dslWithRawResponse and jsonToXMLConverter.

### findObjects(def type, def filters)
Call findObjects on the CD Server using DSL where type is the type of object to find and filters is an array of maps as follows:

    [operator: “operator”, propertyName: “property name”, operand1: “operand”],

Operator may be “and”, “or”, …
If operator is “and or “or” then propertyName and operand1 are ignored

### flowRuntimeActualParams(projectName, flowRuntimeName)
Use DSL to get flow runtime actual parameters by project name and flow runtime name.

### flowRuntimeCompleted(outcome, flowRuntimeId)
Use DSL to wait for completion of flowruntime by Id.  Assert that the outcome is as specified.

### flowRuntimeCompletedWithWarning(flowRuntimeId)
Use DSL to wait for completion of flowruntime by Id.  Assert that the outcome is ‘warning’.

### flowRuntimeFailed(flowRuntimeId)
Use DSL to wait for completion of flowruntime by Id.  Assert that the outcome is ‘error’.

### flowRuntimeStateActualParams(flowRuntimeStateId)
Use DSL to get flow runtime state actual parameters by flow runtime state Id.

### flowRuntimeSucceeded(flowRuntimeId)
Use DSL to wait for completion of flowruntime by Id.  Assert that the outcome is ‘success’.

### getJobSteps(jobid)
Use DSl to get the job steps for a job id. Return an array of job steps.

### getRetrievedArtifacts(jobId, resourceName)

### job(outcome, jobid)
Use DSL to wait for the {job id} to complete.  After 5 minutes check the job status.
job {outcome} {job id} is used to check against any supplied outcome.  If an outcome is specified, retrieve the job status and compare it to the outcome.  Assert that the outcome equals the job status.

### jobCompleted(jobid)
jobCompleted checks for a null outcome

### jobCompletedWithWarning(jobid)
jobCompletedWithWarning checks for outcome = “warning”

### jobDetail(jobid)
Use DSL to get job details for jobid.

### jobFailed(jobid)
jobFailed checks for outcome = ‘error’

### jobSucceeded(jobid)
jobSucceeded checks for outcome = ‘success’

### jobStatus(jobid)
Get the job status for jobid.

### jobStep(jobid, jobstepname)
Get the job step for jobid by name.  Assert that only 1 job step is returned.

### jobSteps(jobid, jobstepname)
Get the job steps for jobid by name.  This can return any number of job steps.

### jsonFile(String filename)
Return a JSON object from the contents of a local file.

### login(userName, password)
Login to the CD Server using userName and password.

### logout()
Log out of CD Server.

### post(uri, payload, options)
HTTP POST the payload to uri. options:
* If options.serverRestartInProgress is false and the response code is SERVICE_UNAVAILABLE.  Wait up to 20 minutes for the server, login again, set options.serverRestartInProgress to true and retry the POST
* If options.ignoreStatusCode is false log any error code.  If it is possible to find the error message in the response, then fail the test with that message.

### publishArtifact(artifactInfo)
If the artifactInfo.artifactName is not found on the CD Server then publish it using doPublishArtifact().

### randomize(str, int len)
Create a string of maximum length len after appending a random UUID onto str.

### randomize(str)
Create a string by appending a random UUID onto str.

### retryOnConnectError(Closure closure)
Run the groovy code in closure for the default timeout period.

### setupMavenPlugin()
Use DSl to get the maven plugin.  If it is not find attempt to load it using the DSL from file “plugin/EC-Maven.dsl”

### setUpSpec()
Runs before every Spock specification (class)

### String loadFromClassPath(fileName) throws IOException
Attempt to load fileName from the SpockTestSupport class path.

### toJson(String text)
Convert text to a JSON object.

### verifyJobStepAndResource(jobId, jobStepName, resourceName)

### waitForServer(timeoutInMinutes)
Return when the server responds with status OK using the /server/status API.

### waitUntil(conditions)
Issue waitUntil using the default poll.timeout value.

### waitUntil(timeout, conditions)
Issue the supplied conditions using the PollingConditions object until the conditions or timeout.
Example from “deleteProjects(projectsMap, foreground)”:

    waitUntil timeout, {`
        dsl ("""
        deleteProject projectName: '$value', foreground: '$foreground'
        """, [:], options)
    }

### workflowCompleted(projectName, workflowName)

## PollingConditions.java

### void call(Closure<?> conditions) throws InterruptedException
An alias for “eventually”

### void call(double seconds, Closure<?> conditions) throws InterruptedException
An alias for “within”

### void eventually(Closure<?> conditions) throws InterruptedException
Calls “within” with a timeout of 1 second.

### void within(double seconds, Closure<?> conditions) throws InterruptedException
Evaluate the Groovy statements contained in “conditions” repeatedly until they finish or the timeout expires.

### double getDelay()
Return the delay between each evaluation of the conditions. Default is 0.1 seconds.

### double getFactor()
Return the factor by which the delay grows after each time the conditions are evaluated. Default is 1.

### double getInitialDelay()
Returns the initial delay (in seconds) before first evaluating the conditions. Defaults to zero seconds.

### double getMaxDelay()
Return the maximum delay value, which computed delay can never exceed.

### double getTimeout()
Return the timeout under which the conditions must finish.  Defaults to 1 second.

### void setDelay(double seconds)
Set the delay in seconds between successive evaluations of the conditions.

### void setFactor(double factor)
Set the factor by which the delay grows between successive evaluations of the conditions.

### void setInitialDelay(double seconds)
Set the delay in seconds before the first evaluation of the conditions.

### void setMaxDelay(double maxDelay)
Set the maximum delay in seconds.

### void setTimeout(double seconds)
Set the timeout under which the conditions must finish.
