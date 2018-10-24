[logo]: https://res.cloudinary.com/snyk/image/upload/v1533761770/logo-1_wtob68.svg
![Snyk Security Scanning](https://res.cloudinary.com/snyk/image/upload/v1533761770/logo-1_wtob68.svg)

# Snyk Codefresh Example [![Codefresh build status]( https://g.codefresh.io/api/badges/pipeline/aarlaud/snyk-playground%2Fcodefresh-pipeline-snyk-app-docker-scan%2Fcodefresh-pipeline-snyk-app-docker-scan?branch=master&type=cf-2)]( https://g.codefresh.io/repositories/snyk-playground/codefresh-pipeline-snyk-app-docker-scan/builds?filter=trigger:build;branch:master;service:5bcfd52a9fd1f422617b0eb3~codefresh-pipeline-snyk-app-docker-scan)


This example application has a sample application along with a Codefresh pipeline that can build, scan, and promote a Docker image.

*Notice* These instructions are new, if you run into any issues running this yourself, please create an issue. 
## Using the plugin
### Scan Code
```
  SnykAppScan:
    title: Snyk Test Application Dependencies
    stage: scan
    image: '${{BuildingDockerImage}}'
    working_directory: IMAGE_WORK_DIR
    environment:
      - SNYK_TOKEN=${{SNYK_TOKEN}}
      - SNYK_ORG=${{SNYK_ORG}}
    commands:
      - npm install -g snyk
      - snyk test --severity-threshold=high
    on_success:
      metadata:
        set:
          - '${{BuildingDockerImage.imageId}}':
              - CF_QUALITY: true
    on_fail:
      metadata:
        set:
          - '${{BuildingDockerImage.imageId}}':
              - CF_QUALITY: false
```

### Scan Docker Image
```
  SnykScanImage:
      stage: scan
      type: composition
      composition:
        version: '2'
        services:
          targetimage:
            image: ${{BuildingDockerImage}} # Must be the Docker build step name
            command: sh -c "exit 0"
            labels:
              build.image.id: ${{CF_BUILD_ID}} # Provides a lookup for the composition
      composition_candidates:
        scan_service:
          image: aarlaudsnyk/snyk-container-scan-docker
          command: python snyk-cli.py "${{IMAGE_NAME}}:${{CF_BRANCH_TAG_NORMALIZED}}"
          environment:
          - SNYK_TOKEN=${{SNYK_TOKEN}}
          - SNYK_ORG=${{SNYK_ORG}}
          - CFCR_ACCOUNT=${{CFCR_ACCOUNT}}
          - CF_USER_NAME=${{CF_USER_NAME}}
          - CFCR_LOGIN_TOKEN=${{CFCR_LOGIN_TOKEN}}
          depends_on:
            - targetimage
          volumes: # Volumes required to run DIND
            - /var/run/docker.sock:/var/run/docker.sock
            - /var/lib/docker:/var/lib/docker
      add_flow_volume_to_composition: true
      on_success: # Execute only once the step succeeded
        metadata: # Declare the metadata attribute
          set: # Specify the set operation
            - ${{BuildingDockerImage.imageId}}: # Select any number of target images
              - SECURITY_SCAN: true

      on_fail: # Execute only once the step failed
        metadata: # Declare the metadata attribute
          set: # Specify the set operation
            - ${{BuildingDockerImage.imageId}}: # Select any number of target images
              - SECURITY_SCAN: false
```

## Instructions

### Pre-requisites
- [Codefresh account](https://codefresh.io/) (free or paid)
- [Snyk account](https://snyk.io/) (free or paid - Docker scanning is a paid feature, contact Snyk at snyk.io about it)
- Dockerhub account (Optional)

### Add Repo to Codefresh
Signin to Codefresh and click "Add Repository" from the [repositories screen](https://g.codefresh.io/repositories). Paste in the url for this repo and click next. Then select "I have a Codefresh.yml" and put `./.codefresh/codefresh.yml` for the path. This will preview the Codefresh yaml, then follow the instructions to finish creating the pipeline.

### Add Environmental Variables
You can type in the variables by hand, or just copy and paste the following:
```
PORT=8080
SNYK_ORG=<your snyk organization name>
IMAGE_NAME=<imagename of your choice>
SNYK_TOKEN=<Snyk API token>
CFCR_ACCOUNT=<your Codefresh Registry account>
CF_USER_NAME=<your Codefresh username>
CFCR_LOGIN_TOKEN=<your Codefresh Registry login token>
```

Select "Import from Text" to import.

We'll also add a token from Snyk. You can get this from your [Snyk account settings](https://app.snyk.io/account). Add this variable with `SNYK_TOKEN` as the key. Then check encrypt to store the token securely.
The Codefresh registry bit is optional (remove the lines from the yaml if you do not want to use it). We are testing the docker image built and stored in Codefresh registry before pushing it to Docker Hub if all Snyk tests are successful.

### Add Dockerhub (optional)
Codefresh has a built-in private Docker registry. In this example we're building and pushing a public image so we'll use Docker hub. Follow the instructions in the [Docker Registry integration page](https://g.codefresh.io/account-conf/integration/registry).

You can skip this step by removing the promote to Dockerhub step. If using your own registry, update the name in the push to repo step at the end of the pipeline. 

### Go run your pipelne.
