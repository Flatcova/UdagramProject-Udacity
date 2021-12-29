[//]: # (Image Reference)
[image1]: ./img/s3bucket.jpeg "S3 bucket running"
[image2]: ./img/rds.jpeg "RDS postgres database"
[image3]: ./img/eb.jpeg "Elastic Beanstalk server Health and running"
[image4]: ./img/circleci.jpeg "CircleCi project pipeline"

[image5]: ./img/IAM.jpeg "IAM user created"
[image6]: ./img/IAM-credentials.jpeg "User credentials"
[image7]: ./img/s3bucket-frontend.jpeg "S3 public for the frontend"
[image8]: ./img/circleci-complete.jpeg "CircleCi complete run"
[image7]: ./img/s3bucket-images.jpeg "S3 for images"

# Udagram

This application is provided as an alternative starter project. The udagram application is a fairly simple application that includes all the major components of a Full-Stack web application.

The purpose of this project it's to create all the configurations and services on AWS so it would run, creating simple scripts for each section (frontend and backend), and at the end create an automated pipeline with CircleCI. I'll be explaining all the process that was made with images, and also the few changes made on the startup code.

## Getting Started with Code

I was provided with the original repo that you can also download and follow the steps provided: [udagram-starter-project](https://github.com/udacity/nd0067-c4-deployment-process-project-starter)

once the repo it's cloned and installed all the dependencies, i procced to create the ``.env`` file inside the udagram-api folder with the following structure.

```
POSTGRES_USERNAME=postgres
POSTGRES_PASSWORD=postgres
POSTGRES_DB=postgres
DATABASE_PORT=5432
POSTGRES_HOST=XXXX
AWS_REGION=us-east-1
AWS_BUCKET=api-udagram
JWT_SECRET=XXXX
AWS_ACCESS_KEY=XXXX
AWS_SECRET=XXXX
```

## AWS S3, RDS, IAM, and EB configurations

now that the project was already prepared, I started the configuration of all the AWS services, starting with the RDS, and try the connection on local with the user and password created on the process

![RDS postgres database][image2]

When the DB start working the next process was to create a S3 bucket so the app could save the Feed images, so i created a public s3 bucket, enabeling all the CORS.

![S3 for images][image9]

Once everything started working great, i proccede to create the EB application, manually on the AWS console. but error were showing, so I changed a few code, to fix the routes and also changin the way to add the credentials of the images from the S3 bucket.

### changin the main call on package.json
The initial configuration on the package.json was using the ``src/server.js`` that was wrong for the build because the `server.js` file was located on the root, so the first change was to update the `Package.json` file

Before
```
 "name": "udagram-api",
  "version": "2.0.0",
  "description": "",
  "main": "src/server.js",
 .....
```

After
```
 "name": "udagram-api",
  "version": "2.0.0",
  "description": "",
  "main": "server.js",
 .....
```

### changin profile default on S3 call
After deploying the API to the EB server, there was a problem with the credentials when trying to call and upload new images to the S3 bucket. so the method was that the API was calling the local AWS profile, but when on EB there was no AWS file to take the credentials, so I changed the method called on the `aws.ts` file, passing directly the credentials from the ENV

Before
```
    ...
    // Configure AWS
    const credentials = new AWS.SharedIniFileCredentials({ profile: "default" });
    ...
```

After
```
    ...
    // Configure AWS
    const credentials = new AWS.Credentials({ accessKeyId: process.env.AWS_ACCESS_KEY, secretAccessKey: process.env.AWS_SECRET});
    ...
```

For obtaining the credentials, I needed to create a new user on the IAM service from AWS and give it the following permissions.
![User credentials][image6]

from there the EB Deployment worked fine giving a Health status of OK, and giving us the URL for the API.
![Elastic Beanstalk server Health and running][image3]

Now that the API was up and running, it was time to update the ``apiHost`` value on the `environment.ts` file inside the Frontend application, from localhost to the new url provided by the EB server. the file was located on `src/environments/environment.ts`

Before
```
    ...
    export const environment = {
    production: false,
    appName: 'Udagram',
    apiHost: 'localhost:8080/api/v0'
    };
    ...
```

After
```
    ...
    export const environment = {
    production: false,
    appName: 'Udagram',
    apiHost: 'http://udagram-env.eba-vsctgkv6.us-east-1.elasticbeanstalk.com/api/v0'
    };
    ...
```

Now that everything was updated and working fine on local, it was time to upload the build on the S3 with static website configuration using the AWS console.

![S3 public for the frontend][image7]

## CircleCI configuration

Now that everything was working smothly the next step was to create a pipeline, that help us creating a new deploy, everytime the code was submited on the github repo. for that i created a `config.yml` file with the following structure.

```
version: 2.1
orbs:
  node: circleci/node@5.0.0
  aws-cli: circleci/aws-cli@2.0.6
  aws-ebcli: circleci/aws-elastic-beanstalk@2.0.1
  aws-s3cli: circleci/aws-s3@3.0.0
jobs:
  deployment:
    docker:
      - image: cimg/node:16.10
    steps:
      - node/install
      - checkout
      - aws-cli/setup
      - aws-ebcli/setup
      - run:
          name: Backend-API install
          command: npm run backend:install
      - run:
          name: Frontend install
          command: npm run frontend:install
      - run:
          name: Backend-API testing
          command: npm run backend:test
      - run:
          name: Installing Typescript global
          command: npm i -g typescript
      - run:
          name: Backend-API build
          command: npm run backend:build
      - run:
          name: Frontend build
          command: npm run frontend:build
      - run:
          name: Backend-API deploy
          command: npm run backend:deploy
      - run:
          name: Frontend deploy
          command: npm run frontend:deploy
workflows:
  deploy:
    jobs:
      - deployment
```

As you can notice, there are multiple Orbs mostly the AWS so we can deploy everything to our already created services. Now that everything was setup was time to configure the circleCI with the github repo and giving access to make changes and run. 

Up next you can see all the commits and pipe lines that were running, and at the end there is a fully completed one, that checks everything and also deploys to production with the same github hash.

![CircleCi project pipeline][image4]

![CircleCi complete run][image8]