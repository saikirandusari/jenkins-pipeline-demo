jettyUrl = 'http://localhost:8081/'

def servers

def choice = new ChoiceParameterDefinition('AGENT', ['mock', 'docker', 'ec2'] as String[], 'Where do you want to build this?')
properties ([[$class: 'ParametersDefinitionProperty', parameterDefinitions: [choice]]])

stage 'Build'
//def nodeLabel = input message: 'Where do you want to run this?', parameters: [choice]

echo "The agent is $AGENT"
node("$AGENT") {
   checkout scm
   servers = load 'servers.groovy'
   def mvnHome = tool 'M3'
   sh "${mvnHome}/bin/mvn clean package"
   dir('target') {stash name: 'war', includes: 'x.war'}
}

stage 'QA'
parallel(longerTests: {
    runTests(servers, 30)
}, quickerTests: {
    runTests(servers, 20)
})

stage name: 'Staging', concurrency: 1
node {
    servers.deploy 'staging'
}

input message: "Does ${jettyUrl}staging/ look good?"
try {
    checkpoint('Before production')
} catch (NoSuchMethodError _) {
    echo 'Checkpoint feature available in CloudBees Jenkins Enterprise.'
}

stage name: 'Production', concurrency: 1
node {
    sh "wget -O - -S ${jettyUrl}staging/"
    echo 'Production server looks to be alive'
    servers.deploy 'production'
    echo "Deployed to ${jettyUrl}production/"
}

def mvn(args) {
    sh "${tool 'M3'}/bin/mvn ${args}"
}

def runTests(servers, duration) {
    node {
        checkout scm
        servers.runWithServer {id ->
            mvn "-o -f sometests test -Durl=${jettyUrl}${id}/ -Dduration=${duration}"
        }
    }
}
