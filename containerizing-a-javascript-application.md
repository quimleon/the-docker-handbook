# Containerizing a JavaScript Application

Now that you've got some idea of creating images, it's time to work with something a bit more relevant. In this sub-section, you'll be working with the source code of the [fhsinchy/hello-dock](https://hub.docker.com/r/fhsinchy/hello-dock) image that you worked with on a previous section. In the process of containerizing this very simple application, you'll be introduced to volumes and multi-staged builds, two of the most important concepts in Docker.

To begin with, open up the directory where you've cloned the repository that came with this article. Code for the `hello-dock` application resides inside the sub-directory with the same name.

```text
.
├── index.html
├── package.json
├── public
│   └── favicon.ico
└── src
    ├── App.vue
    ├── assets
    │   └── docker-handbook-github.webp
    ├── components
    │   └── HelloDock.vue
    ├── index.css
    └── main.js
```

This is a very simple JavaScript project powered by the [vitejs/vite](https://github.com/vitejs/vite) project. Don't worry though, you don't need to know JavaScript or vite in order to go through this sub-section. Having a basic understanding of [Node.js](https://nodejs.org/) and [npm](https://www.npmjs.com/) will suffice.

Just like any other project you've done in the previous sub-section, you'll begin by making a plan of how you want this application to run. In my opinion, the plan should be as follows:

* Get a good base image for running JavaScript applications i.e. [node](https://hub.docker.com/_/node).
* Set the default working directory inside the image.
* Copy the `package.json` file into the image.
* Install necessary dependencies.
* Copy rest of the project files.
* Start the vite development server by executing `npm run dev` command.

This plan should always come from the developer of the application that you're containerizing. If you're the developer yourself, then you should already have a proper understanding of how this application needs to be run. Now if you put the above mentioned plan inside `Dockerfile.dev`, the file should look like as follows:

```text
FROM node:lts

EXPOSE 3000

WORKDIR /app

COPY ./package.json .
RUN npm install

COPY . .

CMD [ "npm", "run", "dev" ]
```

Explanation for this code is as follows:

* The `FROM` instruction here sets the official Node.js image as the base giving you all the goodness of Node.js necessary to run any JavaScript application. The `lts` tag indicates that you want to use the long term support version of the image. Available tags and necessary documentation for an image is usually found on [node](https://hub.docker.com/_/node) hub page.
* Then the `WORKDIR` instruction sets the default working directory to `/app` directory. By default the working directory of any image is the root. You don't want any unnecessary files sprayed all over your root directory, do you? Hence you change the default working directory to something more sensible like `/app` or whatever you like. This working directory will be application to any consecutive `COPY`, `ADD`, `RUN` and `CMD` instructions.
* The `COPY` instruction here copies the `package.json` file which contains information regarding all the necessary dependencies for this application. The `RUN` instruction executes `npm install` command which is the default command for installing dependencies using a `package.json` file in Node.js projects. The `.` at the end represents the working directory.
* The second `COPY` instruction copies rest of the content from the current directory \(`.`\) of the host filesystem to the working directory \(`.`\) inside the image.
* Finally, the `CMD` instruction here sets the default command for this image which is `npm run dev` written in `exec` form.
* The vite development server by default runs on port `3000` hence adding an `EXPOSE` command seemed like a good idea, so there you go.

Now, to build an image from this `Dockerfile.dev` you can execute the following command:

```text
docker image build --file Dockerfile.dev --tag hello-dock:dev .

# Step 1/7 : FROM node:lts
#  ---> b90fa0d7cbd1
# Step 2/7 : EXPOSE 3000
#  ---> Running in 722d639badc7
# Removing intermediate container 722d639badc7
#  ---> e2a8aa88790e
# Step 3/7 : WORKDIR /app
#  ---> Running in 998e254b4d22
# Removing intermediate container 998e254b4d22
#  ---> 6bd4c42892a4
# Step 4/7 : COPY ./package.json .
#  ---> 24fc5164a1dc
# Step 5/7 : RUN npm install
#  ---> Running in 23b4de3f930b
### LONG INSTALLATION STUFF GOES HERE ###
# Removing intermediate container 23b4de3f930b
#  ---> c17ecb19a210
# Step 6/7 : COPY . .
#  ---> afb6d9a1bc76
# Step 7/7 : CMD [ "npm", "run", "dev" ]
#  ---> Running in a7ff529c28fe
# Removing intermediate container a7ff529c28fe
#  ---> 1792250adb79
# Successfully built 1792250adb79
# Successfully tagged hello-dock:dev
```

A container can be run using this image by executing the following command:

```text
docker container run --detach --publish 3000:3000 --name hello-dock-dev hello-dock:dev

# 21b9b1499d195d85e81f0e8bce08f43a64b63d589c5f15cbbd0b9c0cb07ae268
```

Now visit `http://127.0.0.1:3000` do see the `hello-dock` application in action.

![](.gitbook/assets/hello-dock-dev.png)

Congratulations on running your first real-world application inside a container. The code you've just written is okay but there is one big issue with it and a few places where it can be improved. Let's begin with the issue first.

## Working with Volumes

If you've worked with any front-end JavaScript framework before, you should know that the development servers in these frameworks usually come with a hot reload feature. That is if you make a change in your code, the server will reload automatically reflecting any changes you've made immediately.

But if you make any change in your code right now, you'll see nothing happening to your application running in the browser. The reason behind this is the fact that you're making changes in the code that you have in your local file system but the application you're seeing in the browser resides inside the container file system.

![](.gitbook/assets/local-vs-container-file-system.svg)

To solve this issue, Docker has something called [volumes](https://docs.docker.com/storage/volumes/). In the [Working With Executable Images](container-manipulation-basics.md#working-with-executable-images) sub-section under [Container Manipulation Basics](container-manipulation-basics.md) section, I said that volumes are used for granting your container access to your local file system.

What that means is, using volumes, you can easily mount one of your local file system directory inside a container. Instead of making a copy of the local file system inside the container, the volume can reference the local file system directly inside the container.

![](.gitbook/assets/bind-mounts.svg)

This way, any changes you make to your local source code will reflect immediately inside the container triggering the hot reload feature of vite development server. Changes made to the file system inside the container will reflect on your local file system as well and that how the [fhsinchy/rmbyext](https://hub.docker.com/r/fhsinchy/rmbyext) image deletes files from local file system.

A volume can be created using the `--volume` or `-v` option for the `container run` or `container start` commands. Kill and remove your previously started `hello-dock-dev` container, open up a terminal window inside the `hello-dock` project directory and start a new container by executing the following command:

```text
docker container run --detach --publish 3000:3000 --name hello-dock-dev --volume $(pwd):/app hello-dock:dev

# c07ee6ba946b0c6948c2f813290c8218023a01249c4fd008664964e88c646c1f
```

The `--volume` option can take three fields separated by colons \(`:`\). The generic syntax for the option is as follows:

```text
--volume <local file system directory absolute path>:<container file system directory absolute path>:<read write access>
```

The third field is optional but you must pass the absolute path of your local directory and the absolute path of the directory inside the container that will reference the local directory. In the command above `$(pwd)` will be replaced the the absolute path of your local directory which if you're following the article properly, should be the absolute path of your `hello-dock` project directory.

Although the usage of a volume solves the issue of hot reloads, it introduces another one. If you have any previous experience with Node.js, you may know that the dependencies of a Node.js project lives inside the `node_modules` directory on the project root.

Now that you're mounting the project root on your local file system as a volume inside the container, the content inside the container gets replaced along with the `node_modules` directory that contains all the dependencies.

This problem here can be solved using an anonymous volume. An anonymous volume is identical to a regular volume except the fact that you don't need to specify the source of the volume here. The generic syntax for creating an anonymous volume is as follows:

```text
--volume <container file system directory absolute path>:<read write access>
```

So the final command for starting the `hello-dock` container with both volumes should be as follows:

```text
docker container run --detach --publish 3000:3000 --name hello-dock-dev --volume $(pwd):/app --volume /app/node_modules hello-dock:dev

# 53d1cfdb3ef148eb6370e338749836160f75f076d0fbec3c2a9b059a8992de8b
```

Here, Docker will take the entire `node_modules` directory from inside the container and tuck it away in some other directory managed by the Docker daemon on your host file system and will mount that directory as `node_modules` inside the container.
