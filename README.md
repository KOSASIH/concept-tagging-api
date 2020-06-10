# OCIO STI Concept Tagging Service

An API for exposing models created with [STI concept training](https://github.com/nasa/concept-tagging-training).  

## Index
1. [Using Endpoint](#using-endpoint)
    1. [Request](#request)
    2. [Response](#response)
2. [Running Your Own Instance](#running-your-own-instance)
    1. [Installation](#installation)
        1. [Pull Docker Image](#pull-docker-image)
        2. [Build Docker Image](#build-docker-image)
        3. [With Local Python](#with-local-python)
    2. [Download Models](#downloading-models)
    3. [Running Service](#running-service)
        1. [Using Docker](#using-docker)
        2. [Using Local Python](#using-local-python)

## Using Endpoint
### Request
The endpoint accepts a few fields, shown in this example:
```json
{
    "text": [
        "Astronauts go on space walks.",
        "Basalt rocks and minerals are on earth."
    ], 
    "probability_threshold":"0.5",
    "topic_threshold":"0.9", 
    "request_id":"example_id10"
}
```
- **text** *(string or list of strings)* -- The text(s) to be tagged.
- **probability_threshold** *(float in [0, 1])* -- a threshold under which a concept tag will not be returned by the API. For example, if the threshold is set to 0.8 and a concept only scores 0.5, the concept will be omitted from the response. Setting to 1 will yield no results. Setting to 0 will yield all of the classifiers and their scores, no matter how low.
- **topic_threshold** *(float in [0, 1])* -- A probability threshold for categories. If a category falls under this threshold, its respective suite of models will not be utilized for prediction. If you set this value to 1, only the generalized concept models will be used for tagging, yielding significant speed gains.
- **request_id** *(string)* -- an optional ID for your request.  

You might send this request using curl. In the command below:
1. Substitute `example_payload_multiple.json` with the path to your json request.
2. Substitute `http://0.0.0.0:5000/` with the address of the API instance.
```
curl -X POST -H "Content-Type: application/json" -d @example_payload_multiple.json http://0.0.0.0:5000/findterms/
```
### Response
You will then receive a response like that [here](docs/multiple_response.json). In the `payload`, you will see multiple fields, including:
- **features** -- words and phrases directly extracted from the document. 
- **sti_keywords** -- concepts and their prediction scores. 
- **topic_probability** -- model scores for all of the categories.

## Running Your Own Instance
### Installation
For most people, the simplest installation entails [pulling the docker image](#pull-docker-image), [downloading the models without bucket access](#without-bucket-access), [running the docker container](#using-docker).

#### Pull Docker Image
If you are on the NASA network, you should be able to pull a stable API image:
```bash
docker pull storage.analytics.nasa.gov/datasquad/concept_tagging_service:stable
```

#### Build Docker Image
First, clone this repository and enter its root.
Now, you can build the image with:
```
docker build -t concept_tagging_service:example .
```
\* Developers should look at the `make build` command in the [Makefile](Makefile). It has an automated process for tagging the image with useful metadata.

#### With Local Python
\* tested with python:3.7  
First, clone this repository and enter its root.  
Now, create a virtual environment. For example, using [venv](https://docs.python.org/3/library/venv.html):
```
python -m venv venv
source venv/bin/activate
```
Now install the requirements with:
```
make requirements
```


### Downloading Models
Then, you need to download the machine learning models upon which the service relies. 

#### With Bucket Access
If you have access to the `hq-ocio-ci-bigdata`, you should be able to do this one of the [Makefile](Makefile) commands. You can see the different models at `s3://hq-ocio-ci-bigdata/home/DataSquad/classifier_scripts/models/`. Let's say you want the `10_23_2019` models. You would then use:
```
make sync_models_from_s3 EXPERIMENT_NAME=10_23_2019 PROFILE=my_aws_profile
```
This command requires awscli. Substitute `my_aws_profile` with whatever aws profile has access to the bucket.
This downloads the models and places them in the appropriate directory to work with the tagging API.

#### Without Bucket Access
If you do not have access to the `hq-ocio-ci-bigdata`, then contact [anthony.r.buonomo@nasa.gov](mailto:anthony.r.buonomo@nasa.gov) or [brian.a.thomas@nasa.gov](mailto:brian.a.thomas@nasa.gov). You will likely be provided with a zipped file which contains all of the models. Now, to get the models in the right place and unzip:
```bash
mkdir models
mv <YOUR_ZIPPED_MODELS_NAME>.zip models
cd models
unzip <YOUR_ZIPPED_MODELS_NAME>.zip
```

### Running Service

#### Using Docker
With the docker image and model files in place, you can now run the service with a simple docker command. In the below command be sure to:
 1. Substitute `concept_tagging_service:example` for the name of your image.
 2. Substitute `$(pwd)/models/10_23_2019` to the path to your models directory. 
 3. Substitute `5001` with the port on your local machine from which you wish to access the API.
```
docker run -it \
    -p 5001:5000 \
    -v $(pwd)/models/10_23_2019:/home/service/models/experiment \
    concept_tagging_service:example
```

Note that you you may experience permission errors when you start the container. To resolve this issue, set the user and group of your `models` directory to 999. This is the uid for the user 

**optional**
The entrypoint to the docker image is [gunicorn](https://docs.gunicorn.org/en/stable/index.html), a python WSGI HTTP Server which runs our flask app. You can optionally pass additionally arguments to gunicorn. For example:
```bash
docker run -it \
    -p 5001:5000 \
    -v $(pwd)/models/10_23_2019:/home/service/models/experiment \
    concept_tagging_service:example --timeout 9000 
```
See [here](https://docs.gunicorn.org/en/stable/design.html#async-workers) for more information about design considerations for these gunicorn settings.


#### Using Local Python
With the requirements installed and the model files in place, you can now run the service with python locally. 
In the command below, substitute `models/test` with the path to your models directory. For example, if you followed the example from [With Bucket Access](#with-bucket-access), it will be `models/10_23_2019`.
```
export MODELS_DIR=models/test; \
python service/app.py
```