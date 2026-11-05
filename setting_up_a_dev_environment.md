# Setting Up an XNAT Development Environment Using Docker

This page covers the requirements and procedures for setting up a development environment for either the core XNAT application, subserved by the [xnat repository](https://github.com/NrgXnat/xnat), or any of various plug-ins, for example, the XNAT Container Service, which have their own individual repository.\* The purpose of this page is not to duplicate every bit of documentation found in the repos -- you're encouraged to read the documentation there. Rather, it's to provide a friendly, high-level orientation, especially since setting up a development environment requires understanding how a few repositories relate to each other.

If you're new to Java or web development you might come across some unfamiliar terms while gearing up to contribute to XNAT, in which case you can consult the [glossary](dev-glossary.md).

There are two potential approaches you can take to setting up your development environment. One is to install the various components of XNAT on your host machine. The other is to run XNAT in a Docker container. The latter approach is simpler, and that's what this page is going to focus on, but if you're interested in the former approach, you can find some of the steps to setting up an XNAT instance documented in the [CONTRIBUTING.md](https://github.com/NrgXnat/xnat/blob/develop/CONTRIBUTING.md) file in the xnat repository.

Basic requirements and their installation instructions:

- [Java 21](https://www.oracle.com/java/technologies/downloads/)
- [Docker](https://www.docker.com/products/docker-desktop/)
- [git](https://git-scm.com/install/)

The easiest way to get an XNAT instance running in Docker is with [xnat-docker-compose](https://github.com/NrgXnat/xnat-docker-compose). Using this repo gives you a dockerized application that runs on Linux, comprised of several containers, including Tomcat+XNAT, Postgres (the XNAT database), and nginx (a web proxy). Postgres and nginx come from public images for those applications. The Tomcat+XNAT, on the other hand, is built and released by XNAT and distributed through the NrgXnat Github (you can see the most recent [here](https://github.com/nrgxnat/xnat/pkgs/container/xnat)). Instructions on how to assemble these containers into a coordinated project are provided to Docker through a file called docker-compose.yml, and the commands to start (or stop) the processes in this project begin with `docker compose`. The definition of a container in a Docker compose file is a "service." In many cases you will also need an override file (docker-compose.override.yml), which contains your local changes to the Docker setup.

As always, you're encouraged to look at the xnat-docker-compose [README.md](https://github.com/NrgXnat/xnat-docker-compose/blob/master/README.md) for more detailed setup information.

1. First get the xnat-docker-compose source code. At the command line run:

```
git clone https://github.com/NrgXnat/xnat-docker-compose.git
```

Code snippets on this page are going to assume that all repositories are cloned in the same parent directory. Of course you don't have to do this, but if you don't, adjust any relative paths accordingly.

2. Next inside the xnat-docker-compose directory, copy the default settings to a real .env file:

```
cp default.env .env
```

You can change the default settings later, but it's not necessary for quickly standing up a development environment.

<details>
<summary>Some notes for Linux users</summary>

If you're on Linux and are running Docker without Docker Desktop, you have a couple of extra steps. (Everyone else can skip this section.) Get two values from the command line. The first is your uid, the result of running `id -u` at the command line. The second is your Docker group id, the result of running `stat -c %g /var/run/docker.sock` at the command line. Edit your .env file, and set `XNAT_UID` to your uid and `DOCKER_GID` to the Docker group id.

You also need to create several subdirectories. Run the following command from the xnat-docker-compose directory:

```
mkdir -p xnat-data/archive xnat-data/build xnat-data/cache xnat-data/prearchive xnat-data/home/logs xnat-data/home2/logs xnat/plugins
```

</details>

<details>
<summary>Some notes for Apple Silicon users</summary>

If you are using an Apple Silicon Mac, or any computer with ARM64 architecture, you need to create a file in your xnat-docker-compose directory called docker-compose.override.yml with the following lines:

```dockercompose
services:
  xnat-web:
    platform: linux/amd64
```

</details>

3. After having pulled the xnat-docker-compose code you can check that you can run the container. From the xnat-docker-compose directory, run

```
docker compose up --detach
```

You need to wait about a minute for the application to start up, but then you can go to <http://localhost> in a browser and log in to XNAT with the default credentials: admin/admin. The first time you log in, you'll get a page that asks you to choose your site settings; the defaults are fine.

To check the process, run

```
docker compose logs --follow xnat-web
```

and look for a line like `Server startup in [x] milliseconds`. Note that there's a similar line right at the beginning, `Server initialization in [x] milliseconds`, which does not indicate that your container is up and running. (Press `Ctrl-C` to exit the follow process.)

But since you're not planning on simply *using* the XNAT setup out of the box, but rather making changes to source code, you'll want to see your changes to source code reflected in your local running XNAT instance, which requires some changes to the setup. The changes you'll have to make to the default setup will vary with what kind of development you're doing. If you're planning on making changes to code in the core xnat repo, you'll still use the Tomcat/XNAT container distributed by XNAT, but you will mount your local copy of the XNAT web application onto it. If you're only planning on making changes to a plugin, you will need to make your plugin jar available to the XNAT webapp.

Now you come to a fork in the road: are you going to work on the core XNAT application or only a plug-in?

### I'm going to work on code in the main repository

In this case you will need to obtain the xnat source code, build it into a war file, unpack it into a subdirectory within the xnat-docker-compose directory, and instruct the Docker compose process to mount that subdirectory into the container.

1. Pull the xnat repo code. In your parent directory:

```
git clone https://github.com/NrgXnat/xnat.git
```

Here's a highly abbreviated version of what your file tree looks like now, assuming you cloned both repos into the same parent directory:

```
<your-parent-directory>/
├── xnat/
│   ├── build/
│   ├── xnat-web/
│   └──gradlew      
└── xnat-docker-compose/
    ├── xnat/
    │   ├── plugins/
    │   └── webapps/
    └── docker-compose.yml
```

2. Navigate to the top level of the xnat repository and build your war file:

```
./gradlew :xnat-web:war
```

Note: since xnat-web is only one of many modules in the xnat monorepo, it's easy to get confused and think that the above command only compiles and bundles the xnat-web module. However, it also compiles and bundles every module that xnat-web depends on, which is most of them.

3. Your war should have wound up at `<your-parent-directory>/xnat/xnat-web/build/libs/xnat-web-<version>.war`. Unpack it into the directory where Docker will look for it. Navigate to the xnat-docker-compose directory.

```
rm -rf xnat/webapps/ROOT 
mkdir -p xnat/webapps/ROOT 
unzip -q ../xnat/xnat-web/build/libs/xnat-web-<version>.war -d xnat/webapps/ROOT
```

It may seem a little non-intuitive to have first packed your code into an archive, only to unpack it, but the Gradle build accomplishes more than just archive building: it compiles your files and assembles them into the layout that Tomcat expects. The compiled classes from xnat/xnat-web go in `WEB-INF/classes`, the other modules' jar files and jar files for third party libraries go in `WEB-INF/lib`, etc. You can inspect this layout by navigating to `xnat-docker-compose/xnat/webapps/ROOT` if you're curious.

4. Instruct Docker to mount your web app files. Create a docker-compose override file: it should live next to `docker-compose.yml` in the top level of the xnat-docker-compose repository and be named `docker-compose.override.yml`.

Add these lines to that file:

```
services:
  xnat-web:
    volumes:
      - ./xnat/webapps/ROOT:/usr/local/tomcat/webapps/ROOT
```

(Or if you have an override file already because you have an ARM64 machine, simply add the lines

```
    volumes:
      - ./xnat/webapps/ROOT:/usr/local/tomcat/webapps/ROOT
```

under

```
services:
  xnat-web:
```

)

Finally you need to restart your container. Docker only reads your setup configuration on creating the container, and the XNAT web app only loads code into memory once, so any change to your code will require restarting the process for the result to take effect. This page uses the `force-recreate` flag to avoid any lingering effects of previous setups.

5. Run

```
docker compose up -d --force-recreate xnat-web
```

If you'd like to assure yourself that you have a working development environment and can see your work on your local XNAT, make a small symbolic change. In `<your-parent-directory>/xnat/xnat-web/src/main/webapp/xnat-templates/screens/Login.vm`, line 116 on the current develop branch, change the Login text in between the &lt;button&gt; tags, like so:
`<button class="submit btn" id="loginButton" type="submit" name="login">Kenny Loggins</button>`

Then repeat steps 2, 3, and 5, and after about a minute, check the login page at <http://localhost> for your change.

A final thing you may want to do to make sure you have a development environment set up is run the tests in the main xnat repo.

6. Navigate back to the main xnat repo and run

```
./gradlew build
```

That rebuilds the entire application, not just the xnat-web war, and runs tests. (The Docker daemon must be running for all the tests to pass.)

If your build successfully runs and tests pass, you have a working development environment.

### I'm going to work on plugin code outside of the main repository.

There are two main differences between working on code in the main XNAT app and working on code for a plugin. For development on plugin code, you need to build the jar for the plugin any time you've made a change that you want to see live, and make sure that jar is in the right place. You don't necessarily need to build the XNAT war or make a docker-compose.override file.

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
    └── gradlew    
```

Note: if you followed along with the first section of the page, you also have docker-compose.override.yml in your xnat-docker-compose directory. If you are not making changes to core XNAT code and don't need to rebuild, you may want to rename it docker-compose.override.yml.off (you can change it back later), because depending on your operating system it can slow down XNAT's startup process. If you created a docker-compose.override.yml because you have an Apple Silicon Mac and ARM64 architecture, however, you need your override file regardless; you can comment out the lines that direct Docker to your local war like this:

```
#  volumes:
#    - ./xnat/webapps/ROOT:/usr/local/tomcat/webapps/ROOT
```

that is, put `#` at the very beginning of those two lines.

2. Then navigate into your newly created container-service directory, use gradle to build the Container Service jar:

```
./gradlew fatJar
```

Your jar has landed in `<your-parent-directory>/container-service/build/libs/container-service-<version>-fat.jar`

3. Now navigate to the xnat-docker-compose directory and run

```
cp ../container-service/build/libs/container-service-<version>-fat.jar ./xnat/plugins/
```

4. Restart the Docker container. As mentioned in the core XNAT development section, the force-recreate flag here is used to ensure you're starting a container fresh. In this case it's necessary to incorporate the plugin to the running container.

```
docker compose up --detach --force-recreate
```

For the Container Service in particular, you can check whether you've successfully installed your plugin by navigating to Administer -&gt; Plugin Settings on the site. If you see the Container Service, you've succeeded. For other plugins that may not have an obvious web front end, you can log in to XNAT in your browser and navigate to <http://localhost/xapi/plugins> and check that your plugin is listed in the response.

If you are working on a plugin with a visible web frontend, just as with the main xnat web app, it's a useful test to check if you can make a change that is reflected in your local XNAT.

For the Container Service, for instance, you can open this file: `<your-parent-directory>/container-service/src/main/resources/META-INF/xnat/spawner/containers/site-settings.yaml` and change a visible label. If you change the line `label: Compute Backend `in the file (around line 304) to `label: Baby Got Backend`, repeat steps 2 and 3 above (starting from the parent directory where you cloned the xnat-docker-compose and plugin repos), and restart the running Docker container with `docker compose up --detach --force-recreate`, after about another minute and you should be able to see your change to the Container Service (or another plugin with a web front end).

5. You may also want to see if you can run your plugin's tests as a final success criterion for setting up a development environment. The plugins vary widely in their test coverage and requirements for running tests, so an overview is out of scope here: check the documentation in each repository for details. For the container service you can run: `./gradlew unitTest` from inside the container-service directory. Your test results will be in `container-service/build/reports/tests/unitTest/index.html`.

Note: getting the Container Service to actually launch containers requires some extra setup. See the xnat-docker-compose [README](https://github.com/NrgXnat/xnat-docker-compose#notes-on-using-the-container-service) for more information.

\* (There are other repositories, like xnat-rest-tests, that may have their own requirements; for those requirements and procedures you should see the repositories' own documentation on Github.)

### Troubleshooting

- You ran `docker compose up` and got an error that said "no matching manifest for linux/arm64."
  You may be on an Apple Silicon Mac or another computer with ARM64 architecture. You need to add lines to your docker compose override file to tell Docker to emulate AMD64 architecture (see above under "Notes for Apple Silicon users").

- Docker gives you multiple warnings of the form "X variable is not set. Defaulting to a blank string." and then exits with the error `invalid reference format`.
  You may have skipped the step where you copy the `default.env` file to .env (step 2 above, before the fork.)

- You successfully started your Docker container, but when you run `docker ps` xnat-web appears as "unhealthy."
  There could be multiple causes for this, but one possibility is that it's slow, especially if you're on an operating system other than Linux. Don't assume the process is broken.

- localhost doesn't load.
  If you are using your own XNAT war, did you accidentally skip the step where you unzipped the war file into the ROOT dir? Or did you unzip it into the wrong place?

- You ran `docker compose restart` or `docker compose up -d` and don't see your changes.
  Using `up` with the `--force-recreate` flag ensures that your edits are incorporated in the new running container.

- You get an error that says the Docker daemon isn't running.
  You need to start Docker. If you downloaded and installed Docker Desktop, just search for it on your computer and start it like you would any other application. Or if you're on Linux and not using Docker Desktop, `sudo service docker start`. (For other options for starting Docker, see [their documentation](https://docs.docker.com/engine/daemon/start/).)

- `go-yaml load error in parser (while parsing a block mapping)` 
  Something's likely wrong with your spacing in your docker-compose.override.yml file, or you're breaking another YAML rule.