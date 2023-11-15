**Lab 5 - Integration Server Setup**
Dependencies:
   - ansible 2.9.1
   - Python 3.10
   - docker 19.03
   - gitlab-ce 13.2
   - Gitlab-runner 14.3.2

     
**Step A: Create the Integration Server**
1. Navigate to the working directory:
   ```shell
   cd ~/devops-master/pipeline/s2-automate-build/integration-server
   ```

2. Create the missing data folder:
   ```shell
   mkdir ../../data
   
   # or 
   sudo rm -R ../../data && mkdir ../../data
   ```

3. Create the integration server using Vagrant:
   ```shell
   vagrant up
   ```
**note:** check Vagrantfile content and fix IP 192.168.33.9 >> 192.168.56.9

**Step B: Configure GitLab**
1. Connect to the integration server:
   ```shell
   vagrant ssh
   ```

2. Edit the GitLab configuration file:
   ```shell
   sudo nano /etc/gitlab/gitlab.rb
   ```

   - Replace `external_url http://hostname` with:
     ```
     external_url 'http://192.168.56.9/gitlab'
     ```

   - Replace `# unicorn['port'] = 8080` with:
     ```
     unicorn['port'] = 8088
     ```

3. Reconfigure and restart GitLab:
   ```shell
   sudo gitlab-ctl reconfigure
   sudo gitlab-ctl restart unicorn
   sudo gitlab-ctl restart
   ```

4. Access GitLab in your browser at http://192.168.56.9/gitlab/ and create a user with a password (minimum 8 characters). (12345678)

5. Create a second user in GitLab:
   - Visit http://192.168.56.9/gitlab/users/sign_in
   - Enter the following details:
     - Username: dev
     - Email: mld_dev@job.com
     - Password: 12345678

If you need to reset a GitLab user's password, follow the instructions [here](https://docs.gitlab.com/ee/security/reset_root_password.html).

6. Use the root account to approve user registrations:
   - Go to http://192.168.56.9/gitlab/admin/users?filter=blocked_pending_approval
   - Logout of gitlab but not vagrant

**Step C: Configure Docker**
1. Add the current user to the docker group to access the CLI:
   ```shell
   sudo usermod -aG docker vagrant
   ```

2. Validate the Docker installation with the hello-world example:
   ```shell
   docker run --name hello-world hello-world
   ```

   Expected output:
   ```
   Hello from Docker!
   ```

3. Remove the Docker container and image:
   - To remove the container & the image:
     ```shell
     docker rm hello-world
     docker rmi hello-world
     
     ```

**Step D: Use GitLab as VCS**
1. Exit the Vagrant session:
   ```shell
   exit
   ```

2. Change to the project directory:
   ```shell
   cd ~/devops-master/pipeline/s1-create-skeleton/MavenHelloWorldProject
   ```

3. Login to GitLab using the non-root account and create a project at http://192.168.56.9/gitlab/projects/new#blank_project.

4. Configure Git globally with your user name and email:
   ```shell
   git config --global user.name "dev"
   git config --global user.email "mld_dev@job.com"
   ```
5. Ensure the `pom.xml` file is in the root of the local repository.

6. Create a `.gitignore` file to exclude logs, dev settings, and binaries:
   ```shell
   nano .gitignore
   ```

   Add the following contents to the `.gitignore` file:

   ```
   # Binaries
   target/

   # Log files
   *Log

   # Dev settings
   .settings
   ```

7. Initialize the Git repository, add the remote origin, commit, and push the project:
   ```shell
   git init
   git remote add origin http://192.168.56.9/gitlab/dev/mavenhelloworldproject.git
   git add .
   git commit -m "Initial commit"
   git push -u origin master
   ```

**Step E: Automate Build**
1. Navigate to the integration server directory:
   ```shell
   cd ~/devops-master/pipeline/s2-automate-build/integration-server
   ```

2. Connect to the integration server:
   ```shell
   vagrant ssh
   ```

3. Install GitLab Runner:
   ```shell
   sudo curl -L --output /usr/local/bin/gitlab-runner https://gitlab-runner-downloads.s3.amazonaws.com/v14.3.2/binaries/gitlab-runner-linux-amd64
   sudo chmod +x /usr/local/bin/gitlab-runner
   sudo gitlab-runner install -u gitlab-runner
   gitlab-runner --version   
   ```

4. Get the runner token for the project at http://192.168.56.9/gitlab/dev/mavenhelloworldproject/-/settings/ci_cd.

5. Register the runner:
   ```shell
   sudo gitlab-runner register
   ```

   Follow the prompts and use the provided token. Specify the runner description, tags, executor, and Docker image.

   
   1. GitLab instance URL enter: 	http://192.168.56.9/gitlab/
   
   2. Enter the token generated 	use previously generated
   
   3. Enter for the description: 	docker
   
   4. Enter the gitlab-ci tag: 	integration
   
   5. Enter the executor: 		docker

   6. ﻿﻿﻿For the docker image enter: 	alpine:latest

   
7. Restart the runner:
   ```shell
   sudo gitlab-runner restart
   ```

8. Change the runner configuration to accept jobs without tags at http://192.168.56.9/gitlab/dev/mavenhelloworldproject/runners/1/edit.

**Step F: Create GitLab CI**
1. Exit the Vagrant session:
   ```shell
   exit
   ```

2. Change to the project directory:
   ```shell
   cd ~/devops-master/pipeline/s1-create-skeleton/MavenHelloWorldProject
   ```

3. Create a `.gitlab-ci.yml` configuration file:
   ```shell
   nano .gitlab-ci.yml
   ```

   Add the following contents to the `.gitlab-ci.yml` file:

   ```yaml
   image: maven:latest

   stages:
     - build
     - test
     - run

   cache:
     paths:
       - target/

   build_app:
     stage: build
     script:
       - mvn compile

   test_app:
     stage: test
     script:
       - mvn test

   run_app:
     stage: run
     script:
       - mvn package
       - mvn exec:java -Dexec.mainClass="com.jcg.maven.App"
   ```

4. Push the changes to GitLab:
   ```shell
   git add .
   git commit -m "Added GitLab CI"
   git push -u origin master
   ```

5. Inspect the CI/CD pipelines at http://192.168.56.9/gitlab/dev/mavenhelloworldproject/-/pipelines.


**Step G: Storing the "Binary"**

1. Modify the `.gitlab-ci.yml` file to include a deploy stage:
   ```shell
   nano .gitlab-ci.yml
   ```

   Add - deploy to "stages" as below, and add a "deploy_app" with stage with artifact storage:

   ```yaml
   stages:
     - build
     - test
     - run
     - deploy

   deploy_app:
     stage: deploy
     script:
       - echo "Deploy review app"
     artifacts:
       name: "my-app"
       paths:
         - target/*.jar
    ```


2. Push the changes to GitLab:
   ```shell
   git add .
   git commit -m "Added deploy step to GitLab pipeline"
   git push -u origin master
   ```


3. Inspect the CI/CD pipelines at http://192.168.56.9/gitlab/dev/mavenhelloworldproject/-/pipelines.

These organized and concise instructions guide you through setting up an automated build process using GitLab, Docker, and GitLab Runner.
