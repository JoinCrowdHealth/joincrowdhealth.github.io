---
layout: post
title:  "Database migrations on AWS Elastic Beanstalk Docker platform"
date:   2023-04-13 09:30:13 -0500
categories: devops aws
author: Blake Gardner
---
At Crowd Health, we migrated to Amazon Web Services because of the added flexibility over our previous hosting solution. We decided to use the [Docker platform branch of AWS Elastic Beanstalk](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/create_deploy_docker.html) to manage our deployments since we use Docker for local development and our CI/CD pipelines.

In the provided example the stack is Ruby on Rails, however this method could work for any language or framework that uses migrations, simply substitute the `/app/bin/rails db:migrate` and `/app/bin/rails db:seed` commands provided in the example.

The Docker platform branch provides the ability to trigger scripts throughout the deployment using [platform hooks](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/platforms-linux-extend.html) such as `“prebuild”`, `“predeploy”` and `“postdeploy”`.

### The solution

Add this script to the `.platform/hooks/postdeploy` directory of your project.

```shell
#!/bin/sh
if [[ "$EB_IS_COMMAND_LEADER" == "true" ]]; then
  sudo docker exec -i $(sudo docker ps --format "{{.ID}}") /app/bin/rails db:migrate
  sudo docker exec -i $(sudo docker ps --format "{{.ID}}") /app/bin/rails db:seed
fi
```

### How this works

Note the usage of the environment variable `$EB_IS_COMMAND_LEADER`. This is used to ensure the migrations only run one time in case you are deploying to multiple servers, this will only execute on the EC2 host that is considered the "leader". The script finds the running container on the EC2 host and executes them inside the running container.
