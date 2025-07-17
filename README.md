For the jenkins build  to work smooth in ec2 , never run them on root volumes
create new volume , mount it to ec2, create filesystem in directory and then use it
so while running a container give this path volume so that it plugs in the new volume you created not in the old one
-v /home/ubuntu/myvol/jenkins:/var/jenkins_home 

if in case you created a volume in root and want to copy that whole jenkins_home directory on host new mounted volume lets say - /home/ubuntu/myvolume then do this
use sudo -i  - you cant access /var/lib/docker/volumes/jenkins_home/_data as ubuntu user on host machine ec2
cd /var/lib/docker/volumes/jenkins_home/_data
rsync -avh /var/lib/docker/volumes/jenkins_home/_data/ /home/ubuntu/myvol/jenkins/
rsync actually copies with the same permissions

After copying, Jenkins container may not be able to access the files due to permission mismatch.
If Jenkins runs as user jenkins inside the container (usually UID 1000), set correct permissions:
chown -R 1000:1000 /home/ubuntu/myvol/jenkins

Jenkins thinks the controller (host EC2) still has low disk space, specifically in /tmp or the partition Jenkins uses for temporary files.
If your new volume (e.g., /home/ubuntu/myvol) has enough space, you can instruct Jenkins to use it for temporary files instead of /tmp.

use this in docker run <image>    --env JAVA_OPTS=-Djava.io.tmpdir=/var/jenkins_home/tmp

mkdir /home/ubuntu/myvol/jenkins/tmp
chmod -R 777 /home/ubuntu/myvol/jenkins/tmp

Example
docker run -d -u root \
  --name jenkins \
  -p 8000:8080 -p 50000:50000 \
  -v /var/run/docker.sock:/var/run/docker.sock \     - if this is not given then you cannot run docker commands in jenkins so this enables jenkins to access docker daemon on host and background process like  build  and run
  -v /home/ubuntu/myvol/jenkins:/var/jenkins_home \
  --env JAVA_OPTS=-Djava.io.tmpdir=/var/jenkins_home/tmp \   - this is for storing temporary files then container wont face disk space issues
  jenkins/jenkins:lts  - create own image that involves ansible,docker,if needed aws cli


********  Volume Mounting

-> sudo mkdir -p /mnt/newvol
Mount the volume to /mnt/newvol
Assuming your volume is /dev/xvdk (check with lsblk):
-> sudo mount /dev/xvdk /mnt/newvol
Make it persist after reboot
-> sudo nano /etc/fstab
it is good practice to offload heavy usage (like Jenkins builds, Docker images) to EBS to reduce root disk usage and improve performance.
-> sudo mkdir -p /mnt/newvol/docker
-> sudo nano /etc/docker/daemon.json
-> {
  "data-root": "/mnt/newvol/docker"
}
-> docker info | grep "Docker Root Dir"
Docker Root Dir: /mnt/newvol/docker

******* ssh into another ec2 instance using ansible playbook

Method 1: Directly mount the volume during docker run (But this is not safe cause if some has access to that ec2 then they might know the key)
-v /home/ubuntu/.ssh:/root/.ssh/

Method 2 : If you want to keep the key secure, use jenkins ui sshUserPrivateKey (Secured Method)
username:ubuntu
KEY:privatekey
use this in stage in pipeline using withCredentials([]){}
example:
   withCredentials([sshUserPrivateKey(credentialsId:'ec2-ssh',keyFileVariable: 'KEY')]){
        sh '''
        cd ansible
        ansible-playbook -i hosts.ini deploy.yml --private-key=$KEY -e "ansible_ssh_common_args='-o StrictHostKeyChecking=no'"  
        '''
          }

  
And if ansible is installed in jenkins container then its fine to run the playbook no need to be installed in the ec2 instance where jenkins container is running
