## Instructions on getting a schema from a Supabase production instance and updating a test instance
---

This guide will leverage a subset of the installed tools in the official Postgres docker image to pull a SQL schema from a Supabase instance and save it to a local file, and then explain how to set up a new Supabase instance and run the generated file to match the table layout of the original instance. 

### Checking your system for docker installation

The rest of guide assumes you already have Docker installed on your system and that both the Docker client and the daemon are running. You can check this by running `docker info` in your terminal and you should get an output that has 2 or 3 sections divided by an empty line. The first section should be `Client:` followed by the cli client info, and the second section should be `Server:` with the daemon info. 
- Depending on your system config and Docker config, you may have docker installed but it may not start automatically. This will lead to an error saying something to the effect of "Docker is not a valid command". Make sure to start docker and then try `docker info` again
- If you need to install Docker, you can do so [at the official site here](https://docs.docker.com/get-docker/)
- Check which shell you are using. This guide should work on any unix-like shell - bash, zsh, standard Mac terminal etc. If you are using a different shell (Powershell, etc.) you might have to adapt commands to your shell on your own. 

### Gathering required information

In order to connect to the running supabase instance in the cloud, you will need to gather some information from the cloud dashboard. Fortunately it's all in the same place. 

- In a browser, navigate to Supabase, sign in, and open the project that contains the schema you want to copy. 
- Go to settings on the bottom left of the dashboard, and then go to the `Database` section. 
- In Database Setting, scroll down to the "Connection String" section, select the URI tab, and copy the provided connection string
  - Note that the connection string has a section labeled `[YOUR-PASSWORD]` - you will need to replace this for the connection to succeed, see the next step
- You will also need the database password that you provided when you originally set up the instance. If you do not have that password for some reason, this page has a section near the top for "Database Password" that will allow you to reset it, but **--WARNING--** - if you have any applications connected to the database that are relying on that password in a connection string, they will fail to connect once you change the password! Be sure you won't break anything before you reset the password! And AS ALWAYS, KEEP YOUR PASSWORD SAFE! 

### Setting up the docker container for use

- Run the command to get the postgres container built and running
  - `docker run -d --name postgres1 -e POSTGRES_PASSWORD=startup1234 postgres`
  - The breakdown of this command is as follows: 
    - The `-d` flag tells docker to run the container "detached" so you will still have your terminal instance available after the container starts
    - The `--name` flag gives the container a label so it's easy to find later. I picked 'postgres1' for this guide as an example. 
    - the `-e` flag sets the password environment variable in the container - required for it to start
    - the `postgres` on the end dictates which container image to use. 
 - Once the container is downloaded and started, the terminal will output a hash for the container and return control back to you. This means the container is started, but give it about a minute to finish setting up the database internally.
 
### Extracting the schema from Supabase

This section will leverage the `pg_dump` tool inside your newly created postgres container

- In your terminal, navigate to the directory that you want to use to store the generated SQL file. 
- Run the command `docker exec postgres1 pg_dump -d <Your-database-connection-uri> --schema-only --schema=public > schema.sql`
  - Don't forget to put the database URI you located early into the `<Your-database-connection-uri>` section
  - Also don't forget to substitute in your db password. 
  - This command might take some to run, depending on how big your schema is. Be patient - a few minutes is probably too long though. 
  - You may get an error starting with `pg_dump: error: server....` - I'll address this in the next section if you do. 
  - This command breaks down as follows: 
    - `docker exec` tells docker that you want to execute a command inside a running container  
    - `postgres1` is the name of the container you previously created. Anything after this is run inside the container, up to the `>` character we'll get to below.  
    - `pg_dump -d <Your-database-connection-uri>` is telling postgres to pull a DB dump from the database you specify through the URI
    - `--schema-only` **CRITICAL** this flag tells pg_dump to only pull the schema. If omitted, it will instead pull the schema and all data!
    - `--schema=public` tells pg_dump to only output the "public" schema. 
      - When Supabase initialized your database, it included several other schemas related to auth and internals, and your new instance will have those set up by default. Therefore you only need the public schema at this point. 
    - `> schema.sql` tells your shell to pipe the output of the command to a local file named `schema.sql`. This file will be created in your local filesystem outside of the docker container, in the directory you navigated to earlier. 
- Once you have the schema file generated, open it with a text editor and navigate to the line `CREATE SCHEMA public;`, and DELETE this line and EVERY LINE BEFORE IT! Then save the file. 
- Remember where this file it located, and it's now ready to use. 
- Stop the container by running `docker stop postgres1`

### If you get version error from pg_dump

You may see a version error that looks like this: 

![PG_dump error](https://user-images.githubusercontent.com/46945607/217048572-31e50a59-3a4d-44eb-92ee-09c4d00366d7.jpg)

What this is telling you is that the Postgres version in your docker container and the postgres version Supabase is using for your instance don't match. In the picture above, the `server version` is what we need to match for this to work properly. Fortunately you just need to specify a version when you set up your docker container. In the case of the picture above, version `15.1` is called for. 
- The new command to set up the container will be `docker run -d --name postgres1 -e POSTGRES_PASSWORD=startup1234 postgres:15.1` - this tags the postgres image with a version and makes sure you get the correct version
- the `--name postgres1` flag might produce an error if you already have a container with the name `postgres1` - you have 2 options
  - Delete the existing container that already has that name and then start again with the tagged image
  - change the name to `--name <your-choice>` to specify any name you want for the container. Just remember to substitute it in these instructions where needed. 

### Tearing down the docker container after use

Once this process is done you might not want the postgres container hanging around on your system. **NOTE** - you might want to do the next section first in case you still need a container for some reason. 
- You can list all containers on your system by running `docker ps -a`. This will show all running and stopped containers. 
- You can delete a container by running `docker rm <container-name>` - in this case `docker rm postgres1`
- You can also delete the local copy of the image by running the following steps: 
  - Run `docker image ls` to list local images. You can find the image you pulled for postgres and also the tag, which will either be "latest" or a version number if you had to specify a version above. Make note of the tag if it's not "latest"
  - run `docker image rm postgres` if the image was tagged as "latest" and the image will be deleted
  - run `docker image rm postgres:<tag>` if you had to pull a specific version. 
- DON'T FORGET to address all the containers and images you created through this process.   

### Starting the new Supabase instance and importing the schema

- In a web browser, navigate to Supabase and start a new project if you haven't already. 
- Once you go through the initial setup, allow enough time for Supabase to provision the database. 
- Then navigate to the SQL editor on the left side menu buttons
- On the top left, hit the button for `New Query`. This should pull up a blank SQL editor box. 
- Copy the entire contents of the `schema.sql` file that you made previously and past it into the editor box
- On the bottom-right corner of the editor box, there is a `Run` button - click it
- You should get an console output stating that the script ran and no rows were returned
- You can check that all tables were added by going to the database button on the left hand menu and looking at the `Tables` option, which should display all the tables you just built. 

### If all went according to plan, your new Supabase instance should be ready to connect to and use. 
  
  
