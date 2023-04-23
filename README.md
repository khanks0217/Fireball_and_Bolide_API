# **Fireball and Bolide Tracker**

## **Project Objective**
Sift through an abundance of positional and velocity data on fireball and bolide reportings documented by NASA. Build a Flask application for querying and returning information from the NASA data set.The included Dockerfile containerizes fireball_api.py to make it portable. The Fireball and Bolide Tracker application is hosted on the Kubernetes cluster and accessible to the outside world at a unique, public URL listed below.

The front-end REST API has an abundance of endpoints to post, get, and delete data from Redis in addition to endpoints for submitting a job to plot data and retrieve the results. The back end workers use queue functionality to watch  for jobs being submitted by the user. Aditionally, the workers generate a plot and adds the image back to Redis such that the user can download it at will. 

**Data Collected From:**
https://data.nasa.gov/api/views/mc52-syum/rows.xml?accessType=DOWNLOAD

**Fireball and Bolide Data Description**

The data collected from the above link is generated by U.S. Government sensors and ground-based observers. Note that this data is is not meant to be a complete list of all fireball events. Only the brightest fireballs are noted. 

What is a fireball or bolide? A fireball is an unusually bright meteor that reaches a visual magnitude of -3 or brighter when seen at the observer's zenith. Objects causing fireball events can exceed one meter in size. Fireballs that explode in the atmosphere are technically referred to as bolides although the terms fireballs and bolides are often used interchangeably. 


	Distance - [km]
	Velocity - [km/s]
	Radiated Energy - [J]
	Impact Energy - [kT]
	*Data Description adapted from https://catalog.data.gov/dataset/fireball-and-bolide-reports* 
<details>
<summary>Python Script Information</summary>
<br>
**fireball_api.py Description:**

The fireball_api.py script initializes a Flask instance, loads data from the NASA API linked above, and stores the data in Redis. The flask application tracks the altitude, location and velocity of the bolides which given in latitudes/longitudes and Cartesian vectors for both position.This data along with a time stamp, and calculated radiated and impact energy describe the complete state of the documented fireballs and bolides.

**jobs.py Description:**

The jobs.py script allows the user to queue up jobs through the Flask application initialized by the fireball_api.py script. It's through this queue that we're able
to instance many jobs at once, and allow them to run when a worker becomes free, or is available. This script allows for the creation of job instances, updating them, and retrieving past/current jobs that have finished or are currently runnning. When retrieving jobs, you get the specific job ID, start & finish time as well as its current status.

**worker.py Description:**

The workers.py script is what allows the jobs.py script to complete the job that was requested. Where the worker is, well the worker for the 'boss' or jobs.py script in this case. This script is what retrieves items from the queue list, and then updates the status of the job as it completes its task. The task being the analysis, or whatever was requested from the user via the jobs.py script. Where once the job is finished, this script then writes the analysis or task to the Redis databases that are shared between the three scripts.
</details>
#### **Dockerfile Description & Instructions**

How to pull and use the Dockerfule from Docker Hub:

	docker pull khanks0217/fireball_api:1.1

	docker pull khanks0217/fireball_api:1.wrk

How to build a new image from Dockerfile:

	Note: Be sure to include your username with the following commands.

	docker build -f docker/Dockerfile.api -t username/fireball_api:1.1

	docker build -f docker/Dockerfile.wrk -t username/fireball_api:1.wrk

#### **Instructions - How to use genes_api on Kubernetes**

Create a persistent volume claim (pvc)

	kubectl apply -f app-prod-db-pvc.yml

Deploy redis on the kubernetes cluster

	kubectl apply -f app-prod-db-deployment.yml


Deploy redis service on kubernetes cluster

	kubectl apply -f app-prod-db-service.yml

Add the redis IP address to the app-prod-api-deployment.yml

	kubectl get services

	Copy app-test-service IP address to the REDIS_IP value in the api deployment file.

Create a flask deployment:

	kubectl apply -f app-prod-api-deployment.yml

Create a flask service:

	kubectl apply -f app-prod-api-service.yml

Create a worker deployment:

	kubectl apply -f app-prod-wrk-deployment.yml

Create a NodePort:

	kubectl apply -f app-prod-api-nodeport.yml

Copy the nodeport port to app-prod-api-ingress.yml

	kubectl get services

	NAME                         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
	flasktest-service-nodeport   NodePort    10.233.15.48   <none>        5000:31587/TCP   45s

	Edit app-prod-api-ingress.yml 
	host: "username.coe332.tacc.cloud" <-- This should be your username
	...
	port:
	  number: 31587   <--This should be the same as the port from NodePort 5000:/XXXXX

Create the Ingress object:

	kubectl apply -f app-prod-api-ingress.yml

	kubectl get ingress

	NAME                CLASS    HOSTS                       ADDRESS   PORTS   AGE
	flasktest-ingress   <none>   username.coe332.tacc.cloud             80      102s

We can test by running the following curl command from anywhere, including our laptops.

	curl -X POST username.coe332.tacc.cloud/data

##### **Fireball API Front End:**

The API front end is expose on port 5000 inside the container. Try the following routes:

$ curl -X [POST, GET, DELETE] username.coe332.tacc.cloud/

| Route         | Method        | Return |
| ------------- |:-------------:| ------------- |
| `/data`     | GET | Return all data in Redis database | 
| 	      | DELETE |  Delete data from Redis database | 
| 	      | POST | Post data into Redis database | 
| `/timestamp`    | GET |  Returns all available timestamps for each fireball recorded |
| `/timestamp/<string: pr_date>`  | GET |  Return all data for a specific timestamp |
| `/timestamp/<string: pr_date>/speed`  | GET |  Returns the speed of the fireball for a specific timestamp |
| `/timestamp/<string: pr_date>/energy`  | GET |  Returns the energy for a specific timestamp |
| `/timestamp/<string: pr_date>/location`  | GET |  Return geographical position for a specific timestamp |
| `/jobs`     | GET | List all of the jobs in the Redis database |
|             | POST | Create a new job |
| `/jobs/<string: job_id>`     | GET | Get status of a specific job by id. |
| `/jobs/<string: job_id>/results`     | GET | Return the outputs of a completed job. |
| `/help`  | GET |  Returns text that describes each route & what they do |
|`/graph_energy`    | GET | Returns energy plot file from Redis_energy database|
| 	     | DELETE |  Deletes plot from Redis_image database | 
| 	     | POST | Posts plot into Redis_image database | 
|`/graph_speed`    | GET | Returns speed plot file from Redis_speed database|
|            | DELETE |  Deletes plot from Redis_image database |
|            | POST | Posts plot into Redis_image database |

To download graphs to local computer:

	[local] $ curl -X GET khanks.coe332.tacc.cloud/graph_energy --output energy_graph.jpg

	[local] $ curl -X GET khanks.coe332.tacc.cloud/graph_speed --output speed_graph.jpg

###### **Running fireball_api.py**
	
**Expected Output, Sample**

curl -X POST khanks.coe332.tacc.cloud/data
```
Fireball and Bolide data loaded into Redis.	
```

curl -X GET khanks.coe332.tacc.cloud/data
```
{
    "_address": "https://data.nasa.gov/resource/mc52-syum/row-5m8i_9fxe~2q47",
    "_id": "row-5m8i_9fxe~2q47",
    "_position": "0",
    "_uuid": "00000000-0000-0000-2D6B-D1ADF7A5E88A",
    "impact_energy": "0.12",
    "latitude": "3.2N",
    "longitude": "45.4W",
    "peak_brightness": "2014-08-28T03:07:45",
    "radiated_energy": "34000000000"
  },
  {
    "_address": "https://data.nasa.gov/resource/mc52-syum/row-43ir-khtx~jjqx",
    "_id": "row-43ir-khtx~jjqx",
    "_position": "0",
    "_uuid": "00000000-0000-0000-682E-439BE7C0D4F8",
    "impact_energy": "0.12",
    "latitude": "22.2N",
    "longitude": "132.9W",
    "peak_brightness": "2014-10-21T18:55:37",
    "radiated_energy": "34000000000"
  }

	...continued
```

curl -X DELETE khanks.coe332.tacc.cloud/data
```
Fireball and Bolide data DELETED from Redis.
```

curl khanks.coe332.tacc.cloud/timestamp
```
[
  "2012-08-27T06:57:43",
  "2014-05-08T19:42:37",
  "2012-09-11T22:07:30",
  "2015-04-30T10:21:01",
  "2015-03-08T04:26:28",
  "2012-07-27T04:19:50",
  "2014-11-28T11:47:18",
  "2014-02-13T06:47:42",
  "2014-07-29T03:07:43",
  "2012-09-10T01:03:32"
  (continued)
  ]
```

curl khanks.coe332.tacc.cloud/timestamp/2012-07-25T07:48:20
```
{
  "_address": "https://data.nasa.gov/resource/mc52-syum/row-atif-qn38_syje",
  "_id": "row-atif-qn38_syje",
  "_position": "0",
  "_uuid": "00000000-0000-0000-5740-8F57B1576ED5",
  "altitude": "26.8",
  "impact_energy": "0.39",
  "latitude": "36.4N",
  "longitude": "41.5E",
  "peak_brightness": "2012-07-25T07:48:20",
  "radiated_energy": "133000000000",
  "velocity_magnitude": "343.19999999999993",
  "x_velocity": "0.8",
  "y_velocity": "2",
  "z_velocity": "-18.4"
}
```

curl khanks.coe332.tacc.cloud/timestamp/2012-07-25T07:48:20/speed
```
{
  "velocity_magnitude": "18.525657883055057 [km/s]",
  "x_velocity": "0.8 [km/s]",
  "y_velocity": "2 [km/s]",
  "z_velocity": "-18.4 [km/s]"
}
```

curl khanks.coe332.tacc.cloud/timestamp/2012-07-25T07:48:20/energy
```
{
  "calculated_impact_energy": "0.39 [kT]",
  "radiated_energy": "133000000000 [J]"
}
```
```
curl khanks.coe332.tacc.cloud/timestamp/2012-07-25T07:48:20/location

{
  "altitude": "26.8",
  "county": "",
  "district": null,
  "geo_country": "Iraq",
  "latitude": 36.4,
  "longitude": 41.5,
  "pos_unit": "km",
  "region": "",
  "state": "Nineveh Governorate",
  "x_velocity": "0.8 [km/s]",
  "y_velocity": "2 [km/s]",
  "z_velocity": "-18.4 [km/s]"
}
```

curl -X POST khanks.coe332.tacc.cloud/graph
```
Image has been posted.
```

curl -X GET khanks.coe332.tacc.cloud/graph --output graph.jpg
```
% Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 19335  100 19335    0     0   248k      0 --:--:-- --:--:-- --:--:--  248k
```
```
curl -X POST khanks.coe332.tacc.cloud/jobs

{
  "end": 2023,
  "id": "4f345b4a-41f9-43f8-a88c-1a978540d7b7",
  "start": 2023,
  "status": "submitted"
}
```

curl -X GET khanks.coe332.tacc.cloud/jobs
```
[
  {
    "end": "2023",
    "id": "4f345b4a-41f9-43f8-a88c-1a978540d7b7",
    "start": "2023",
    "status": "submitted"
  },
  {
    "end": "2023",
    "id": "d265cece-97e2-4518-95d4-f0d22ef0f93c",
    "start": "2023",
    "status": "complete"
  },
  {
    "end": "2023",
    "id": "1ae08d3c-dc4a-419f-9c15-3cd228c89d39",
    "start": "2023",
    "status": "complete"
  },
  {
    "end": "2023",
    "id": "9434fa6e-021b-4a97-8f74-c602291cd73c",
    "start": "2023",
    "status": "complete"
  },
  {
    "end": "2023",
    "id": "f84a5c8f-6607-4e8d-85b0-da7308051a6c",
    "start": "2023",
    "status": "complete"
  }
]
```
curl khanks.coe332.tacc.cloud/jobs/d265cece-97e2-4518-95d4-f0d22ef0f93c
```
complete
```
curl khanks.coe332.tacc.cloud/help
```
Available routs and methods: 
/data [POST,GET,DELETE,HEAD,OPTIONS]

    POST - Post all fireball and bolide data to Redis.

    GET - Return all fireball and bolide data from redis to the user.

    DELETE - Delete all fireball and bolide data from Redis.

/timestamp [HEAD,OPTIONS,GET]

    Description:
    API endpoint that returns a list of peak brightness dates for all objects in the database.

    Returns:
    JSON object containing a list of peak brightness dates for all objects in the database.
    

/timestamp/<string:pb_date> [HEAD,OPTIONS,GET]

    Description:
    API endpoint that returns all data for a given timestamp.

    Returns:
    JSON object containing the data for a specific timestamp in the database.
    

/timestamp/<string:pb_date>/speed [HEAD,OPTIONS,GET]

    Description:
    API endpoint that returns the velocity values for a specific timestamp in the database.

    Returns:
    JSON object containing a dictionary of velocity data. 
    

/timestamp/<string:pb_date>/energy [HEAD,OPTIONS,GET]

    Description:
    API endpoint that returns the velocity values for a specific timestamp in the database.

    Returns:
    JSON object containing a dictionary of energy data for a specific timestamp.
    

/timestamp/<string:pb_date>/location [HEAD,OPTIONS,GET]

    Route that returns latitude, longitude, altitude, and geoposition for a given <epoch>.

    Returns:
    Dictionary containing latitude, longitude, altitude, and geoposition.
    

/help [HEAD,OPTIONS,GET]

    Description:
    This function is an API endpoint that returns information about all available routes and HTTP methods in the application.
    
    Returns:
    A string containing a list of all available routes and their associated HTTP methods, as well as the docstrings for each endpoint function.
    

/graph_energy [POST,GET,DELETE,HEAD,OPTIONS]

    POST - Write energy graph data to Redis.
    GET - Return energy graph to user in form of jpg file.
    DELETE - Delete energy graph data from redis. 
    

/graph_speed [POST,GET,DELETE,HEAD,OPTIONS]

    POST -  Post speed graph data to Redis.
    GET - Return speed graph to user.
    DELETE - Delete speed graph data from Redis. 
    

/jobs [POST,HEAD,OPTIONS,GET]

    POST - API route for creating a new job to do some analysis. This route accepts a JSON payload
    describing the job to be created.

    GET - API route to return jobs to user
    

/jobs/<string:this_job_id> [HEAD,OPTIONS,GET]

    Description:
    API endpoint to return status of a job specified by job_id to user. 

    Returns:
    String describing status of specified job.
```
