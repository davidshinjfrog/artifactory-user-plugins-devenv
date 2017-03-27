#!/usr/bin/env groovy
import java.io.File
import static groovy.io.FileType.FILES

// Instructions for setting up Jenkins 
// 1. Setup slaves - for this example four Jenkins slaves are used for the unit tests to complete within 40 minutes. 
// 2. Set number of executor = 1 for each slave.  More than one causes multiple workspaces to be created and Jenkins will not be able to retrieve the test results.  
// 3. Change the labels below with the name of the Jenkins slaves. 
// 4. Jenkins master is 'master'
// 5. Run this Jenkins file in the Jenkins pipeline. 
// 6. Change the credential id for repo21.  The credential id is configured in Jenkins. 
// 7. Install Git on the slaves

def pluginList = []

// Jenkins Parallel Queues
def stepsForParallel = [:]
def cleanupSlaves = [:]
def builders =[:]

// four Jenkins slave - set number of executor = 1
def labels =['jslave4', 'jslave2', 'jslave3']

//
// create list of all user plugins 
//

node ('master') {
    stage ('Create Plugin List') {
        git url: 'https://github.com/JFrogDev/artifactory-user-plugins.git'
        copyPluginList(userPluginList(new File("${env.WORKSPACE}")))
        printUserPluginList ()
    }

    stage ('Prepare Slaves') {
        println "Prepare and start artifactory branch '${ARTDEVENVBRANCH}' on the slaves"
        def server = Artifactory.server SERVER_ID

        dir ('artifactory-user-plugins-devenv') {
            git url: 'https://github.com/JFrogDev/artifactory-user-plugins-devenv.git', branch: "${ARTDEVENVBRANCH}"
            sh "cd ${env.WORKSPACE}/artifactory-user-plugins-devenv"
            withCredentials([usernamePassword(credentialsId: REPO21_CRED, passwordVariable: 'repoPwd', usernameVariable: 'repoUser')]) {
                insertCredProperties ('username=xxx', "username=${repoUser}")
                insertCredProperties ('password=yyy', "password=${repoPwd}")
            }
            fetchArtifactoryLicense (server)
            stash name: "gradle-property", includes: "gradle.properties"
            stash name: "artifactory-lic", includes: "artifactory.lic"

            try {
                for (x in labels) {
                    def label = x
                    builders[label] = buildersOnSlaves(label, ARTDEVENVBRANCH)
                }
                parallel builders
            } catch (Exception e) {
                println "Caught exception at preparing slaves"
            }
        }
    }
}

//
// Run unit tests on all the availabe Jenkins Slaves
//
try {
    node ('master') {
        stage ("Unit Test") {
            println "Unit Test started"
            def numberPlugins = getPluginSize()        
            for (i = 0; i < numberPlugins; i++ ) {
                def stepName = getPluginIndex(i)
                stepsForParallel[stepName] = runUserPluginTest(stepName)
            }
            parallel stepsForParallel
        }
    }
} catch (Exception e) {
    println "Caught exception at Unit Test: ${e.message}"    
} finally {
    node ('master') {
        stage ("Clean Up") {
            println "Clean up started"
            for (x in labels) {
                def label = x
                cleanupSlaves[label] = cleanUpSlaves(label)
            }
            parallel cleanupSlaves    
        }
    }
}

node ('master') {
    stage ('Deploy') {
        reportOnTestsForBuild()
    }
}

//
// utilities
//

def fetchArtifactoryLicense (server) {
    def downloadSpec = """{
        "files": [
                {
                    "pattern": "artifactory-lic/artifactory.lic",
                    "target": "${env.WORKSPACE}/artifactory-user-plugins-devenv/"
                }
            ]
        }"""
    server.download spec: downloadSpec
}

def getPluginSize () {
    return pluginList.size()
}

def getPluginIndex (index) {
    return pluginList[index]
}

def printUserPluginList () {
    for (i = 0; i < pluginList.size(); i++) {
        println pluginList[i]
    }
}

@NonCPS
def userPluginList (filepath) {
    def list = []
    filepath.eachFileRecurse(FILES) { file ->
        if (file.name.endsWith('.groovy') && file.name != 'setup.groovy' && !file.name.endsWith('Test.groovy')) {
           String prefix = "${file.name.minus('.groovy')}"
           String tmp = file.absolutePath.minus("/$file.name")
           String fullPrefix = tmp.minus("$filepath.absolutePath/")
           list << fullPrefix                
        }
    }
    return list
}

def copyPluginList (pList) {
    pluginList = pList
}

def buildersOnSlaves (label, artbranch) {
    return {
        node (label) {
            try {

            } catch (Exception e) {
                println "Info: Exception caught while cleanup before preparing Artifactory"
            }
            dir ('artifactory-user-plugins-devenv') {
                git url: 'https://github.com/JFrogDev/artifactory-user-plugins-devenv.git', branch: artbranch
                unstash "gradle-property"
                unstash "artifactory-lic"
                def workspace = pwd()
                sh "cp artifactory.lic $workspace/local-store/"
                sh "docker build --build-arg artifactoryVersion=_latest --build-arg artbranch=${artbranch} --build-arg Build_NUMBER=${env.BUILD_NUMBER} -f $workspace/Dockerfile -t user-plugin:test ."
            }
        }
    }
}


def runUserPluginTest (pluginName) {
    return {
        node ('docker') {
            dir ('artifactory-user-plugins-devenv') {
                echo "User Plugin Test: ${pluginName}"
                sh "mkdir -p build/reports/tests"
                def containerPlugin = pluginName.toLowerCase().replaceAll("\\/", "-")
                try {
                    //def cn = pluginName.toLowerCase().replaceAll("\\/", "-").concat(${env.BUILD_NUMBER})
                    sh "docker run -e pluginName=${pluginName} --name ${containerPlugin} user-plugin:test"
                } catch (Exception e) {
                    println "Caught Exception with plugin ${pluginName}. Message ${e.message}"
                } finally {
                    sh "docker cp ${containerPlugin}:/data/artifactory-user-plugins-devenv/build/reports/tests/ build/"
                    sh "docker cp ${containerPlugin}:/data/artifactory-user-plugins-devenv/build/test-results build/"
                    sh "docker rm ${containerPlugin}"
                }
            }
        }
    }
}

@NonCPS
def insertCredProperties (toReplace, byString) {
    println "Updating gradle.properties"
    try {
        def property = new File("${env.WORKSPACE}/artifactory-user-plugins-devenv/gradle.properties")
        processFileInPlace (property) { text ->
            text.replace(toReplace, byString)
        }
    } catch (Exception e) {
        println "Exception caught while inserting credentials. Exception: " + e.message 
        throw e
    }
}

@NonCPS
def processFileInPlace (file, Closure processText) {
    def text = file.text
    file.write(processText(text))
}


def cleanUpSlaves (label) {
    return {
        node (label) {
            try {
                step([$class: 'JUnitResultArchiver', testResults: '**/build/test-results/*.xml'])
            } catch (Exception e) {
                println "Caught exception while archiving test results. ${e.message}"
            } finally {
                    sh "cd ${env.WORKSPACE}/artifactory-user-plugins-devenv; ./gradlew stopArtPro"
                    sh "cd ${env.WORKSPACE}/artifactory-user-plugins-devenv; ./gradlew cleanArtPro"
                    sh "cd ${env.WORKSPACE}/artifactory-user-plugins-devenv; rm gradle.properties"
            }
        }
    }
}

@NonCPS
def reportOnTestsForBuild () {
    def failedTests = []
    def build = manager.build
    if (build.getAction(hudson.tasks.junit.TestResultAction.class) == null) {
        return ("No Tests")
    }
    def result = build.getAction(hudson.tasks.junit.TestResultAction.class).result
    if ((result == null) || (result.failCount < 1)) {
        println "No test results"
    } else {
        println "Failed test count: " + result.getFailCount() 
        println "Passed test count: " + result.getPassCount()
        failedTests = result.getFailedTests()
        failedTests.each { test ->
            println test.name
        }
    }
}
