# CI Using Jenkins to build docker image and Push it to hub.docker.com

1. Install Jenkins Machine in VM
    1. Using Vagrantfile create a Ubuntu machine and install Jenkins with required packages 
        ```vagrantfile
        Vagrant.configure("2") do |config|
          # Master node
          config.vm.define "jenkins-machine" do |master|
            master.vm.box = "ubuntu/focal64"
            master.vm.network "private_network", type: "static", ip: "192.168.50.40"
            master.ssh.insert_key = false
            master.vm.provider "virtualbox" do |v|
              v.memory = 2048
            end
            master.vm.provision "shell", inline: <<-SHELL
              sudo apt-get update
              sudo apt-get install -y python unzip net-tools
              sudo echo jenkins-machine > /etc/hostname
              sudo hostnamectl set-hostname jenkins-machine

              # Install Packer
              wget https://releases.hashicorp.com/packer/1.7.4/packer_1.7.4_linux_amd64.zip
              unzip packer_1.7.4_linux_amd64.zip
              sudo mv packer /usr/local/bin/
              rm packer_1.7.4_linux_amd64.zip

              # Install daemon (required dependency for Jenkins)
              sudo apt-get install openjdk-11-jre -y
              curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo tee  /usr/share/keyrings/jenkins-keyring.asc > /dev/null
              sudo echo "deb [trusted=yes] https://pkg.jenkins.io/debian binary/" > /etc/apt/sources.list.d/jenkins.list
              sudo apt-get update
              sudo apt-get install jenkins -y

              sudo systemctl enable --now jenkins
              sudo apt-get install git

              # Install Maven
              # sudo apt-get install maven -y

              # Install Docker
              sudo apt-get install -y docker.io
              sudo usermod -aG docker $USER

              # Open firewall for Jenkins (port 8080)
              sudo ufw allow 8080
              sudo ufw enable

              # Fixing name server issue
              sudo mkdir -p /etc/systemd/resolved.conf.d/
              sudo echo "[Resolve]" > /etc/systemd/resolved.conf.d/custom.conf
              sudo echo "DNS=8.8.8.8" >> /etc/systemd/resolved.conf.d/custom.conf
              sudo echo "DNS=8.8.4.4" >> /etc/systemd/resolved.conf.d/custom.conf
              sudo systemctl restart systemd-resolved

            SHELL
          end
        end
        ```
        Once running you can ssh it using `vagrant ssh ansible-machine`
        
2. Here i am using this private github.com repo `https://github.com/roychandrasekhar/flask-welcome` for this demo. Its a flask API running in port 5000. Also it has Dockerfile in it so that it can be run as micro service. *Docker file is provided here as a copy*
3. I am using my public `hub.docker.com` account to store the Docker image. `https://hub.docker.com/repository/docker/roychandrasekhar/flask-welcome/general`
4.  Ok, now in your VM start Jenkins and configure it. It should be in `http://192.168.50.40:8080`
5. For secure access of your github private account i use SSH key base authentication method. And for this i update the `jenkins` service user in the server and create `id_rsa.pub` and add this in my github account. So now the Jenkins can pull the repo.
    
    
    
6. Now create 1 credential in Jenkins for dockerhub
    ![](https://i.imgur.com/v4z4rRI.png)
    
    

7. Now add a pipeline project with the given `gradle.txt` file 
    ```gradle
    pipeline {
        agent any

        environment {
            REPO_URL = 'git@github.com:roychandrasekhar/flask-welcome.git'
            DOCKER_IMAGE_NAME = 'flask-welcome:latest'
            DOCKER_HUB_REPO = 'roychandrasekhar/flask-welcome'
            DOCKER_HUB_CREDENTIAL_ID = 'chandradockerhub'
        }

        stages {
            stage('Checkout') {
                steps {
                    script {
                        // Clean the existing directory
                        sh 'rm -rf flask-welcome'

                        // Use the existing SSH key from the Jenkins host machine
                        sh 'ssh-agent bash -c "ssh-add /home/jenkins/.ssh/id_rsa; git clone ${REPO_URL}"'
                    }
                }
            }

            stage('Push Docker Image to Docker Hub') {
                steps {
                    script {
                        // Retrieve Docker Hub credentials from Jenkins credentials
                        withCredentials([usernamePassword(credentialsId: DOCKER_HUB_CREDENTIAL_ID, usernameVariable: 'DOCKERHUB_USERNAME', passwordVariable: 'DOCKERHUB_PASSWORD')]) {
                            // Log in to Docker Hub using the retrieved credentials
                            sh "docker login -u $DOCKERHUB_USERNAME -p $DOCKERHUB_PASSWORD"

                            // Tag the Docker image with the Docker Hub repository
                            sh "docker tag $DOCKER_IMAGE_NAME $DOCKER_HUB_REPO"

                            // Push the Docker image to Docker Hub
                            sh "docker push $DOCKER_HUB_REPO"
                        }

                    }
                }
            }

            // Add more stages as needed, such as testing, deployment, etc.
        }

        post {
            always {
                script {
                    // Display a message in the Jenkins console log
                    echo 'Pipeline completed successfully!'
                }
            }
        }
    }

    ```

    ** add this in Pipeline Definition as Pipeline script.
    
    
8. Update your parameter as per your environment.
    ![](https://i.imgur.com/UIgA0ER.png)

9. This is the final run screenshot
    ![](https://i.imgur.com/52Z40EO.png)
    
    ![](https://i.imgur.com/YeA2ZuW.png)

