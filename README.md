# Notes for Edge Service

#### BOOTING UP THIS PROJECT
First, we need both Catalog Service and Order Service 
up and running. From each project’s root folder, 
run `./gradlew bootBuildImage` to package them as container images. 
Then start them via Docker Compose. 
Open a Terminal window, navigate to the folder where your 
`docker-compose.yml` file is located `(polar-deployment/docker)`, 
and run the following command:
```bash
docker-compose up -d catalog-service order-service
```
Since both applications depend on PostgreSQL, Docker Compose 
will also run the PostgreSQL container.
When the downstream services are all up and running, 
it’s time to start Edge Service. 
From a Terminal window, navigate to the project’s root folder 
`(edge-service)`, and run the following command:
```bash
./gradlew bootRun
```
Terminate all the containers by giving the path to 
`docker-compose.yml` file:
```bash
docker-compose -f <path_to_docker-compose.yml> down
```