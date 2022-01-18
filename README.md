# Web-Scraping with Scrapy: Spiders and Utilities
This repository contains working code for web-scraping with scrapy and MongoDB (see below), it contains code for URL scraping, introductions to BeautifulSoup, and other assorted legacy and utility scripts. The scrapy code comes from the [command-line scrapy repo](https://github.com/URAP-charter/web_scraping) and was frozen on January 18th, 2022. That repo now contains more up-to-date, but less flexible code for distributed web-crawling with scrapy. For a universal crawling web-portal, please check out the [scrapy server repo](https://github.com/URAP-charter/scraping_server).

## Introduction

Our goal is a scalable, robust web-crawling pipeline (in Python) applicable across web designs and accessible for researchers with minimal computational skills. Our method involves using Scrapy spiders to recursively gather website links (to a given depth), collect and parse items (text, images, PDFs, and .docs) using BeautifulSoup and textract, and saving them locally and/or to MongoDB. 

The most recent web scraper is `scrapy/schools/schools/spiders/scrapy_vanilla.py`. See usage notes below.

Expect our spider and setup to continue being improved on, with better documentation and maybe even a web portal/user interface in the future. But if you're just trying to scrape website text, this should prove resilient as is.


## Usage notes
`scrapy_vanilla.py` uses scrapy's LinkExtractor to crawl recursively--meaning that it will crawl not just the a given input URL, but also those links it finds that start with that. For instance, if you feed it `site.com`, it will scrape `site.com/page`-- but not 'yelp.com' or other places outside the given root. 

URLs to scrape are loaded from `scrapy/schools/schools/spiders/test_urls.csv`. The simplest way to feed the spider new URLs is by updating this file. To test scalability, feel free to use the full list of 6K+ charter school URLs (gathered in 2018) in `scrapy/charter_schools_urls_2018.csv`. 

You'll get best results if you deploy the spider from within a scrapy project, like in our `scrapy/schools/` folder, because we've fine-tuned the `items.py`, `middlewares.py`, `settings.py`, and `pipelines.py` configurations (all under `scrapy/schools/schools/`)--and `scrapy_vanilla.py` draws on these.


## Running the web scraper

### Method 1: install and run locally

#### Install dependencies
Navigate to the `/web_scraping/scrapy/schools` directory.
Then run the following commands:
```bash
# Create and start a virtual environment for Python dependencies.
python3 -m venv .venv
source .venv/bin/activate
# Install dependencies.
pip3 install -r requirements.txt
```

#### Optional: Store crawling output to MongoDB 
If not yet installed, [install MongoDB](https://docs.mongodb.com/manual/installation/).

Ensure that this key-value pair of `ITEM_PIPELINES` in `/web_scraping/scrapy/schools/schools/settings.py` is NOT commented out:
```python
'schools.pipelines.MongoDBPipeline': 300
```
**If you DON'T want to use the MongoDB pipeline (e.g., you only want to store results locally), remove this element.**

Also make sure that in `/web_scraping/scrapy/schools/schools/settings.py`:
```python
# This next line must NOT be commented.
MONGO_URI = 'mongodb://localhost:27017' 
# This next line is commented.
# MONGO_URI = 'mongodb://mongodb_container:27017'

# This line also must NOT be commented
MONGO_DATABASE = 'schoolSpider'
```
Start MongoDB. This step depends on the operating system. On Ubuntu 18.04, this is:
```bash
sudo systemctl -l start mongodb
```
With Docker, you can start it with:
```bash
docker pull mongo
docker run -p 27017:27017 --name mongodb mongo
```
To stop MongoDB running in Docker:
```bash
docker stop mongodb
```

#### Run instructions
Finally, navigate to `/web_scraping/scrapy/schools/schools/` and run:

```bash
scrapy crawl schoolspider -a school_list=spiders/test_urls.csv -o schoolspider_output.json
```

This line means to run the schoolspider crawler with the given csv or tsv input file
and append the output to a json file. Note that subsequent runs of the crawler wsill append output, rather than replace. 
Appending to a json file is optional and the crawler can be run without doing this. Not specified in this command, is that data is saved behind the scenes
to a MongoDB database named "schoolSpider" if the MongoDB pipeline is used.

### Adding Logging
To add a log output file, you'll need to add '-L' and '--logfile' flags. The -L flag specifies the level of logging, such as 'INFO', 'DEBUG', or 'ERROR', and specifies what types of log messages appear in the log file. The --logfile flag specifies the output file location, such as './schoolspider\_log\_1\_1\_2021.log'

A sample command with a log file:

```bash
scrapy crawl schoolspider -a school_list=spiders/test_urls.csv -o schoolspider_output.json -L INFO --logfile ../logs/schoolspider_log_4_13_2021.log
```

When you're finished and you don't need to run the scraper anymore run:
```
# Deactivate the environment.
deactivate
```

### Method 2: install and run in a container

Firstly, [get Docker](https://docs.docker.com/get-docker/).

If you want to use MongoDB, ensure that in `/web_scraping/scrapy/schools/schools/settings.py`, that:
```python
# This next line is commented.
# MONGO_URI = 'mongodb://localhost' 
# This next line must NOT be commented.
MONGO_URI = 'mongodb://mongodb_container:27017'

Also, ensure that you uncomment the line:
MONGO_DATABASE = 'schoolSpider'
```
And, that:
```python
'schools.pipelines.MongoDBPipeline': 300
```
is one of the key-value pairs in of `ITEM_PIPELINES` in `/web_scraping/scrapy/schools/schools/settings.py` by uncommenting the line. If you don't want to use the pipeline, remove the element.

You will also need to set the "MONGO\_USERNAME" and "MONGO\_PASSWORD" properties in order to connect to MongoDB. 

Finally, inside `/web_scraping`, run:
```bash
# build the containers and run in the background
docker-compose up --build -d 
# to shutdown when finished
docker-compose down
```
No output json file is created through this method. Rather, the primary method of storing data is through MongoDB
(if it's enabled). Data inside MongoDB is persisted through a volume defined in `docker-compose.yml`.

The Mongo container can be easily accessed by "exec"ing into the container:
```bash
docker exec -it mongodb_container bash
```
Note: if running on a Windows machine, you will need to prefix that command with 'winpty'

From there, you can enter 'mongo' to start the Mongo CLI within the container, and it can be navigated the same as Mongo on your computer.


### Method 3: Run with a Flask API wrapper

NOTE that this implementation may have broken the other methods. This is still to be tested.

This method requires a Redis server to handle tasks. You can install Redis with instructions here (Ubuntu 18.04): https://www.digitalocean.com/community/tutorials/how-to-install-and-secure-redis-on-ubuntu-18-04

You will need at least 2 screens/terminals for this, although there may be a way to run Redis and/or Flask headless to remove this need. "Screen R" will be your screen for Redis, and "Screen F" will refer to your screen for Flask. 

1. In either screen, start your Mongo database. This can be done easily with Docker:
```bash
docker pull mongo && docker run --name mongodb -e MONGO_INITDB_ROOT_USERNAME=admin -e MONGO_INITDB_ROOT_PASSWORD=someSecretPassword -p 27017:27017 mongo
```
2. In both screens, start a virtual env and install your requirements (see Method 1)

3. In Screen R, start your Redis worker
```bash
rq worker crawling-tasks
```
4. In Screen F, start your Flask server from the scrapy/schools/schools/ directory (you can also run this from another directory, just swap "app.py" for the path to "app.py". This will run on Flask's default port 5000
```bash
python app.py
```
5. Requests for crawling tasks can be sent to the Flask server with a POST command. Currently only .csv files are accepted, but .tsv files will be available soon.
```bash
curl -X POST -d 'file=@path/to/your/csv/file.csv' localhost:5000/job
```
6. The returned object will have the status of the request (200 for ok, 400 for a bad input, such as a missing file), and the ID of the task being run. This ID can be queried from the Mongo database to check the status of the running task (TODO: status update when job is finished). These tasks are stored in the "task\_list" database in Mongo.

7. Gathered data can be found in 2 places. First, parsed text data and file/image urls are stored as objects in Mongo in the schoolSpider database. Second, raw files and images are stored locally, in "files/" and "images/" directories. TODO: modify the file and image pipelines so that they get stored to Mongo (which can handle raw files like that)

#### Simpler API instructions using `docker-compose`

1. Make sure you have Docker and Docker-Compose installed

2. Navigate to the directory with docker-compose.yml

3. Run "docker-compose up --build" (optional flags: " --force-recreate --renew-anon-volumes" if you have old docker volumes -- this can cause "authentication errors" due to Mongo not updating your credentials for the db)

4. Once it's running, run "curl -X POST -H 'Content-Type: multipart/form-data' -F 'file=@./scrapy/schools/schools/spiders/test_urls.csv' localhost:5000/job"

5. From there, you can check the status (work in progress -- it says it's done when it's still running): "curl localhost:5000/task?task_id=" + the "job_id" from the last CURL command

6. Or check Mongo: "docker exec -it mongodb_container bash" "mongo -u admin -p mdipass" "show dbs" "use schoolSpider" "db.otherItems.count()" "db.otherItems.find_one()" etc...
