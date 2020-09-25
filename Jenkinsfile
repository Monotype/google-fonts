//properties([parameters([choice(choices: "full\nincremental", description: 'Select build type.', name: 'build_type')]), [$class: 'JiraProjectProperty']])

def date = ''
def branch = 'office-fonts'
def files = [:]

def getDate() {
    return sh(returnStdout: true, script: "date '+%Y-%m-%d %H:%M:%S'").trim()
}

def split(array, n) {
    def partitions = []
    int size = array.size()
    int k = size / n
    int m = size % n

    for(int i=0; i < n; i++) {
        def start = i * k + (i < m ? i : m)
        def end = (i + 1) * k + (i +1 < m ? i +1 : m)
        if (start >= size) {
            break
        }
        partitions << array[start..end-1]
    }
    return partitions
}


node('fonttools-dev') {
    properties([disableConcurrentBuilds()])

    stage('Initialize') {
        date = getDate()
        print(date)
    }

    stage('Get docker image') {
        withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: 'Artifactory-Credentials-for-Jenkins-User', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD']]) {
            sh """
            docker login -u $USERNAME -p $PASSWORD docker-artifact.monotype.com
            docker pull docker-artifact.monotype.com/fonttools/fonttoolkit:latest
            docker logout docker-artifact.monotype.com
            """
        }
    }

    stage('Clean workspace before build') {
        try {
            if (false) {
                deleteDir()
            }
        } catch (e) {
            // can happen if old builds expire before next one runs; just proceed.
            currentBuild.result = 'SUCCESS'
        }
    }

    stage('Git checkout') {
        try {
            scm_data = checkout([
                $class: 'GitSCM',
                branches: [[name: branch]],
                gitTool: 'Default',
                userRemoteConfigs: [[credentialsId: 'jenkins-github', url: 'https://github.com/Monotype/google-fonts.git']]]
            )
        } catch(e) {
            currentBuild.result = 'FAILURE'
            slackSend(channel: "fonttools-jenkins",
                      color: "#bf0000",
                      message:":linux: google-fonts failed during Git checkout.")
            throw(e)
        }
    }

    stage('Update Git Clone') {
        withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: 'jenkins-github', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_TOKEN']]) {
            sh """
                echo 'echo \$GIT_TOKEN' > .git-askpass
                chmod +x .git-askpass
                export GIT_ASKPASS=\$PWD/.git-askpass
                git config --local credential.username ${env.GIT_USERNAME}
                git config --local user.email \"fonttools-jenkins@monotype.com\"
                git config --local user.name \"Font Tools Jenkins\"
                # we are not on branch ${branch} !!!
                git checkout origin/${branch}
                git pull origin ${branch}
                git remote -v | grep -wq upstream || git remote add upstream git://github.com/google/fonts.git
                git fetch upstream
                git merge upstream/master -m \"update from Google fonts ${date}\"
            """
            if (true || params.build_type == 'full') {
                files = sh(script: "find . -iregex '.*\\(\\.ttf\\|\\.cff\\|\\.otf\\)\$' | cut -c3-", returnStdout: true).split("\n")
            } else {
                files = sh(script: "git diff origin/${branch} --name-only --diff-filter=d | grep -iE '(\\.otf|\\.ttf|\\.cff)\$' || x=0", returnStdout: true).split("\n")
            }
            sh """
                export GIT_ASKPASS=\$PWD/.git-askpass
                git push origin HEAD:${branch}
            """
        }
    }
}