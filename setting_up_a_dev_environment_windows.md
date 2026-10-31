# Setting Up an XNAT Development Environment Using Docker

This page covers the requirements and procedures for setting up a development environment for either the core XNAT application, subserved by the [xnat repository](https://github.com/NrgXnat/xnat), or any of various plug-ins, for example, the XNAT Container Service, which have their own individual repository.\* The purpose of this page is not to duplicate every bit of documentation found in the repos -- you're encouraged to read the documentation there. Rather, it's to provide a friendly, high-level orientation, especially since setting up a development environment requires understanding how a few repositories relate to each other.

If you're new to Java or web development you might come across some unfamiliar terms while gearing up to contribute to XNAT, in which case you can consult the [glossary](dev-glossary.md).

There are two potential approaches you can take to setting up your development environment. One is to install the various components of XNAT on your host machine. The other is to run XNAT in a Docker container. The latter approach is simpler, and that's what this page is going to focus on, but if you're interested in the former approach, you can find some of the steps to setting up an XNAT instance documented in the [CONTRIBUTING.md](https://github.com/NrgXnat/xnat/blob/develop/CONTRIBUTING.md) file in the xnat repository.

Basic requirements and their installation instructions:

- [Java 21](https://www.oracle.com/java/technologies/downloads/)
- [Docker](https://www.docker.com/products/docker-desktop/)
- [git](https://git-scm.com/install/)

The easiest way to get an XNAT instance running in Docker is with [xnat-docker-compose](https://github.com/NrgXnat/xnat-docker-compose), which gives you a dockerized application comprised of several containers, including Tomcat+XNAT, Postgres (the XNAT database), and nginx (a web proxy). As always, you're encouraged to look at the xnat-docker-compose [README.md](https://github.com/NrgXnat/xnat-docker-compose/blob/master/README.md) for more detailed setup information.

First get the xnat-docker-compose source code. At the command line run:

```
git clone https://github.com/NrgXnat/xnat-docker-compose.git
```

Code snippets on this page are going to assume that all repositories are cloned in the same parent directory. Of course you don't have to do this, but if you don't, adjust any relative paths accordingly.

Now you come to a fork in the road: are you going to work on the core XNAT application or a plug-in?

![A fork in the road (from The Muppet Movie)](fork-in-the-road-upscaled-480.jpg)

### I'm going to work on code in the main repository

1. After having pulled the xnat-docker-compose code you can check that you can run the container.  From the xnat-docker-compose directory, run

```
docker compose up --detach
```

You need to wait about a minute for the application to start up, but then you can go to <http://localhost> in a browser and log in to XNAT with the default credentials: admin/admin.

If you get an error that suggests that the Docker daemon isn't running, that's because you need to start Docker. Assuming you downloaded and installed Docker desktop, just search for it on your computer and start it like you would any other application. (For other options for starting Docker, see [their documentation](https://docs.docker.com/engine/daemon/start/).)

xnat-docker-compose pulls an image with the XNAT application from the Github container repository. You can see the most recent [here](https://github.com/nrgxnat/xnat/pkgs/container/xnat).

But of course, if you're planning on working on the main XNAT application code you aren't going to want to use that image unmodified, but rather use the xnat-docker-compose infrastructure with your own XNAT code in development.

2. You also need to pull the xnat repo code. In your parent directory:

```
git clone https://github.com/NrgXnat/xnat.git
```

Here's a highly abbreviated version of what your file tree looks like now, assuming you cloned both repos into the same parent directory:

```
<your-parent-directory>/
├── xnat/
│   ├── build/
│   ├── xnat-web/
│   └──gradlew.bat      
├── xnat-docker-compose/
│   ├── xnat/
│   │   ├── plugins/
│   │   └── webapps/
│   └── docker-compose.yml
```

3. Navigate to the top level of the xnat repository and build your war file:

```
.\gradlew.bat :xnat-web:war
```

4. Your war should have wound up at `<your-parent-directory>\xnat\xnat-web\build\libs\xnat-web-<version>.war`. Unpack it into the directory where Docker will look for it.  Navigate to the xnat-docker-compose directory.

```
Remove-Item -Recurse -Force xnat\webapps\ROOT -ErrorAction SilentlyContinue
New-Item -ItemType Directory -Force xnat\webapps\ROOT
tar -xf ..\xnat\xnat-web\build\libs\xnat-web-<version>.war -C xnat\webapps\ROOT
```

5. Finally, you need to instruct Docker to look for your war file instead of pulling the remote image. Create a docker-compose override file: it should live next to `docker-compose.yml` in the top level of the xnat-docker-compose repository and be named `docker-compose.override.yml`.

Add these lines to that file:

```
services:
  xnat-web:
    volumes:
       - ./xnat/webapps/ROOT:/usr/local/tomcat/webapps/ROOT
```

Every time you make a change to the XNAT code and want to see your change in the running app, you need to return to steps 3 and 4 and build your war file and unpack it into the directory where Docker will look for it.

6. Finally run

```
docker compose up -d --force-recreate xnat-web
```

If you'd like to assure yourself that you have a working development environment and can see your work on your local XNAT, make a small symbolic change. In `<your-parent-directory>\xnat\xnat-web\src\main\webapp\xnat-templates\screens\Login.vm`, line 116 on the current develop branch, change the Login text in between the &lt;button&gt; tags, like so:
`<button class="submit btn" id="loginButton" type="submit" name="login">Kenny Loggins</button>`

Then repeat steps 3, 4, and 6, and after about a minute, check the login page at <http://localhost> for your change.

A final thing you may want to do to make sure you have a development environment set up is run the tests in the main xnat repo.

7. Navigate back to the main xnat repo and run

```
.\gradlew.bat build
```

That rebuilds the entire application, not just the xnat-web war, and runs tests. (The Docker daemon must be running for all the tests to pass.)

If your build successfully runs and tests pass, you have a working development environment.

### I'm going to work on plugin code outside of the main repository.

There are two main differences between working on code in the main XNAT app and working on code for a plugin. For development on plugin code, you don't need to build the XNAT war or make a docker-compose.override file. But you do need to build the jar for the plugin any time you've made a change that you want to see live, and make sure that jar is in the right place.

Let's take the Container Service as our example plugin. In order to work on Container Service development, of course you need the code for the Container Service, and in addition you need the running XNAT instance, just as you do for core XNAT development.

1. Next to the xnat-docker-compose repo, clone the Container Service repo (or the repo for whichever plugin you're planning on working on).

```
git clone https://github.com/NrgXnat/container-service.git
```
This is a highly abbreviated version of what your file tree looks like now:

```
<your-parent-directory>/
├── xnat-docker-compose/
│   ├── xnat/
│   │   ├── plugins/
│   │   └── webapps/
│   └── docker-compose.yml
└── container-service/
    ├── build/
    │  └──libs/
    ├── src/
    └── gradlew.bat    
```
2. Then navigate into your newly created container-service directory, use gradle to build the Container Service jar:

```
.\gradlew.bat fatJar
```

Your jar has landed in `<your-parent-directory>\container-service\build\libs\<container-service-version>-fat.jar`

3. Now navigate to the xnat-docker-compose directory and run

```
Copy-Item ..\container-service\build\libs\<container-service-version>-fat.jar .\xnat\plugins\
```

4. Now you can start the Docker container:

```
docker compose up --detach
```

For the Container Service in particular, you can check whether you've successfully installed your plugin by navigating to Administer -&gt; Plugin Settings on the site. If you see the Container Service, you've succeeded. For other plugins that may not have an obvious web front end, you can log in to XNAT in your browser and navigate to <http://localhost/xapi/plugins> and check that your plugin is listed in the response.

If you are working on a plugin with a visible web frontend, just as with the main xnat web app, it's a useful test to check if you can make a change that is reflected in your local XNAT.

For the Container Service, for instance, you can open this file: `<your-parent-directory>\container-service\src\main\resources\META-INF\xnat\spawner\containers\site-settings.yaml` and change a visible label. If you change the line `label: Compute Backend `in the file (around line 304) to `label: Baby Got Backend`, repeat steps 2 and 3 above (starting from the parent directory where you cloned the xnat-docker-compose and plugin repos), and restart the running Docker container with `docker compose restart xnat-web`, after about another minute and you should be able to see your change to the Container Service (or another plugin with a web front end).

5. You may also want to see if you can run your plugin's tests as a final success criterion for setting up a development environment. The plugins vary widely in their test coverage and requirements for running tests, so an overview is out of scope here: check the documentation in each repository for details. For the container service you can run: `.\gradlew.bat unitTest` from inside the container-service directory. Your test results will be in `container-service\build\reports\tests\unitTest\index.html`.

\* (There are other repositories, like xnat-rest-tests, that may have their own requirements; for those requirements and procedures you should see the repositories' own documentation on Github.)