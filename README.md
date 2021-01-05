## Docker build utility

This utility helps you with building your project with Docker if your project is not shipped as 
a container (such as frontend web projects or native apps). In order to use it, your Dockerfile 
must be specifically structured. This structure comes from our long-term experience 
with Docker - it utilizes the Docker layer cache which reduces build time and resolves file 
permission issues when building on Linux. 

The script is compatible with Python >= 2.7 and Python >= 3.5. It was not tested on other versions.

## Installation
There are two basic ways how to install and run this script. Your choices are:

**1. Copy the script into your project (either commit it or download it before each build)**  
`wget -q -O docker-build.py https://raw.githubusercontent.com/zaslat/docker-build/master/docker-build.py`  
*Note: to execute the script, call `python docker-build.py ...[arguments]...`*

**2. Install the script into your (Linux) system**  
`sudo wget -q -O /usr/local/bin/docker-build https://raw.githubusercontent.com/zaslat/docker-build/master/docker-build.py`  
`sudo chmod +x /usr/local/bin/docker-build`  
*Note: to execute the script, call `docker-build ...[arguments]...`*

## How to structure your Dockerfile
These are rules that your Dockerfile should follow in order to get maximal efficiency of using this tool.

**1. Create new Dockerfile (Build.Dockerfile)**  
If you use Docker (or Docker Compose) for running your application, you might want to create
a new Dockerfile (e.g. `Build.Dockerfile`) which will be used for building. This way you have 
maximal control over the structure of the Dockerfile, and you won't break things if you make changes to it.

**2. Add .dockerignore file**  
Each time you execute `docker build` command, Docker sends all project files to its context directory and performs
the build. The build time may get slower when there are huge files in your project, so it's highly recommended
to create a `.dockerignore` file in the root of your project to specify which files are not sent to the context. 

Add `node_modules` if you use NPM or Yarn, `vendor` if using Composer, etc. These files will not be used for the build.

Example:
```
.idea
.vscode
.git

.dockerignore
.gitignore
Build.Dockerfile
```

**3. Set WORKDIR**  
Setting `WORKDIR` in your Dockerfile simplifies all operations with files, you don't need to 
type the absolute path for each command or change the working directory during 
`RUN` and `CMD` commands.

Simply add `WORKDIR <your-absolute-path>` right after the `FROM` command at the beginning of the Dockerfile.

If you are unsure about which path to chose, we recommend `/usr/src/app`. This path is commonly used in applications
far beyond the Docker environment.

Example: `WORKDIR /usr/src/app`

**4. Install system dependencies first**  
If you need any system dependencies for the build (e.g. `wget`, `unzip`, etc.), install them right
after the `WORKDIR` command, ideally using a single `RUN` command. Split them into multiple `RUN` statements
only if some packages take very long time to install. In such case install the long-running first.

The reason for this is that if you later add one new system dependency into the list, you won't need to
wait for the long-running dependencies to install. The Docker will simply rebuild only the layer that changed.

Don't forget to update your package cache (e.g. `apt-get update`) in each layer. Otherwise, you might end up
installing obsolete or non-existing packages if you later (after few weeks/months) change only one layer of 
system dependencies. 

Example:
```
# Install Java - this may take very long time
RUN apt-get update && apt-get install -y java

# Install other packages
RUN apt-get update && apt-get install -y wget unzip curl nano
```

**5. Install project dependencies**  
If your project uses external libraries (via NPM, Composer, Maven, etc.), it's recommended to 
install them right after the system dependencies. 

*This is the most critical part that might bring you the highest reduction of build time.*

First, copy the **minimal amount of files** you need to install your packages:

|Package manager|Command|
|---------------|-------|
|NPM            |`COPY package.json package-lock.json ./`|
|Yarn           |`COPY package.json yarn-lock.json ./`   |
|Composer       |`COPY composer.json composer.lock ./`   |
|Maven          |`COPY pom.xml ./`                       |

Second, install the packages (and clear the package cache):

|Package manager|Command|
|---------------|-------|
|NPM            |`RUN npm install && npm cache clean --force`  |
|Yarn           |`RUN yarn install && yarn cache clean`        |
|Composer       |`RUN composer install && composer clear-cache`|
|Maven          |`RUN mvn install`                             |

Example:
```
COPY package.json package-lock.json ./
RUN npm install && \
    npm cache clean --force
```

Installing packages this way enables Docker to cache them and reinstall them only if the specification
(e.g. `package.json`) changes.


**6. Copy project files**  
First, make sure you specified your package manager folder (e.g. `node_modules`) in `.dockerignore`. Otherwise, 
you might end up with your local package manager files overwriting the ones in Docker image. 

Then simply copy all files (that are present in build context) into the current directory.

Example: `COPY . .`

Remember that files listed in `.dockerignore` are not copied.

**7. Execute build with CMD**  
The build itself must be performed with `CMD` command. This is because you don't want the build itself
to be cached with Docker in order to be able to re-build your app easily.

Example: `CMD npm run build`

**8. Use multi-stage Dockerfile if possible**   
If you are familiar with multi-stage Dockerfiles, feel free to use it!

## Example
See the [JavaScript example](examples/javascript) in examples directory.

To build it, execute this from root of this repository:  
`python3 docker-build.py --workdir examples/javascript --dockerfile Build.Dockerfile --dist-dir dist`

The build is stored into `dist` folder.

The script expects that:
1. The project resides in `examples/javascript` directory (`--workdir examples/javascript`)
1. The Dockerfile is named `Build.Dockerfile` and it sits in the root of your project (`--dockerfile Build.Dockerfile`)
3. Build artifacts (output of the build) are saved into `dist` folder (`--dist-dir dist`)
4. Dockerfile follows the practices listed above

## Script arguments
The script performs 4 Docker operations - builds the image (`docker build`), creates and runs the container
(`docker run`), copies build artifacts (`docker cp`) and removes the container (`docker rm`). You can freely control
all of these operations by adding arguments to the script. These arguments must be enclosed in quotes
and must be set with the equality operator, e.g. `--docker="--my-arg=my-arg-value"`. Multiple
arguments of the same type can be specified.

**Docker arguments**  
To add an argument to all Docker commands (argument placed before the operation specifier - 
e.g. `docker <arg-here> build`), specify it via `--docker` argument. 

Example: `python docker-build.py --docker="--host=1.2.3.4"`  
Result: `docker --host 1.2.3.4 build`

**Docker build arguments**  
To add an argument to Docker build command (argument placed after the operation specifier - 
e.g. `docker build <arg-here>`), specify it via `--docker-build` argument. 

Example: `python docker-build.py --docker-build="--build-arg=MY_ARG=MY_VALUE"`    
Result: `docker build --build-arg MY_ARG=MY_VALUE`

**Docker run arguments**  
To add an argument to Docker run command (argument placed after the operation specifier - 
e.g. `docker run <arg-here>`), specify it via `--docker-run` argument. 

Example: `python docker-build.py --docker-run="--env=MY_ARG=MY_VALUE"`    
Result: `docker run --env MY_ARG=MY_VALUE`

**Docker cp arguments**  
To add an argument to Docker copy command (argument placed after the operation specifier - 
e.g. `docker cp <arg-here>`), specify it via `--docker-cp` argument. 

Example: `python docker-build.py --docker-cp="--archive"`    
Result: `docker cp --archive`

Script arguments:

```
usage: docker-build.py [-h] [--version] --dist-dir DIST_DIR [--out-dir OUT_DIR] [--workdir WORKDIR] [--dockerfile DOCKERFILE] [--docker-context DOCKER_CONTEXT] [--docker DOCKER_ARGS] [--docker-build DOCKER_BUILD_ARGS]
                       [--docker-run DOCKER_RUN_ARGS] [--docker-cp DOCKER_CP_ARGS]

Build project with Dockerfile.

optional arguments:
  -h, --help            show this help message and exit
  --version             show program's version number and exit
  --dist-dir DIST_DIR   Docker directory which contains build artifacts
  --out-dir OUT_DIR     Output directory into which copy build artifacts, relative to current working directory (or --workdir if set), defaults to --dist-dir
  --workdir WORKDIR     working directory where to execute scripts
  --dockerfile DOCKERFILE
                        path to the Dockerfile relative to current working directory (or --workdir if set)
  --docker-context DOCKER_CONTEXT
                        context of docker build command relative to current working directory (or --workdir if set)
  --docker DOCKER_ARGS  any argument passed to docker calls, e.g. --docker="--host=127.0.0.1"
  --docker-build DOCKER_BUILD_ARGS
                        any argument passed to docker build call, e.g. --docker-build="--no-cache"
  --docker-run DOCKER_RUN_ARGS
                        any argument passed to docker run call, e.g. --docker-run="--rm"
  --docker-cp DOCKER_CP_ARGS
                        any argument passed to docker cp call, e.g. --docker-cp="--archive"
```

## License
This project is dual-licensed under [UNLICENSE](LICENSE-2) and [MIT license](LICENSE). 
