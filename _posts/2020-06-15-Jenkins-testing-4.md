---
published: true
---
# Things done today
- Venv in main directory
- git push only yaml and docker


Original jenkins file:
```groovy

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
     GIT_CREDENTIAL_ID = credentials('github-divya')
     GIT_URL = 'science-experiements-divya/taxanomy-trial-jenkins'
     IS_PYTHON_EXIST = fileExists 'Python-3.7.7'
     IS_ENV_EXIST = fileExists '../../.env'
     IS_ALFRED_EXIST = fileExists 'Alfred'

    }
agent any
  stages {
      stage('Get python package'){
        when { expression { IS_PYTHON_EXIST == 'false' } }
        steps{
          sh"""
              cd ..
              apt-get -y update
              apt install -y build-essential wget zlib1g-dev libncurses5-dev libgdbm-dev libnss3-dev libssl-dev libreadline-dev libffi-dev
              wget https://www.python.org/ftp/python/3.7.7/Python-3.7.7.tgz
              tar -xf Python-3.7.7.tgz
          """
        }
      }
      
      stage ('Installing python'){
        when { expression { IS_PYTHON_EXIST == 'false' } }
        steps{
            sh """
                  apt -y install python3-pip
                  apt -y install python-pip
                  cd Python-3.7.7
                  ./configure --enable-optimizations
                  make altinstall
                  ls
                  """
                  sh 'python3.7 --version'
        }
      }

      stage("Attempt start at virtual environment"){
        when { expression { IS_ENV_EXIST == 'false' } }
        steps{
          sh """
              python3.7 -m pip install virtualenv
              python3.7 -m venv env
          """
        }
      }
      stage("Fetch Alfred"){
        when { expression { IS_ALFRED_EXIST == 'false' } }
        steps {
                withCredentials([usernamePassword(credentialsId: 'github-divya', usernameVariable: 'USER', passwordVariable: 'PASS')]) 
                {
                    sh """
                    ls
                    virtualenv .env
                    source .env/bin/activate
                    git config user.name "${USER}"
                    git config user.email "${USER}@gmail.com"
                    git clone https://${USER}:${PASS}@github.com/science-experiements-divya/alfredtest.git
                    cp -r alfredtest/Alfred ./
                    """
                } 
            }
      }

      stage("Installing Alfred"){
        steps{
          sh """
              virtualenv .env
              source .env/bin/activate
              cd deploy/scripts
              python3.7 -m pip install -r alfred-requirements.txt
              """
          sh """
              ls
              virtualenv .env
              source .env/bin/activate
              cd Alfred
              python3.7 -m pip install --ignore-installed -e .
          """
        }
      }
      stage('Running Alfred_script'){
          steps{
               // dir("env"){
                  sh """
                  virtualenv .env
                  source .env/bin/activate
                  cd deploy
                  python3.7 scripts/Alfred_script.py
                """
          }
      }

      stage("Attempt trigger other job"){
        steps {
          build job: 'microservice/testing-pipeline', propagate: true, wait: true
        }
      }

      // stage('Create Branch & Push Branch') {
      //       steps {

      //           withCredentials([usernamePassword(credentialsId: 'github-divya', usernameVariable: 'USER', passwordVariable: 'PASS')]) 
      //           {
      //               sh """
      //               git config user.name "${USER}"
      //               git config user.email "${USER}@gmail.com"
      //               git checkout -b output/${APP_TAG}
      //               git add .
      //               git commit -m "update repo"
      //               git push https://${USER}:${PASS}@github.com/science-experiements-divya/alfred-pipeline-test.git
      //               """
                    
      //           } 
      //       }
      // }

  }
  // post {
  //   always {
  //     dir("${env.WORKSPACE}@tmp") {
  //       deleteDir()
  //     }
  //     dir("${env.WORKSPACE}@script") {
  //       deleteDir()
  //     }
  //     dir("${env.WORKSPACE}@script@tmp") {
  //       deleteDir()
  //     }
  //   }
  // }
}
```


# Venv in main directory
1. Configure global paths
Manage Jenkins > System Configuration:
![jenkins-test-3.PNG]({{site.baseurl}}/img/jenkins-test-3.PNG)

2. Add the `${env.WORKSPACE_DIR}` in the code

Envirnoment snippet:
```
    environment{
     IS_PYTHON_EXIST = fileExists "/${env.WORKSPACE_DIR}/Python-3.7.7"
     IS_ENV_EXIST = fileExists "/${env.WORKSPACE_DIR}/.env"
     IS_ALFRED_EXIST = fileExists "/${env.WORKSPACE_DIR}/Alfred"

    }
```

Installing virtual Environment:
```
      stage("Attempt start at virtual environment"){
        when { expression { IS_ENV_EXIST == 'false' } }
        steps{
           dir("${env.WORKSPACE_DIR}"){
            sh """
                ls
                python3.7 -m pip install virtualenv
                python3.7 -m venv env
            """
           }
        }
      }
```

Navigating in different directories:
```
      stage("Fetch Alfred"){
        when { expression { IS_ALFRED_EXIST == 'false' } }
        steps {
                withCredentials([usernamePassword(credentialsId: 'github-divya', usernameVariable: 'USER', passwordVariable: 'PASS')]) 
                {
                    sh """
                    cd ${env.WORKSPACE_DIR}
                    virtualenv .env
                    source .env/bin/activate
                    cd ${env.WORKSPACE}
                    rm -r alfredtest
                    git config user.name "${USER}"
                    git config user.email "${USER}@gmail.com"
                    git clone https://${USER}:${PASS}@github.com/science-experiements-divya/alfredtest.git
                    cp -r alfredtest/Alfred ${env.WORKSPACE_DIR}
                    """
                } 
        }
      }
```


Notes:
- each sh is a new shell (You have to start the virtual environment each time)
- Venv is stored in main workspace as set by the global variable {WORKSPACE_DIR}


Resource: [Global environment](https://dzone.com/articles/jenkins-02-changing-home-directory)

![jenkins-test-4.PNG]({{site.baseurl}}/img/jenkins-test-4.PNG)

## Code explaination
We are using a declarative pipeline


### Environment
```
    environment{
     PROJECT_TITLE = 'science-experiments-divya'
     IS_PYTHON_EXIST = fileExists "/${env.WORKSPACE_DIR}/Python-3.7.7"
     IS_ENV_EXIST = fileExists "/${env.WORKSPACE_DIR}/.env"
     IS_ALFRED_EXIST = fileExists "/${env.WORKSPACE_DIR}/Alfred"

    }
```
- Here is where we define the global environment for this pipeline

The more important point is that we are using the fileExists boolean to check if a file exist, this will assist us in the checking if our python has been installed already.

`${env.WORKSPACE_DIR}` here refers to the path of our main workspace and is configured as a global variables.


### Installing python package

```
stage('Get python package'){
        when { expression { IS_PYTHON_EXIST == 'false' } }
        steps{
          dir("${env.WORKSPACE_DIR}"){
            sh"""
              ls
              apt-get -y update
              apt install -y build-essential wget zlib1g-dev libncurses5-dev libgdbm-dev libnss3-dev libssl-dev libreadline-dev libffi-dev
              wget https://www.python.org/ftp/python/3.7.7/Python-3.7.7.tgz
              tar -xf Python-3.7.7.tgz
              apt -y install python3-pip
              apt -y install python-pip
              cd Python-3.7.7
              ./configure --enable-optimizations
              make altinstall
              ls
              """
              // Start a new shell
              sh """
                  python3.7 -m pip install virtualenv
                  python3.7 -m venv env
              """
          }
        }
      }
```

- To install python, we will get the tar file from the python website
- Notice how we do it using `dir("${env.WORKSPACE_DIR}")`, this ensure that the library is install in the main workspace and not in our job's workspace. In the future, other jobs can access python as well due to this installation
- We also install virtualenv in order to run alfred in a virtual environment later on

### Virtual Environment
```
      stage("Attempt start at virtual environment"){
        when { expression { IS_ENV_EXIST == 'false' } }
        steps{
           dir("${env.WORKSPACE_DIR}"){
            sh """
                ls
                python3.7 -m pip install virtualenv
                python3.7 -m venv env
            """
           }
        }
      }
```
- Installs only virtual envionment and skips this step if it has already be installed

> The reason I put this code snippet in was because in case within the jenkins server, python has already been install but not virtual env, in that case, this stage will complete that requirement

### Alfred

Fetching:
```
      stage("Fetch Alfred"){
        when { expression { IS_ALFRED_EXIST == 'false' } }
        steps {
                withCredentials([usernamePassword(credentialsId: 'github-divya', usernameVariable: 'USER', passwordVariable: 'PASS')]) 
                {
                    sh """
                    cd ${env.WORKSPACE_DIR}
                    virtualenv .env
                    source .env/bin/activate
                    cd ${env.WORKSPACE}
                    rm -r alfredtest
                    git config user.name "${USER}"
                    git config user.email "${USER}@gmail.com"
                    git clone https://${USER}:${PASS}@github.com/science-experiements-divya/alfredtest.git
                    cp -r alfredtest/Alfred ${env.WORKSPACE_DIR}
                    """
                } 
        }
      }
```
- Skips this stage if Alfred already exist in the main workspace
- The alfred package is fetched from a private github repository with the help of credentials which can be set up under `manage Jenkins` configuration
- `cd ${env.WORKSPACE_DIR}` ensures that the directory is in the main working directory
	- Start the virtual environment
- `cd ${env.WORKSPACE}` will change the directory back to our job's directory
- Because we cannot do git commands in the main workspace, we did it in this job's directory
- After cloning the repo `alfredtest`, we copy the contents and place it in the main directory

> It is possible for this stage to be its own job since we are ensuring that Alfred is always installed

- However there might be a problem whereby an update to Alfred will not be detected 

Installing:
```
      stage("Installing Alfred"){
        steps{
          dir("${env.WORKSPACE_DIR}"){
          sh """
              virtualenv .env
              source .env/bin/activate
              cd ${env.WORKSPACE}
              cd deploy/scripts
              python3.7 -m pip install -r alfred-requirements.txt
          """
          // if possible, have alfred-requirements stored in alfred
          sh """
              ls
              cd Alfred
              python3.7 -m pip install --ignore-installed -e .
          """
        }
        }
      }
```
-  Installing the requirements for Alfred before installing Alfred itself

Running the script:
```
      stage('Running Alfred_script'){
          steps{
                dir("${env.WORKSPACE_DIR}"){
                  sh """
                  virtualenv .env
                  source .env/bin/activate
                  cd ${env.WORKSPACE}
                  cd deploy
                  python3.7 scripts/Alfred_script.py
                """
            }
          }
      }
```
- Start the env in the main workspace
- Navigate back to where our job workspace file is using `cd ${env.WORKSPACE}`
- go to deploy
- run Alfred_script

This would generate the 2 required deployment files



### Triggering another job
```
      stage("Attempt trigger other job"){
        steps {
          build job: 'microservice/testing-pipeline', propagate: true, wait: true
        }
      }
```

- This would run the other pipeline job I have set up titled 'testing-pipeline'

## Final file
```

def namespace = "default"
pipeline {

    environment{
     IS_PYTHON_EXIST = fileExists "/${env.WORKSPACE_DIR}/Python-3.7.7"
     IS_ENV_EXIST = fileExists "/${env.WORKSPACE_DIR}/.env"
     IS_ALFRED_EXIST = fileExists "/${env.WORKSPACE_DIR}/Alfred"
    }
agent any
  stages {
      stage('Get python package'){
        when { expression { IS_PYTHON_EXIST == 'false' } }
        steps{
          dir("${env.WORKSPACE_DIR}"){
            sh"""
              ls
              apt-get -y update
              apt install -y build-essential wget zlib1g-dev libncurses5-dev libgdbm-dev libnss3-dev libssl-dev libreadline-dev libffi-dev
              wget https://www.python.org/ftp/python/3.7.7/Python-3.7.7.tgz
              tar -xf Python-3.7.7.tgz
              apt -y install python3-pip
              apt -y install python-pip
              cd Python-3.7.7
              ./configure --enable-optimizations
              make altinstall
              ls
              """
              sh 'python3.7 --version'
              sh """
                  python3.7 -m pip install virtualenv
                  python3.7 -m venv env
              """
          }
        }
      }

      stage("Attempt start at virtual environment"){
        when { expression { IS_ENV_EXIST == 'false' } }
        steps{
           dir("${env.WORKSPACE_DIR}"){
            sh """
                ls
                python3.7 -m pip install virtualenv
                python3.7 -m venv env
            """
           }
        }
      }
      stage("Fetch Alfred"){
        when { expression { IS_ALFRED_EXIST == 'false' } }
        steps {
                withCredentials([usernamePassword(credentialsId: 'github-divya', usernameVariable: 'USER', passwordVariable: 'PASS')]) 
                {
                    sh """
                    cd ${env.WORKSPACE_DIR}
                    virtualenv .env
                    source .env/bin/activate
                    cd ${env.WORKSPACE}
                    rm -r alfredtest
                    git config user.name "${USER}"
                    git config user.email "${USER}@gmail.com"
                    git clone https://${USER}:${PASS}@github.com/science-experiements-divya/alfredtest.git
                    cp -r alfredtest/Alfred ${env.WORKSPACE_DIR}
                    """
                } 
        }
      }

      stage("Installing Alfred"){
        steps{
          dir("${env.WORKSPACE_DIR}"){
          sh """
              virtualenv .env
              source .env/bin/activate
              cd ${env.WORKSPACE}
              cd deploy/scripts
              python3.7 -m pip install -r alfred-requirements.txt
          """
          // if possible, have alfred-requirements stored in alfred
          sh """
              ls
              cd Alfred
              python3.7 -m pip install --ignore-installed -e .
          """
        }
        }
      }
      stage('Running Alfred_script'){
          steps{
                dir("${env.WORKSPACE_DIR}"){
                  sh """
                  virtualenv .env
                  source .env/bin/activate
                  cd ${env.WORKSPACE}
                  cd deploy
                  python3.7 scripts/Alfred_script.py
                """
            }
          }
      }

      stage("Attempt trigger other job"){
        steps {
          build job: 'microservice/testing-pipeline', propagate: true, wait: true
        }
      }

      stage('Create Branch & Push Branch') {
            steps {

                withCredentials([usernamePassword(credentialsId: 'github-divya', usernameVariable: 'USER', passwordVariable: 'PASS')]) 
                {
                    sh """
                    git config user.name "${USER}"
                    git config user.email "${USER}@gmail.com"
                    git checkout -b output/${APP_TAG}
                    git add .
                    git commit -m "update repo"
                    git push https://${USER}:${PASS}@github.com/science-experiements-divya/alfred-pipeline-test.git
                    """
                    
                } 
            }
      }

  }
}
```
