pipeline {
  agent any

  environment {
    APP_HOST = "<APP_PUBLIC_IP>"   // set here or in Jenkinsfile as parameter
    SSH_CRED_ID = 'app-ssh-key'    // credential id in Jenkins (SSH Username with private key)
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Build') {
      steps {
        sh 'mvn -B -DskipTests=false clean package'
        // artifact will be in target/*.war
      }
    }

    stage('Prepare Ansible') {
      steps {
        // Ensure the inventory is rendered with the real IP
        writeFile file: 'ansible/inventory.ini', text: """
[app]
app-node ansible_host=${env.APP_HOST} ansible_user=ubuntu ansible_private_key_file=/var/lib/jenkins/.ssh/id_rsa
"""
        sh 'ls -la ansible'
      }
    }

    stage('Inject SSH Key to Jenkins (on master)') {
      steps {
        // This step requires the private key stored in Jenkins as 'app-ssh-key' (SSH Username with private key)
        // We will write the private key to /var/lib/jenkins/.ssh/id_rsa so ansible can use it.
        // Use withCredentials to get the private key
        withCredentials([sshUserPrivateKey(credentialsId: "${SSH_CRED_ID}", keyFileVariable: 'KEYFILE', usernameVariable: 'SSHUSER')]) {
          sh '''
            sudo mkdir -p /var/lib/jenkins/.ssh
            sudo cp $KEYFILE /var/lib/jenkins/.ssh/id_rsa
            sudo chown jenkins:jenkins /var/lib/jenkins/.ssh/id_rsa
            sudo chmod 600 /var/lib/jenkins/.ssh/id_rsa
          '''
        }
      }
    }

    stage('Run Ansible — Install Tomcat & Deploy WAR') {
      steps {
        sh '''
          # run ansible from workspace
          ansible-playbook -i ansible/inventory.ini ansible/playbook-tomcat.yml --ssh-extra-args="-o StrictHostKeyChecking=no -i /var/lib/jenkins/.ssh/id_rsa"
        '''
      }
    }
  }

  post {
    success {
      echo "Build & deploy complete. App should be available at http://${env.APP_HOST}:8080/"
    }
    failure {
      echo "Pipeline failed."
    }
  }
}
