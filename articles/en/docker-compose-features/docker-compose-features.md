# Cool features in Docker Compose — profiles and templates
Let us tell you a story. A story of how we developed an API and decided to cover it with E2E-tests. Those were simple tests describing and checking API functionality, which turned out to be difficult to run. But first things first.
In this article, we will consider a solution that uses simple Docker Compose configurations.
## Manual test run
We were searching for a handy tool to write E2E-tests for the  API. Pretty soon we came across a tool called [Karate](https://karatelabs.github.io/karate/). We browsed the documentation and at first decided to run tests manually using the following command:
```bash
java -jar karate.jar .
```
But we found that we’ll have to install Karate and the Java-runtime for it, and then write an instruction about it for three operating systems — Windows, Linux, and macOS.
## Using Docker Compose
To avoid installing a bunch of tools (and specific versions of them!), we decided to run tests in Docker, where all of the dependencies are described in the Dockerfile and installed automatically during the container build process. Here is an example of our Dockerfile for running Karate-tests, inspired by the official [documentation](https://github.com/karatelabs/karate-examples/blob/main/docker/README.md). To run tests in Docker, we decided to use Docker Compose, because it allows you to start multiple services with a single command. In our case it is the API, database and container with tests: ![](https://habrastorage.org/webt/fd/72/rh/fd72rhkzmu6ar5lpbx_h_qa_iwc.jpeg)
To run all these services, we wrote a `docker-compose.yml` file.
```yaml
version: '3.8'

services:
  db:
    image: postgres:13
    container_name: 'db'
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: postgres
    ports:
      - 5432:5432

  api:
    container_name: 'api'
    build:
      context: .
      dockerfile: Api/Dockerfile # Path to the docker file from which the API image is retrieved
    ports:
      - 5000:80
    depends_on: # When starting the API, migration is applied to the database automatically,
                # therefore the API is only launched when the database is already launched
      - db

  karate_tests:
    container_name: 'karate_tests'
    build:
      dockerfile: KarateDockerfile  # Path to the file from which you build
                                    # Karate image (downloads karate.jar file,
                                    # and Java already exists inside the container)
      context: .
    depends_on:
      - api # If the API is not running, the tests will fail, so we wait for the API to run
    command: ["/karate"] # Run tests from /karate folder
    volumes:
      - .:/karate # Mount folder with tests to /karate folder
    environment:
      API_URL: 'http://api'
```
**KarateDockerfile**
```dockerfile
FROM openjdk:11-jre-slim

RUN apt-get update && apt-get install -y curl

RUN apt-get install -y unzip

RUN curl -o /karate.jar -L 'https://github.com/intuit/karate/releases/download/v1.3.0/karate-1.3.0.jar'

ENTRYPOINT ["java", "-jar", "/karate.jar"]
```
## Two Docker Compose files
Looks good and convenient, but not everyone in the team needs all of these containers. A developer, for example, only needs the database because they run the API in the IDE. We decided to split the Docker Compose file into two files: `docker-compose.yml`, which remained unchanged, and `docker-compose-db.yml`, which contained just the database container:
**docker-compose-db.yml**
```yaml
version: '3.8'

services:
  db:
    image: postgres:13
    restart: always
    container_name: 'db'
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: postgres
    ports:
      - 11237:5432
```
## Running tests in pipeline
Our convenient configuration is ready, which means we can run tests in the pipeline on the merge request into the main branch. To do so, we used the `docker-compose.yml` file, because it contained all necessary containers to run tests. 
We wrote a pipeline for GitHub, where we started all containers, including tests.
```yaml
run: |
LOGS=$(docker-compose --file docker-compose.yml up --abort-on-container-exit) 
# Write all logs into a variable, 
# to further examine them and check
# if the tests passed or not

# Check that there are no failed tests
if [ "$FAILED" -gt 0 ]; then
  echo "Failed tests found! Failing the pipeline..."
  exit 1
fi
# Check that the tests have passed to avoid false successful completion of the pipeline
if [ "$PASSED" -eq 0 ]; then
  echo "No tests passed! Failing the pipeline..."
  exit 1
fi
```
It's a bit of a crutch, but so far we haven’t found a better way to fail the pipeline if Karate-tests are not completed successfully. In fact, we can use two files to build containers, specifying them using **--file flag**:
```bash
docker-compose --file docker-compose.yml --file docker-compose-db.yml up
```
But this solution is not the best idea. One can get confused between the files and will need to check that the service has not been duplicated in several files. Moreover, there will be a lot of files — as many as there are run configurations. We singled out at least the following:
- **local-environment** — runs the database and API;
- **db-only** — runs only the database to interact with the API started in the IDE;
- **e2e-local-environment** — runs the API, database, and container with Karate-tests for the API started in Docker;
- **e2e-production-environment** — since we are enthusiasts of the test-driven development approach and production testing, we want to run tests not only on a feature branch before merging into the main branch, but also after deployment to a production environment. This configuration runs only the karate_tests container, which is oriented toward the production.
## Profiles
We browsed the Docker Compose documentation and found a convenient tool — [profiles](https://docs.docker.com/compose/profiles/). This feature helps to split services as needed, and keep them in one file. So, to start the necessary services we will need to specify *--profile* argument in the docker-compose up command with the necessary profile name.
```bash
docker-compose --profile db-only up
```
We merged all the containers back into one `docker-compose.yml` file again and spread the profiles between them to choose only the containers we need right now.
```yaml
version: '3.8'

services:
  db:
    image: postgres:13
    container_name: 'db'
    profiles: ['db-only', 'e2e-local-environment', 'local-environment'] # Service will be started only when running with these profiles
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: postgres
    ports:
      - 5432:5432

  api:
    container_name: 'api'
    profiles: ['e2e-local-environment', 'local-environment']
    build:
      context: .
      dockerfile: Api/Dockerfile
    ports:
      - 5000:5000
    depends_on:
      - db

  karate_tests:
    container_name: 'karate_tests'
    profiles: ['e2e-local-environment']
    build:
      dockerfile: KarateDockerfile
      context: .
    depends_on:
      - api
    command: ['/karate']
    volumes:
      - .:/karate
    environment:
      API_URL: 'http://api'
```
## Templates
Now we need to run the same E2E tests, but in production. To do so, we need to replace the address in API_URL to address in production.
We created a second karate_tests service, where the API_URL environment variable has a different value.
```yaml
version: '3.8'

services:
  # other services (API, database)

  local_karate_tests:
    container_name: 'local_karate_tests'
    profiles: ['e2e-local-environment']
    build:
      dockerfile: KarateDockerfile
      context: .
    depends_on:
      - api
    command: ['/karate']
    volumes:
      - .:/karate
    environment:
      API_URL: 'http://api'

  production_karate_tests:
    container_name: 'production_karate_tests'
    profiles: ['e2e-production-environment']
    build:
      dockerfile: KarateDockerfile
      context: .
    depends_on:
      - api
    command: ['/karate']
    volumes:
      - .:/karate
    environment:
      API_URL: 'https://my-deployed-service.com'
```
But it turns out that these two services that run Karate-tests are almost completely duplicated. The only thing that differs between them is the API_URL. We solved this problem using the Docker Compose  [extends](https://docs.docker.com/compose/multiple-compose-files/extends/) block, which allows a service to inherit from some other service and override only a part of the configuration that differs between the service and its inheritor.
We created the **base_karate_tests** template in the `docker-compose.yml` file. It contains the data that containers do not change: KarateDockerfile from which the image is built, the command to run, and the volume.
Now let us apply this template to the services using the extends block in this way:
```yaml
version: '3.8'

services:
  # other services (API, database)

  base_karate_tests:
    build:
      dockerfile: KarateDockerfile
      context: .
    command: ['/karate']
    volumes:
      - .:/karate  

  local_karate_tests:
    container_name: 'local_karate_tests'
    profiles: ['e2e-local-environment']
    extends:
      service: base_karate_tests
    environment:
      API_URL: 'http://api'

  production_karate_tests:
    container_name: 'production_karate_tests'
    profiles: ['e2e-local-environment']
    extends:
      service: base_karate_tests
    environment:
      API_URL: 'https://my-deployed-service.com'
```
One template does not take up much space. However, if we have a system with more templates and services to inherit, the template itself can be moved to a separate file and refer to it. For this purpose, let's create the `templates.yml` file.
```yaml
version: '3.8'

services:
  base_karate_tests:
    build:
      dockerfile: KarateDockerfile
      context: .
    command: ['karate', '/karate']
    volumes:
      - .:/karate
```
Here we will describe all necessary templates, and in the `docker-compose.yml` file we will use the file parameter, in addition to the service parameter, to apply a template from another file.
```yaml
extends:
  file: templates.yml
  service: base_karate_tests
```
It may not be necessary for a small system, but for more complex automation it might be the way to go.
## Summary
Using Docker Compose we were able to successfully and happily run E2E tests of the API in both the pipeline merge request before closing the feature and after deploying that feature to production.
When running in Docker Compose, the solution underwent the following evolution:
- Separate files to run specific services where the manifest is duplicated;
- Binding services to the profiles in which they should run;
- Extracting duplicated service code into templates.
All in all, we have a concise Docker Compose file, from which we can start only the required services with a single command and only one profile argument. Also this solution is extensible when adding new run configurations.
If you've encountered similar problems, we’ll be happy to know how you solved them.