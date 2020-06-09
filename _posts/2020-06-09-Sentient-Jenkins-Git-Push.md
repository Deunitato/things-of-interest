---
published: true
---
# Things done 
- Jenkins push to git
- Jenkins pull only when tag
- Running packages when it is not in the repository


## Base code

```

def namespace = "default"
pipeline {

    environment{
     PROJECT_TITLE = 'science-experiments-divya'
     APP_NAME = 'taxonomy'

     APP_TAG = "0.1.${env.BUILD_NUMBER}" //change this to $env.BRANCH_NAME or $env.BUILD_NUMBER
     CLUSTER = "cluster-1"
     CLUSTER_ZONE = "asia-southeast1-a"
     IMAGE_TAG = "gcr.io/${PROJECT_TITLE}/${APP_NAME}:${APP_TAG}"

     JENKINS_CRED = "${PROJECT_TITLE}"
     GOOGLE_APPLICATION_CREDENTIALS = credentials('gcp_service_account_key_sentient');
    }
agent any
      stage('Install application dependencies'){
          steps{

                sh """
                apt-get -y update
                apt -y install python3-pip
                apt -y install python-pip
                apt install -y build-essential wget zlib1g-dev libncurses5-dev libgdbm-dev libnss3-dev libssl-dev libreadline-dev libffi-dev
                wget https://www.python.org/ftp/python/3.7.7/Python-3.7.7.tgz
                tar -xf Python-3.7.7.tgz
                cd Python-3.7.7
                ./configure --enable-optimizations
                make altinstall
                python3 -m pip install virtualenv
                """

                 sh 'python3.7 --version'
                dir('Alfred'){
                  sh """
                  python3.7 -m pip install -e .
                """
                 }
                sh 'python3.7 -m pip install -r requirements.txt'
                dir('deploy'){
                  sh """
                    python3.7 Alfred_script.py
                    """
                }

          }
      }
  }
}
```

# Jenkins push to git

Plugins:
- git


## Credentials
Added credentials with the id `github-divya`
Credentials username and password belonged to the creator of the repo

Git Website:https://github.com/science-experiements-divya/taxanomy-trial-jenkins.git

## Code snippet added:
```
      stage('Create Branch & Push Branch') {
            steps {

                withCredentials([usernamePassword(credentialsId: 'github-divya', usernameVariable: 'USER', passwordVariable: 'PASS')]) 
                {
                    sh """
                    git config user.name "${USER}"
                    git config user.email "${USER}@gmail.com"
                    git checkout -b test/${APP_TAG}
                    git add .
                    git commit -m "update repo"
                    git push https://${USER}:${PASS}@github.com/science-experiements-divya/taxanomy-trial-jenkins.git
                    """
                    
                } 
            }

      }
```

## References
[{Issue}](https://stackoverflow.com/questions/38769976/is-it-possible-to-git-merge-push-using-jenkins-pipelinea) , [{Official sample}](https://github.com/jenkinsci/pipeline-examples/blob/master/pipeline-examples/push-git-repo/pushGitRepo.groovy) , [{jenkins git push}](https://stackoverflow.com/questions/53325544/jenkins-pipeline-git-push)

## Issues:
- Infinite loop: Find a way to rewrite the jenkinsfile if not it will keep webhooking or only webhook a certain branch