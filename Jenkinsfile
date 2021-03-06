
/** Builds images for the Grouper Training Env
 *  Each class has a set of images. Each class's image sets are independent, but within 
 *  a class each image builds upon the prior exercise. Therefore all images are built
 *  first and then push to the repo. Order of the build is important, but ordering of the 
 *  image pushes are not. If an image is missing the extra layers get pushed, then the
 *  missing layers only get tagged when they are pushed.
**/


/** Each class has a set of modules with a set of steps. 
 *  For examples, ex101.1.1, ex101.1.2, ex101.1.3, ex101.2.1, ex101.2.2, etc.
 *  Each step is an image.
 *  The exceriseSets has the class name and an array of the number of steps for module.     
**/
exceriseSets = [
//    'ex101' : [3, 2],
//    'ex201' : [2, 2, 5, 6],
//    'ex301' : [2, 2, 5, 6],
    'ex401' : [6, 9]
]

pipeline {
    agent any
    environment { 
        maintainer = "t"
        imagename = 'g'
        tag = 'l'
    }
    stages {
        stage('Setting build context') {
            steps {
                script {
                    maintainer = maintain()
                    imagename = imagename()
/*                    if(env.BRANCH_NAME == "master") {
                       tag = "latest"
                    } else {
                       tag = env.BRANCH_NAME
                    }
*/
                    if(!imagename){
                        echo "You must define an imagename in common.bash"
                        currentBuild.result = 'FAILURE'
                     }
                    sh 'mkdir -p bin'
                    sh 'mkdir -p tmp'
                    dir('tmp'){
                      git([ url: "https://github.internet2.edu/docker/util.git", credentialsId: "jenkins-github-access-token" ])
                      sh 'ls'
                      sh 'mv bin/* ../bin/.'
                    }
                }  
             }
        }    
        stage('Clean') {
            steps {
                script {
                   try{
                     sh 'bin/destroy.sh >> debug'
                   } catch(error) {
                     def error_details = readFile('./debug');
                     def message = "BUILD ERROR: There was a problem building the Base Image. \n\n ${error_details}"
                     sh "rm -f ./debug"
                     handleError(message)
                   }
                }
            }
        } 
        stage('Build Base') {
            steps {
                script {
                    docker.withRegistry('https://registry.hub.docker.com/',   "dockerhub-${maintainer}") {
                        def baseImg = docker.build("${maintainer}/${imagename}:base", "--no-cache --pull base")
                        baseImg.push("base")
                    }
                }
            }
        }
        stage('Build Others') {
            steps {
                script {
                    docker.withRegistry('https://registry.hub.docker.com/',   "dockerhub-${maintainer}") {
                        def tagSet = generateTagSet()
                        def builds = build(tagSet)

                        if(env.BRANCH_NAME == "master") {
                            //builds.each{ k, v -> echo ("push ${k}") } //for local testing
                            builds.each{ k, v -> v.push(k) }

                        } else {
                            echo 'skipping push, since the SCM branch is not master'
                        }
                    }
                }
            }
        }
        stage('Notify') {
            steps{
                echo "$maintainer"
                slackSend color: 'good', message: "${maintainer}/${imagename} set pushed to DockerHub"
            }
        }
    }
    post { 
        always { 
            echo 'Done Building.'
        }
        failure {
            // slackSend color: 'good', message: "Build failed"
            handleError("BUILD ERROR: There was a problem building ${maintainer}/${imagename}:${tag}.")
        }
    }
}


def maintain() {
  def matcher = readFile('common.bash') =~ 'maintainer="(.+)"'
  matcher ? matcher[0][1] : 'tier'
}

def imagename() {
  def matcher = readFile('common.bash') =~ 'imagename="(.+)"'
  matcher ? matcher[0][1] : null
}

def generateTagSet() {
    def tagSet = []

    exceriseSets.each{ course, stepCountPerExercise -> 
        stepCountPerExercise.eachWithIndex {stepCount, exIndex ->
            for (int step = 0; step < stepCount; step++) {
                tagSet.add("${course}.${exIndex+1}.${step+1}")
            }
    
            tagSet.add("${course}.${exIndex+1}.end")
        }
    }

    tagSet
}

def build(tagSet) {
    def builds = [:]

    for (String tag : tagSet) {
        def baseImg = docker.build("${maintainer}/${imagename}:${tag}", "--no-cache ${tag.tokenize('.')[0]}/${tag}")
        echo "built ${tag}; adding to the push queue"
        builds.put(tag, baseImg);
    }

    builds
}

def handleError(String message){
  echo "${message}"
  currentBuild.setResult("FAILED")
  slackSend color: 'danger', message: "${message}"
  //step([$class: 'Mailer', notifyEveryUnstableBuild: true, recipients: 'chubing@internet2.edu', sendToIndividuals: true])
  sh 'exit 1'
}
