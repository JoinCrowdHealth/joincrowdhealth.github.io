---
layout: post
title:  "Adding log groups to EB project"
date:   2023-04-13 09:30:13 -0500
categories: jekyll update
---

`docker-compose.yml`

```yaml
services:
  web:
    volumes:
      - /var/log/eb-docker/containers/eb-current-app/container:/app/log
```


`Dockerrun.aws.json`
```json
{
  "Volumes": [
    {
      "HostDirectory": "/var/log/eb-docker/containers/eb-current-app/container",
      "ContainerDirectory": "/app/log"
    }
  ]
}
```


`.platform/hooks/postdeploy/01_restart_cloudwatch_agent.sh`

```shell
#!/bin/sh

# This script makes the CloudWatch agent reload the configuration file after modification in the `predeploy` step

/opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a fetch-config -c file:/opt/aws/amazon-cloudwatch-agent/etc/beanstalk.json -s
```

`.platform/hooks/predeploy/01_configure_log_groups.py`

```python
#!/usr/bin/env python

# This script modifies the AWS CloudWatch agent configuration to include additional log groups. It reads and parses the existing JSON
# config file at defined with LOG_CONFIG_FILE and then compares it to the `list_of_log_files` list.
# If the log group does not exist, it is added to the parsed JSON and then the JSON file is written back to disk.

import os
import json
import subprocess

# Define the location of the log group config
LOG_CONFIG_FILE = '/opt/aws/amazon-cloudwatch-agent/etc/beanstalk.json'

# Define a dynamic environment name based on the current Elastic Beanstalk environment
ENV_NAME = subprocess.check_output(['/opt/elasticbeanstalk/bin/get-config', 'container', '-k', 'environment_name'])

# Define the list of log files to append to the log group config
list_of_log_files = [
    {
        "file_path": "/var/log/eb-docker/containers/eb-current-app/container/synctera_api.log",
        "log_group_name": "/aws/elasticbeanstalk/{}/var/log/eb-docker/containers/eb-current-app/container/synctera_api.log".format(ENV_NAME),
    }
]

# Ensure the Beanstalk config exists before continuing
if not os.path.exists(LOG_CONFIG_FILE):
    exit("Beanstalk config file missing, not setting up new log groups.")

# Read the external JSON file
with open(LOG_CONFIG_FILE) as json_file:
    data = json.load(json_file)

# Loop over the list of dictionaries
for log_file in list_of_log_files:
    # Check the JSON for a matching value
    for item in data["logs"]["logs_collected"]["files"]["collect_list"]:
        if item["file_path"] == log_file["file_path"]:
            # found a match, skipping this iteration and move onto the next
            break
    # This block is a Python feature for/else, executes only after the for is done and no `break` is encountered
    # https://docs.python.org/3/tutorial/controlflow.html#break-and-continue-statements-and-else-clauses-on-loops
    else:
        # No match found, add new item to the JSON array
        new_item = {
            "file_path": log_file["file_path"],
            "log_group_name": log_file["log_group_name"],
            "log_stream_name": "{instance_id}",
        }
        data["logs"]["logs_collected"]["files"]["collect_list"].append(new_item)
        print("Added {}".format(log_file['file_path']))

# Write the updated JSON file back to disk
with open(LOG_CONFIG_FILE, "w") as json_file:
    json.dump(data, json_file, indent=4)
```