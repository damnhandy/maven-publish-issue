# Maven Password Leaks

This project demonstrates a few ways Maven can leak password information.  The issues described here are not specific plugin, but only manifest themselves in certain conditions. The Maven plugins in question are:

   * [The Maven Release Plugin](https://maven.apache.org/maven-release/maven-release-plugin/)
   * [Maven Git Commit ID Plugin](https://github.com/ktoso/maven-git-commit-id-plugin) 
   * [Maven SCM Git Provider](https://maven.apache.org/scm/maven-scm-providers/maven-scm-providers-git/)

The combination of these plugins will expose your Git passwords when using Git over either HTTP or HTTPS when the Maven Release plugins `release:prepare` and `release:perform` plugins are invoked. Additionally if you're using the [Maven Git Commit ID Plugin](https://github.com/ktoso/maven-git-commit-id-plugin)  to capture commit informtaiton in your build, the generated `git.properties` will contain your user name and password and this file will be visible the Maven repository your application is published to. 

## Project Structure

Because these issues only manifest themselves at release time, I've created an environment with Docker compose that sets up the following:

* An instance of [Artifactory OSS](https://www.jfrog.com/open-source/) for our Maven repository
* An instance of [Gitbucket](https://github.com/gitbucket/gitbucket) for our git repo
* A user named [John Yaya](http://www.imdb.com/character/ch0113163/) who have a username of `jyaya` and a password of `password`
* A conatiner with Maven and Git which also mounts a sample Maven project to demo the issue.

## Running the environment

To run everything together,  you will need [Docker Compose](https://docs.docker.com/compose/) installed. To execute the environent, run:

	docker-compose run workspace /bin/bash

This will startup both Artifactory and Gitbucket and startup the "workspace" container and dump you into the container where you'll be able to execute the Maven and git commands. Once your in the containers shell, copy the `settings.xml` to the `/root/.m2` directory:

	root@4bd4bb1a3678:~# cp settings.xml .m2/

For some reason, files under hidden directories don't be copied into the hidden directory in the container. Next, checkout the project from Gitbucket:

	git clone http://gitbucket:8080/git/jyaya/maven-project.git

You should be able to build the project now:

	cd maven-project
	mvn clean install










