# Maven Password Leaks

This project demonstrates a few ways Maven can leak password information.  The issues described here are not specific plugin any single plugin (but it looks like the Maven Release Plugin is the core offender), but the issues only manifest themselves in certain conditions when the group of plugins interact with one another during the release process. The Maven plugins in question are:

   * [The Maven Release Plugin](https://maven.apache.org/maven-release/maven-release-plugin/)
   * [Maven Git Commit ID Plugin](https://github.com/ktoso/maven-git-commit-id-plugin) 
   * [Maven SCM Git Provider](https://maven.apache.org/scm/maven-scm-providers/maven-scm-providers-git/)

The combination of these plugins __will__ expose your Git passwords when using Git over either HTTP or HTTPS when the Maven Release plugins `release:prepare` and `release:perform` plugins are invoked. Additionally if you're using the [Maven Git Commit ID Plugin](https://github.com/ktoso/maven-git-commit-id-plugin)  to capture commit informtaiton in your build, the generated `git.properties` will contain your user name and password when using the default setting and this file __will__ be visible in the Maven repository your artifact is published to. 

## Project Structure

Because these issues only manifest themselves at release time, I've created an environment with Docker Compose that sets up the following:

* An instance of [Artifactory OSS](https://www.jfrog.com/open-source/) for our Maven repository
* An instance of [Gitbucket](https://github.com/gitbucket/gitbucket) for the git repo manager
* A user named [John Yaya](http://www.imdb.com/character/ch0113163/) who has a username of `jyaya` and a password of `password`
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

If build is successful, you can now test the issues.

## Running the release

The `settings.xml` file contains plain text passwords for both Artifactory and Gitbucket. They are crappy passwords and aren't the way you'd want to do things in a prodution environment. I did it this way to demo the bugs. To see what is going on, run the Maven Release plugin prepare goal by running:

	mvn release:prepare

Answer the typical Maven Release plugin questions:

	[INFO] Verifying that there are no local modifications...
	[INFO]   ignoring changes on: **/pom.xml.releaseBackup, **/pom.xml.next, **/pom.xml.tag, **/	pom.xml.branch, **/release.properties, **/pom.xml.backup
	[INFO] Executing: /bin/sh -c cd /root/maven-project && git status
	[INFO] Working directory: /root/maven-project
	[INFO] Checking dependencies and plugins for snapshots ...
	What is the release version for "mvn-release-test"? (com.damnhandy:mvn-release-test) 1.0.0: : 
	What is SCM release tag or label for "mvn-release-test"? (com.damnhandy:mvn-release-test) mvn-release-test-1.0.0: : 
	What is the new development version for "mvn-release-test"? (com.damnhandy:mvn-release-test) 1.0.1-SNAPSHOT: : 

Once you answer these, you'll see the build execute and you'll see this in at the end of the build output:

	[INFO] Checking in modified POMs...
	[INFO] Executing: /bin/sh -c cd /root/maven-project && git add -- pom.xml
	[INFO] Working directory: /root/maven-project
	[INFO] Executing: /bin/sh -c cd /root/maven-project && git status
	[INFO] Working directory: /root/maven-project
	[INFO] Tagging release with the label mvn-release-test-1.0.0...
	[INFO] Executing: /bin/sh -c cd /root/maven-project && git tag -F /tmp/maven-scm-489854065.commit mvn-release-test-1.0.0
	[INFO] Working directory: /root/maven-project
	[INFO] Executing: /bin/sh -c cd /root/maven-project && git push http://jyaya:password@gitbucket:8080/git/jyaya/maven-project.git mvn-release-test-1.0.0
	[INFO] Working directory: /root/maven-project
	[INFO] Executing: /bin/sh -c cd /root/maven-project && git ls-files
	[INFO] Working directory: /root/maven-project

Note the John Yaya's password is clearly visible where the Git URL is displayed. By default, Maven usese version 2.3.2 of the Maven Release plugin. This is fixed as of [2.4.2](https://issues.apache.org/jira/browse/MRELEASE-846), but you have to explicitly define the version of the Maven Release plugin to use:

				   <plugin>
                       <groupId>org.apache.maven.plugins</groupId>
                       <artifactId>maven-release-plugin</artifactId>
                       <version>2.5.3</version>
                   </plugin>

If you're also using the [Maven Git Commit ID Plugin](https://github.com/ktoso/maven-git-commit-id-plugin), you have another issue. Continue on with the release procecss and run:

	mvn release:perform
	
This will run now publish the release artifacts to Artifactory. Once complete, point your web browser on your host system to the Artifactory instance at either:

http://192.168.99.100:8081/artifactory/webapp/#/artifacts/browse/tree/General/libs-snapshot-local/com/damnhandy/mvn-release-test/1.0.0-SNAPSHOT/mvn-release-test-1.0.0-20160325.134639-1.jar

or
 
http://localhost:8081/artifactory/webapp/#/artifacts/browse/tree/General/libs-snapshot-local/com/damnhandy/mvn-release-test/1.0.0-SNAPSHOT/mvn-release-test-1.0.0-20160325.134639-1.jar

click the arrow next to the JAR icon to expand it, click on the `git.properties` file and then select the "view source" tab in the right-hand panel. You should now see that the Git password is still present in the `git.remote.origin.url` property:

![Screenshot](images/screenshot.png)

## Ways to avoid these issues

So this issue kind of sucks, especially if you're working on public projects and using these plugins with HTTP/HTTPS. Basically, your passwords are probably in the CI logs for any Jenkins, TravisCI, or other CI systems logs.  Additionally the artifacts might also contain `git.properties` file with your using name and password in it. There's a few ways to avoid these issues such as:

* Use version 2.4.3 or higher of the Maven release plugin
* If you're using the Git Commit ID Plugin, exclude the `git.remote.origin.url` property from your build
* Use SSH for your CI builds 







