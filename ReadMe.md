### How to set up CI / CD Server 
##### By Yuval Gal


Based on : [Digital Ocean article ](https://www.digitalocean.com/community/tutorials/how-to-set-up-continuous-integration-pipelines-with-gitlab-ci-on-ubuntu-16-04)

*This written tutorial will try explain the matter of “How to set up continuous integration server”*
The final result of this tutorial will be something like that :

![CI Process, Yuval Gal](https://i.imgur.com/aNqH8lM.png)




#### Step 1 : Setting up the ci Server

The first thing anybody should do when starting with the ci process is to rent a server.
The server we will use in this example is of type Ubuntu 16.04
The minimum requirement is as such :
The Server should have at least 4GB of Ram.. 
The Server Should have at least 2 CPU Cores.
In this Example 30GB of DISK memory is allocated, i don't recommend less than that.
The Server type we are using is of type ec2, By Amazon AWS
Please allow outgoing and incoming traffic to tcp and udp port 80 at least.

#### Step 2 : Setting up a root user : 

This tutorial is not about linux in general but after having the access to the root user
Log in with the credentials you set, and add an new user with the command 
``` bash
adduser <username>
```
And then :
``` bash
usermod -aG sudo <username> 
```

This will grant you root privileges while not being in the root user.



#### Step 3 : Installing Docker on your machine  
Run the following commands in order to install docker on your machine 
If you want more details about what each command does, please refer to the digital ocean tutorial provided in the beginning .

``` bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
```

``` bash 
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
```

``` bash
 sudo apt-get update 
``` 
``` bash
 apt-cache policy docker-ce
```
``` bash
  sudo apt-get install -y docker-ce
```

Docker should now be installed, the daemon started, and the process enabled to start on boot. Check that it's running:
sudo systemctl status docker


The output should be : Active, and some details about the process.





####  Step 4 - Configure the docker we just installed :

Run the following commands : 
``` bash 
sudo usermod -aG docker ${USER}
```
``` bash
su - ${USER}
```
``` bash
<username> sudo docker
```
``` bash
sudo usermod -aG docker <username>
```


**Please replace the <username> with the username you set. E.g Yuval**



####  Step 5 - Running the docker container :

**A docker container is just a like another linux machine.
It can run linux command and use images from the docker hub, which can be found [here](https://hub.docker.com/)**

You can run a container by using the command “docker run -it <containerName>”
**Many more commands and usefull information can be found [here](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-ubuntu-16-04)**

#### Step 6 - Configure the Docker machine to serve apache 

Enough with the general information, let's start implement the right side of the diagram we draw at the begining.

**First thing first we will create a docker file in our home directory **
So we will  do the following commands : 

``` bash
cd /home/<username>
```
``` bash
mkdir dockerServer
```
And here we will create a new file named Dockerfile  (the name is important)

We can now configure the docker file as such :
Please write the same things at your docker file.

``` docker
FROM php:7.3-apache
COPY src/ /var/www/html
CMD apachectl -DFOREFROUND
EXPOSE 80
```


**the first line "from php:7.3-apache" is related to the docker hub
its using the php package, and there are many versions to choose from, in this can we are using the version 7.3 which also includes apache thats why its called php:7.3-apahce.
you can learn more about docker [here](https://www.youtube.com/watch?v=YFl2mCHdv24&)**




And then save the file.
These commands will copy an image from the docker hub
Copy the content of the src folder in the the dockerServer directory
Then it will run the apache service in the background and expose port 80 
To listen (it will listen to the host)

You can change the src folder to be something else as you need.


#### Step 7  - Running the newly created docker container : 

**Please run the following command which will listen to port 80 at the host and mount the /src/ folder to the /var/www/html folder **

``` bash
docker run -p 80:80 -v /home/<username>/dockerServer/src/:/var/www/html <ContaineName>
```
**Replace the <username> with your username 
And the <ContainerName> with the name of your choice .**

If you did everything right up until this point, the docker container is running great !
You can test it by placing index.php file inside the /src/ folder and access the public ip of the server.

Works ? great ! 
Let's continue...


#### Step 8 - Gitlab runners introduction 

This tutorial works with gitlab, and gitlab only.

GitLab CI Runners are the servers that check out the code and run automated tests to validate new changes. To isolate the testing environment, we will be running all of our automated tests within Docker containers.

In order to run a Gitlab runner, there is a need for a service, named the gitlab ci runner service

We install this service by doing the following commands :

``` bash
 curl -L https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh -o /tmp/gl-runner.deb.sh
```

``` bash
 less /tmp/gl-runner.deb.sh
```
``` bash
 sudo bash /tmp/gl-runner.deb.sh
```
``` bash
sudo apt-get install gitlab-runner
```

Now, after we installed the service we can go to the repo we want to enable the runner in, 
And enter the Settings Menu → CI /CD → Runners settings
There, there is section that looks [like this one](https://imgur.com/oDkrVYy) : 

Copy the registration token 
(you can also go into the Admin Area and enable the shared runner option)


Now, in the CI Server, run the following command : 
``` bash
sudo gitlab-runner register
```
**This will register an new runner, **
There will be some details there you will need to fill, such as the ** gitlab-ci coordinator **
For example :
**http://gitlab.<yourCompanyName>.com**
In the Gitlab-ci coordinator section 

**set the name and the other options required.
Set the executor to “docker”
Set the default docker image to “alpine:latest”
And at the end, when asked...put the token you copied earlier
You Can leave what wasn't mentioned here, blank.**


To View your Gitlab runners list  : 
Run the following command : 
``` bash
sudo gitlab-runner list
```
Is it showing up ? Great !

We Half way there …. 
Let's continue…




#### Step 9 - Setting up your env .gitlab-cli.yml configuration : 

Your apache server is now listening on port 80, and serving the files on your /src folder
But we do need to tell him, that for every push request he need to take the files we pushed and serve these.

This can be accomplished using the gitlab runner, the gitlab runner can get command which he will execute on every run, these command are written inside file named .gitlab-ci.yml

Detailed explanation about the gitlab-cli.yml can be found here 

We going to use the following script 
``` yml
image: node:10.15.1-jessie

before_script:
# cp - from local gitlab runner to the /src folder which the apache will listen to
# then at next stage run selenium test on global ip of the server.
# this will provide consistent uptime even when not testing and test env at the same time
# the only thing we need to set is the host file in apache server to prevent any(!!!) access to
# production servers any extra layer of security can be limit the ips which can access this server
# and the content which he serve.
- cd /builds/$GITLAB_USER_LOGIN/$CI_PROJECT_NAME
- pwd
- cp -R /builds/$GITLAB_USER_LOGIN/$CI_PROJECT_NAME /home/yuval/dockerPhpCiTest/src/

initalizingTest:
 script:
   - echo "Job completed, apache server is now serving the new files...";

build:
 script:
   - apt-get -y  update &&  apt-get -y upgrade
   - apt-get -y install default-jre
   - wget https://chromedriver.storage.googleapis.com/2.35/chromedriver_linux64.zip
   - unzip chromedriver_linux64.zip
   - apt-get -y --force-yes install make unzip g++ libssl-dev git xvfb x11-xkb-utils xfonts-100dpi xfonts-75dpi xfonts-scalable xfonts-cyrillic x11-apps clang libdbus-1-dev libgtk2.0-dev libnotify-dev libgnome-keyring-dev libgconf2-dev libasound2-dev libcap-dev libcups2-dev libxtst-dev libxss1 libnss3-dev gcc-multilib g++-multilib
   - npm i
   - pwd
   - ls -la
   -  echo "Job completed, env is ready for testing.";
   - xvfb-run node --harmony headlessTest.js
```



This script uses the variables provided by gitlab which you can [read about here](https://docs.gitlab.com/ee/ci/variables/)

This allows us to copy the content of our repo (that we pushed) inside the /src/ folder.
Please notice that the apache will serve the index.php inside /src/ so notice every subfolder inside and change the code accordingly..

If you will try to push not, the runner will fail, because we dont have test file yet named headlessTest.js, but it will create 2 stages named initalizingTest and build
Which will run one after the other, but only if they pass, meaning that the build stage won't run until the initalizingTest  stage will end without a fail.
Please notice that by default everything is removed between stages, unless its stored in cache or artifacts.

[Read More about it here](https://docs.gitlab.com/ee/ci/yaml/#artifacts)


We Almost done .. 

#### Step 10 - Create a puppeteer Test file.
While it's optional to use selenium webdriver, or nightmare in our example we decided to use puppeteer, which is tool developed by google and allows us to create async test files, which use js and node, we will create the headlessTest.js file in our project directory, and put there this test for example :

``` javascript
const puppeteer = require('puppeteer');
async function run() {
  // const browser = await puppeteer.launch({headless:false}); //-- for debuging !!!!! 
    const browser = await puppeteer.launch({args:['--no-sandbox','--disable-setuid-sandbox']});
    const page = await browser.newPage();

    await page.goto('http://www.google.com/');
    browser.close();
}

run();
```









This is a simple test that will go to the chosen website and then will close the browser in headless mode, this mode mean that there will be no ui involved in the process, the puppeteer will just download the dom and analyze it.

After saving the file. And running a commit,and push, you should be ready to go.. And the test you set will run on every push, and the repo will get update as well on the apache side.

**Please notice, exit code which is not 0 is a fail! exit code!!!!**


** So make sure to check the return code of the test when creating one **


Thats it :) 
