# Docker Compose Setup for Project

Before you can run this project using Docker Compose, make sure you have created a `.env` file with the necessary environment variables. 

Here are the steps to set up the project:
Note : **Make Sure Your /helper/source/init.sql have the data**
1. Create a new file named `.env` in the root directory of the project.
2. Open the `.env` file and add the following environment variables:

```
# Source
SRC_POSTGRES_DB=olist-src
SRC_POSTGRES_HOST=localhost
SRC_POSTGRES_USER=postgres
SRC_POSTGRES_PASSWORD=[YOUR PASSWORD]
SRC_POSTGRES_PORT=[YOUR PORT]

# DWH
DWH_POSTGRES_DB=olist-dwh
DWH_POSTGRES_HOST=localhost
DWH_POSTGRES_USER=postgres
DWH_POSTGRES_PASSWORD=[YOUR PASSWORD]
DWH_POSTGRES_PORT=[YOUR PORT]

```

Now you are ready to run the project using Docker Compose. Use the following command in the terminal:

```
docker-compose up -d
```

This will start the project and all its dependencies defined in the `docker-compose.yml` file.
