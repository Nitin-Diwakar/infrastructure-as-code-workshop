# Lab 03 - Run a Docker Container

In this lab, we'll run a Docker container we build locally:

## Step 1 - Create a new project

As before, create a new project in an empty directory and call it `pulumi-docker`

```bash
mkdir pulumi-docker
cd pulumi-docker
pulumi new typescript
```

## Step 2 - Create your application

Now, let's make a very simple HTTP application with typescript. Inside your project directory, create an application directory:

```bash
mkdir app
```

Inside this `app` directory should be two files. We need to bootstrap a webserver application. We'll use [express.js](https://expressjs.com/) for this.

First, let's get all the dependencies we need:

```bash
# create a npm package
npm init --yes
# install typescript
npm install typescript
# install expressjs
npm install express @types/express morgan @types/morgan
```

Now, let's define our express.js webserver. In a file called `index.ts`, let's add the following:

```typescript
import express = require('express');
import morgan = require('morgan');

const app: express.Application = express();

// defines a logger for output
app.use(morgan('combined'))

app.get('/', function(req, res) {
    res.send("Hello world!");
});

app.listen(3000, function() {
    console.log('Starting app on port 3000!');
})
```

Next, create a `Dockerfile` which will be built and will include this webserver

```
FROM node:12

WORKDIR /app

COPY *.json /app/
COPY index.ts /app/

RUN npm install && npm run env -- tsc index.ts
# use dumb-init so docker containers respect signals
RUN wget -O /usr/local/bin/dumb-init https://github.com/Yelp/dumb-init/releases/download/v1.2.2/dumb-init_1.2.2_amd64 && chmod +x /usr/local/bin/dumb-init

EXPOSE 3000

ENTRYPOINT ["/usr/local/bin/dumb-init", "--"]
CMD [ "node", "index.js" ]
```

## Step 3 - Build your Docker Image with Pulumi

Back inside your pulumi program, let's build your Docker image. Inside your `index.ts` add the following:


```typescript                                                                                                                                                                                                        0.0s
import * as pulumi from "@pulumi/pulumi";
import * as docker from "@pulumi/docker";

const config = new pulumi.Config();
const stack = pulumi.getStack();

const imageName = config.require('image_name');

const image = new docker.Image('local-image', {
    build: './app',
    imageName: `${imageName}:${stack}`,
    skipPush: true,
})
```

Make sure you install the `@pulumi/docker` provider:

```
npm install @pulumi/docker
```

You should see some output showing the npm package and the provider being installed

Run `pulumi up` and it should build your docker image

If you run `docker images` you should see your built container.

## Step 4 - Run the container

Finally, let's run your container. Update your pulumi program to add the following:

```typescript
import * as pulumi from "@pulumi/pulumi";
import * as docker from "@pulumi/docker";

const config = new pulumi.Config();
const stack = pulumi.getStack();

const imageName = config.require('image_name');

const image = new docker.Image('local-image', {
    build: './app',
    imageName: `${imageName}:${stack}`,
    skipPush: true,
})

const container = new docker.Container('local-container', {
    image: image.baseImageName,
    ports: [{
        internal: 3000,
        external: 3000,
    }]
})
```

Re-run your Pulumi program and your container should launch. You can verify this by looking at the docker stats for the running image:

```bash
docker stats $(pulumi stack output container_id) --no-stream
CONTAINER ID   NAME                       CPU %     MEM USAGE / LIMIT     MEM %     NET I/O     BLOCK I/O   PIDS
4b43bf4c92ab   my-running-image-39e5943   0.03%     10.05MiB / 1.941GiB   0.51%     946B / 0B   0B / 0B     1
```


