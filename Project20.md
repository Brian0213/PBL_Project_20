# MIGRATION TO THE СLOUD WITH CONTAINERIZATION. PART 1 – DOCKER & DOCKER COMPOSE

## MySQL in container

Let us start assembling our application from the Database layer – we will use a pre-built MySQL database container, configure it, and make sure it is ready to receive requests from our PHP application.

Step 1: Pull MySQL Docker Image from Docker Hub Registry
Start by pulling the appropriate Docker image for MySQL. You can download a specific version or opt for the latest release, as seen in the following command:

`docker pull mysql/mysql-server:latest`

[Docker Mysql Download](./images/docker-mysql-download.png)

If you are interested in a particular version of MySQL, replace latest with the version number. Visit Docker Hub to check other tags here

List the images to check that you have downloaded them successfully:

`docker image ls`

[Docker Image Output](./images/docker-image-output.png)

- To create a container:

`docker run --name=P20 -it ubuntu bash`

[Docker Create Container](./images/docker-cont-create.png)

- To delete a container:

`docker rm -f "name of container"`

Step 2: Deploy the MySQL Container to your Docker Engine
1. Once you have the image, move on to deploying a new MySQL container with:

`docker run --name=p20 -e MYSQL_ROOT_PASSWORD=deshaun123 -d mysql/mysql-server:latest`

- Replace <container_name> with the name of your choice. If you do not provide a name, Docker will generate a random one
- The -d option instructs Docker to run the container as a service in the background
- Replace <my-secret-pw> with your chosen password
- In the command above, we used the latest version tag. This tag may differ according to the image you downloaded

[Docker Container MySql](./images/docker-contmysql-create.png)

2. Then, check to see if the MySQL container is running: Assuming the container name specified is mysql-server

`docker ps -a`

[Docker PS A](./images/docker-ps-a-output.png)

You should see the newly created container listed in the output. It includes container details, one being the status of this virtual environment. The status changes from health: starting to healthy, once the setup is complete.

### CONNECTING TO THE MYSQL DOCKER CONTAINER

Step 3: Connecting to the MySQL Docker Container
We can either connect directly to the container running the MySQL server or use a second container as a MySQL client. Let us see what the first option looks like.

Approach 1

Connecting directly to the container running the MySQL server:

`docker exec -it mysql bash` or `docker exec -it mysql mysql -uroot -p`

Provide the root password when prompted. With that, you’ve connected the MySQL client to the server.

Finally, change the server root password to protect your database. Exit the the shell with exit command

Flags used

exec used to execute a command from bash itself
-it makes the execution interactive and allocate a pseudo-TTY
bash this is a unix shell and its used as an entry-point to interact with our container
mysql The second mysql in the command "docker exec -it mysql mysql -uroot -p" serves as the entry point to interact with mysql container just like bash or sh
-u mysql username
-p mysql password

`docker run -d --restart always mysql/mysql-server:latest`  

Approach 2

At this stage you are now able to create a docker container but we will need to add a network. So, stop and remove the previous mysql docker container.

`sudo docker ps -a`

[Docker PS A](./images/docker-ps-a-output2.png)

-To stop the container:

`sudo docker stop p20`

[Docker Stop p20](./images/docker-stop-p20.png)

To remove the container:

`sudo docker rm -f p20`

[Docker Remove p20](./images/docker-remove-p20.png)

- verify that the container is deleted:

`sudo docker ps -a`

[Docker Remove p20 Confirm](./images/docker-remove-confirm.png)

First, create a network:

`sudo docker network create --subnet=172.18.0.0/24 tooling_app_network`

[Docker Network Create](./images/docker-create-network.png)

Creating a custom network is not necessary because even if we do not create a network, Docker will use the default network for all the containers you run. By default, the network we created above is of DRIVER Bridge. So, also, it is the default network. You can verify this by running the docker network ls command.

But there are use cases where this is necessary. For example, if there is a requirement to control the cidr range of the containers running the entire application stack. This will be an ideal situation to create a network and specify the --subnet

For clarity’s sake, we will create a network with a subnet dedicated for our project and use it for both MySQL and the application so that they can connect.

Run the MySQL Server container using the created network.

- First, let us create an environment variable to store the root password:

`export MYSQL_PW=`

verify the environment variable is created:

`echo $MYSQL_PW`

[MySql PW Output](./images/mysql-pw.png)

Then, pull the image and run the container, all in one command like below:

`sudo docker run --network tooling_app_network -h mysqlserverhost --name=mysql-server -e MYSQL_ROOT_PASSWORD=$MYSQL_PW  -d mysql/mysql-server:latest`

[MySql Network link](./images/mysql-network.png)

Flags used

-d runs the container in detached mode
--network connects a container to a network
-h specifies a hostname

If the image is not found locally, it will be downloaded from the registry.

Verify the container is running:

`sudo docker ps -a`

[Docker Container run](./images/cont-run-confirm.png)

As you already know, it is best practice not to connect to the MySQL server remotely using the root user. Therefore, we will create an SQL script that will create a user we can use to connect remotely.

Create a directory:

`mkdir Sql`

cd into the directory

`cd Sql`

Create a file in the directory (touch fileName – Create an empty file. cat > filename – To create a new text file under Ubuntu, use the cat command followed by the redirection symbol ( > ) and the name of the file you want to create. Press Enter, then type the text and once you are done, press the CRTL+D to save the file):

`cat > create_user.sql`

Create a file and name it create_user.sql and add the below code in the file:

`CREATE USER 'ludo'@'%' IDENTIFIED BY 'deshaun123'; GRANT ALL PRIVILEGES ON * . * TO 'ludo'@'%';`

[Create file create_user.sql](./images/file-output.png)

Run the script:
Ensure you are in the directory create_user.sql file is located or declare a path:

`sudo docker exec -i mysql-server mysql -uroot -p$MYSQL_PW < create_user.sql`

[Create file create_user.sql](./images/user-sql-output.png)

Connecting to the MySQL server from a second container running the MySQL client utility
The good thing about this approach is that you do not have to install any client tool on your laptop, and you do not need to connect directly to the container running the MySQL server.

Run the MySQL Client Container:

`sudo docker run --network tooling_app_network --name mysql-client -it --rm mysql mysql -h mysqlserverhost -uludo  -p$MYSQL_PW`

[Connect Mysql Tooling](./images/tooling-mysql.png)

Flags used

-d runs the container in detached mode
--network connects a container to a network
-h specifies a hostname
If the image is not found locally, it will be downloaded from the registry.

Prepare database schema
Now you need to prepare a database schema so that the Tooling application can connect to it.

Clone the Tooling-app repository from here:

`git clone https://github.com/darey-devops/tooling.git`

On your terminal, export the location of the SQL file:

`export tooling_db_schema=tooling/html/tooling_db_schema.sql`

You can find the tooling_db_schema.sql in the tooling/html/tooling_db_schema.sql folder of cloned repo.

Verify that the path is exported

`echo $tooling_db_schema`

[Echo Tooling Schema](./images/echo-tool-schema.png)

Use the SQL script to create the database and prepare the schema. 

[Echo Tooling Schema](./images/sql-to-schema.png)

Run the command:

`cat $tooling_db_schema`


[Echo Tooling Schema](./images/cat-dbstart.png)

[Echo Tooling Schema](./images/cat-db2.png)


With the docker exec command, you can execute a command in a running container.

`sudo docker exec -i mysql-server mysql -uroot -p$MYSQL_PW < $tooling_db_schema`

[Exec Tooling Schema](./images/exec-schema.png)

Update the .env file with connection details to the database
The .env file is located in the html tooling/html/.env folder but not visible in terminal. you can use vi or nano

sudo vi .env

MYSQL_IP=mysqlserverhost
MYSQL_USER=username
MYSQL_PASS=client-secrete-password
MYSQL_DBNAME=toolingdb
Flags used:

MYSQL_IP mysql ip address "leave as mysqlserverhost"
MYSQL_USER mysql username for user export as environment variable
MYSQL_PASS mysql password for the user exported as environment varaible
MYSQL_DBNAME mysql databse name "toolingdb"

Navigate to the db_conn.php file inside the html folder inside the tooling project and update the db credentials

Ensure you are in the html folder, In the terminal, run the command:

`sudo vi db_conn.php`

[Db Conn.php Update](./images/db-conn-php.png)

Run the Tooling App
Containerization of an application starts with creation of a file with a special name - 'Dockerfile' (without any extensions). This can be considered as a 'recipe' or 'instruction' that tells Docker how to pack your application into a container. In this project, you will build your container from a pre-created Dockerfile, but as a DevOps, you must also be able to write Dockerfiles.

You can watch this video to get an idea how to create your Dockerfile and build a container from it.

And on this page, you can find official Docker best practices for writing Dockerfiles.

So, let us containerize our Tooling application; here is the plan:

Make sure you have checked out your Tooling repo to your machine with Docker engine
First, we need to build the Docker image the tooling app will use. The Tooling repo you cloned above has a Dockerfile for this purpose. Explore it and make sure you understand the code inside it.
Run docker build command
Launch the container with docker run
Try to access your application via port exposed from a container
Let us begin:

Ensure you are inside the directory "tooling" that has the file Dockerfile and build your container :

`sudo docker build -t tooling:0.0.1 .`

[Docker Build](./images/docker-build1.png)

[Docker Build](./images/docker-build2.png)

In the above command, we specify a parameter -t, so that the image can be tagged tooling"0.0.1 - Also, you have to notice the . at the end. This is important as that tells Docker to locate the Dockerfile in the current directory you are running the command. Otherwise, you would need to specify the absolute path to the Dockerfile.

Run the container:

`sudo docker run --network tooling_app_network -p 8085:80 -it tooling:0.0.1`

[Docker Build](./images/docker-build1.png)

Let us observe those flags in the command.

We need to specify the --network flag so that both the Tooling app and the database can easily connect on the same virtual network we created earlier.
The -p flag is used to map the container port with the host port. Within the container, apache is the webserver running and, by default, it listens on port 80. You can confirm this with the CMD ["start-apache"] section of the Dockerfile. But we cannot directly use port 80 on our host machine because it is already in use. The workaround is to use another port that is not used by the host machine. In our case, port 8085 is free, so we can map that to port 80 running in the container.
Note: You will get an error. But you must troubleshoot this error and fix it. Below is your error message.

Use your public IP address and expose port 8085 in your security group to access the site on your browser:

[Expose 8085 Output](./images/8085-output.png)

AH00558: apache2: Could not reliably determine the server's fully qualified domain name, using 172.18.0.3. Set the 'ServerName' directive globally to suppress this message

`sudo docker run --network tooling_app_network -p 8085:80 -it tooling:0.0.1`

Launch the Url below:

`http://18.203.245.99:8085/`

[URL Success](./images/url-output.png)