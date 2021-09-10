# Backend

## Introduction

In this lab,
- You will build a Python/Flask Docker image.
- you will deploy the Docker image to OKE.
- then configure the API Gateway.

Estimated time: ~45 -minutes.

### Understanding the Python/backend application

As with most React applications (https://reactjs.org/), this application uses remote APIs to handle data persistence. The backend implements 5 REST APIs including:
- Retrieving the current list of todo items
- Adding a new todo item
- Updating an existing todo items
- Deleting a todo item.


### Objectives

* Set values for environment variables
* Build and deploy the Docker image of the application
* Deploy the image on the Oracle Kubernetes Engine (OKE)
* Describe the steps for Undeploying
* Configure the API Gateway
* Test the backend application

### Prerequisites

- This lab requires the completion of lab 1 and the provision of the OCI components.


## **STEP 1**: Set values for workshop environment variables

1. Set the root directory of the workshop
	```
	<copy>export MTDRWORKSHOP_LOCATION=~/mtdrworkshop</copy>
	```
2. Run source addAndSourcePropertiesInBashrc.sh

	The following command will set the values of environment variables in mtdrworkshop.properties and source ~/.bashrc

	```
	<copy>cd $MTDRWORKSHOP_LOCATION; source addAndSourcePropertiesInBashrc.sh</copy>
	```

## **STEP 2**: Build and push the Docker images to the OCI Registry

1. Ensure that the "DOCKER_REGISTRY" variable is set

 Example: `<region-key>.ocir.io/<object-storage-namespace>/<firstname.lastname>/<repo-name>`
 If the variable is not set or is an empty string, the push will fail (but the docker image will be built).

2. copy the Python/Flask Todo app from github.

     ```
     <copy>
      mkdir ~/mtdrworkshop/python
      cd ~/mtdrworkshop/python
      git clone https://github.com/vijaybalebail/Todo-List-Dockerized-Flask-WebApp.git
      cd Todo-List-Dockerized-Flask-WebApp
  	 ```

3. Unzip the database wallte.zip file within the new web app.
     ```
     <copy>unzip ~/mtdrworkshop/setup-dev-environment/wallet.zip</copy>
  	 ```

3.  Pick mtdrb_tp service alias (see the list of aliases in
   ./tnsnames.ora)

   ![](images/tnsnames-ora.png " ")

4. There are many ways to pass database credentials from the backend to Oracle database. We are using a config file.
   Edit the config.cfg file and edit the username,password and connect string. This user and password  is the one created during lab_setup.

     ```
     <copy>
     ADB_USER="TODOUSER"
     ADB_PASSWORD="Oracle"
     ADB_CONNECTSTRING="mtdrdb_tp"
     </copy>
     ```

7. We now can build the a docker image with Python, Oracle Client , and the todo application app.js.
   Look at the construct of the Dockerfile and execute the command to build the docker image.

    ```
    <copy> docker build  -t todolist-flask:latest . </copy>

     (us-ashburn-1)$ docker build  -t todolist-flask:latest .
    Sending build context to Docker daemon  1.297MB
    Step 1/15 : FROM oraclelinux:7-slim
    Trying to pull repository docker.io/library/oraclelinux ...
    7-slim: Pulling from docker.io/library/oraclelinux
    7627bfb99533: Pull complete


    Step 15/15 : CMD python3 app.py
     ---> Running in 94ef1e230b45
    Removing intermediate container 94ef1e230b45
     ---> 02f268c26542
    Successfully built 02f268c26542
    Successfully tagged todolist-flask:latest
    ```

    Verify that the images are CREATED.
    ```
    $ <copy>
    docker images</copy>
    REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
    todolist-flask      latest              02f268c26542        34 seconds ago      477MB
    oraclelinux         7-slim              0a28ba78f4c9        2 months ago        132MB

    ```
  In a couple of minutes, you should have successfully built and pushed the images into the OCIR repository.

8. Check your container registry from the root compartment
    - Go to the Console, click the hamburger menu in the top-left corner and open
    **Developer Services > Container Registry**.

   ![](images/registry_root_compartment.png " ")

9. Mark Access as Public  (if Private)  
   (**Actions** > **Change to Public**):

   ![](images/Public-access.png " ")

## **STEP 3**: Create ImagePullSecret

## **STEP 3**: Deploy on Kubernetes and Check the Status

1. Run the `deploy.sh` script

	```
	<copy>cd $MTDRWORKSHOP_LOCATION/backend; ./deploy.sh</copy>
	```

	--> service/todolistapp-helidon-se-service created

	--> deployment.apps/todolistapp-helidon-se-deployment created

2. Check the status using the following commands
$ kubectl get services

	The following command returns the Kubernetes service of MyToDo application with a load balancer exposed through an external API
	```
	<copy>kubectl get services</copy>
	```

	![](images/K8-service-Ext-IP.png " ")

3. $ kubectl get pods
	```
	<copy>kubectl get pods</copy>
	```

	![](images/k8-pods.png " ")

4. Continuously tailing the log of one of the pods

  $ kubectl logs -f <pod name>
  Example kubectl lgs -f todolistapp-helidon-se-deployment-7fd6dcb778-c9dbv

  Returns:
  http://130.61.66.27/todolist

## **STEP 4**: UnDeploy (optional)

  If you make changes to the image, you need to delete the service and the pods by running undeploy.sh then redo Steps 2 & 3.

1. Run the `undeploy.sh` script
	```
		<copy>cd $MTDRWORKSHOP_LOCATION/backend; ./undeploy.sh</copy>
	```
2. Rebuild the image + Deploy + (Re)Configure the API Gateway


## **STEP 5**: Configure the API Gateway

The API Gateway protects any RESTful service running on Container Engine for Kubernetes, Compute, or other endpoints through policy enforcement, metrics and logging.
Rather than exposing the Helidon service directly, we will use the API Gateway to define cross-origin resource sharing (CORS).

1. From the hamburger  menu navigate **Developer Services** > **API Management > Create Gateway**
   ![](images/API-Gateway-menu.png " ")

2. Configure the basic info: name, compartment, VCN and Subnet
    - VCN: pick on of the vitual circuit network
    - Subnet pick the public subnet   

	The click "Create".
  	![](images/Basic-gateway.png " ")

3. Click on Todolist gateway
    ![](images/Gateway.png " ")

4. Click on Deployments
   ![](images/Deployment-menu.png " ")

5. Create a todolist deployment
   ![](images/Deployment.png " ")

6. Configure Cross-origin resource sharing (CORS) policies.
	- CORS is a security mechanism that will prevent running application loaded from origin A  from using resources from another origin B.
	- Allowed Origins: is the list of all servers (origins) that are allowed to access the API deployment typically your Kubernetes cluster IP.
	- Allowed methods: GET, PUT, DELETE, POST, OPTIONS are all needed.

	![](images/Origins-Methods.png " ")

7. Configure the Headers
    ![](images/Headers.png " ")

8. Configure the routes: we will define two routes:
    - /todolist for the first two APIs: GET, POST and OPTIONS
    ![](images/Route-1.png " ")

    - /todolist/{id} for the remaining three APIs: (GET, PUT and DELETE)
	![](images/Route-2.png " ")


## **STEP 6**: Testing the backend application through the API Gateway

1. Navigate to the newly create Gateway Deployment Detail an copy the endpoint
   ![](images/Gateway-endpoint.png " ")

2. Testing through the API Gateway endpoint
  postfix the gateway endpoint with "/todolist" as shown in the image below
   ![](images/Backend-Testing.png " ")

  It should display the Todo Item(s) in the TodoItem table. At least the row you have created in Lab 1.

Congratulations, you have completed lab 2; you may now [proceed to the next lab](#next).

## Acknowledgements

* **Author** -  - Kuassi Mensah, Dir. Product Management, Java Database Access
* **Contributors** - Jean de Lavarene, Sr. Director of Development, JDBC/UCP
* **Last Updated By/Date** - Anoosha Pilli, Database Product Management,  April 2021
