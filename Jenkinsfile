pipeline{
    agent any
    environment{
        IMAGE='flask-app:latest'
        EC2_IP='54.183.90.174    '
    }
    stages{
        stage('checkout'){
            steps{
                git url:"https://github.com/pavan931/ci-cd-jen-doc.git" , branch:'main',
                credentialsId:'git-creds'
            }
        }
        stage('Building the application'){
            steps{
                sh '''
                cd app
                docker build -t ${IMAGE} .
                '''
            }
        }
      //  To do keyless ssh into ec2 instance
      // Mount directly at docker run
      // -v /home/ubuntu/.ssh:/root/.ssh \
      //  or use the below 
         // Add SSH Credential to Jenkins
        // Go to Jenkins → Manage Jenkins → Credentials → (global)
       // Click "Add Credentials"
      // Choose "SSH Username with private key"
     // Username: ubuntu
     // Private Key: Enter directly (paste contents of ~/.ssh/ed_25519)
    // ID: ec2-ssh-key (you can name this anything)


        stage('Provision Remote with Docker + bzip2') {
  steps {
    withCredentials([sshUserPrivateKey(credentialsId: 'ec2-ssh', keyFileVariable: 'KEY')]) {
      sh '''
      cd ansible
      ansible-playbook -i hosts.ini setup.yml --private-key=$KEY
      '''
    }
  }
}
      // Here instead of pulling image from repository, i copied the image from control node to remote node using bzip2 , where i installed this manually in control node by going into jenkins container using
      // docker exec -it jenkins bash and installed it . so bzip2 compresses the image in control node and decompresses it in remote node so in remote node also you have to manually install this or add this inplaybook
      // in before stage setup.yml .
      // bzip2 runs on the control node (your Jenkins machine):
     // It compresses the Docker image.
     // bunzip2 runs on the remote EC2 via SSH:
     // It decompresses the image before piping into docker load.
     // Private key - when you generate ssh key-gen the private key should be put in 
      
         stage('Copy Image to EC2') {
    steps {
        withCredentials([sshUserPrivateKey(credentialsId: 'ec2-ssh', keyFileVariable: 'KEY')]) {
            sh '''
            docker save flask-app:latest | bzip2 | \
            ssh -o StrictHostKeyChecking=no -i $KEY ubuntu@${EC2_IP} 'bunzip2 | sudo docker load'
            '''
        }
    }
}

        stage('Run Application with Ansible') {
      steps {
        sh '''
        cd ansible
        ansible-playbook -i hosts.ini deploy.yml
        '''
      }
    }
    }
}
