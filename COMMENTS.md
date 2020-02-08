# COMMENTS
------------

## Design Decisions

#### Using nginx provided nginx image
The `nginx:latest` image is setup well to support our configuration, and so I have chosen it as the image to use. We provide quite a bit of configuration inside of `nginx/nginx.conf` so mounting `nginx/nginx.conf` to `/etc/nginx/nginx.conf`, and our ssl `nginx/files/` to `/etc/nginx/ssl/` allows us to achive the expected behavior while maximizing simplicity in our setup.

Note: the current `nginx.conf` file relies on the upstream service `app` being exposed to networking so that nginx can get the information, without this the configuration file fails to load. The production nginx configuration will have to take this into account.

#### `python:buster` vs `python:alpine`
A large number of apps out there like to roll their images using alpines base, and this tends to be fine for applications built in compiled languages like GO where binaries have everything they need. In python we're presented with some unique challenges as the differences between glibc and musl cause issues with the precompiled PyPI binary wheel. This requires that source code be pulled so the appropriate version can be built increasing space used and build time

Using buster we have a download size of approx 60mb, and a final size on disk after building of 208mb

### docker-compose

#### Flask
Flask's excellent support of environment variables allows us to create a dockerfile that has sane default values in the event other values are provided, while also allowing us to provide a development environment that supports hotloading with the correct variables provided to `FLASK_ENV`, `FLASK_APP`, `FLASK_RUN_HOST`, `FLASK_RUN_PORT`.

#### Dockerfile Builds
Since we built our app container and our `docker-compose.yml` to allow overrides for development values to provide a rapid method of delpoyment for working on changes, it makes sense to build the images locally when using docker-compose. To that end a build section is included in `docker-compose.yml` for the app service. This will build the image locally from the contents of the repositroy, and mount the correct paths to allow edits to be reflected in the container while it is running.

#### Validation of behavior (Feedback on validate.sh)
When using the provided validator on a ubuntu linux machine, I noticed that I had correctly implemented the requested header proxying, however 2 of the tests were failing. 

    ------------------------------------------------------------------------
    | Testing HTTP header and body content...                              |
    ------------------------------------------------------------------------
    Pass: status code is 200
    Fail: X-Forwarded-For should not be 'None'
    Fail: X-Real-IP should not be 'None'
    Pass: X-Forwarded-Proto is present and not 'None'
    Pass: found "It's easier to ask forgiveness than it is to get permission." in response

while the output was clearly including this information

    # curl -Lk http://127.0.0.1
    It's easier to ask forgiveness than it is to get permission.
    X-Forwarded-For: 172.18.0.1
    X-Real-IP: 172.18.0.1
    X-Forwarded-Proto: https

    # curl -Lk http://159.65.76.175
    It's easier to ask forgiveness than it is to get permission.
    X-Forwarded-For: 159.65.76.175
    X-Real-IP: 159.65.76.175
    X-Forwarded-Proto: https

Through a little debugging of the validator I found that the included regex doesn't work on ubuntu, but it does work on mac. 

*Mac* using the regex in validate.sh `X-Forwarded-For: \b\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}\b`

    echo -e "It's easier to ask forgiveness than it is to get permission.\nX-Forwarded-For: 172.18.0.1\nX-Real-IP: 172.18.0.1\nX-Forwarded-Proto: https"|grep -E "X-Forwarded-For: \b\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}\b"
    X-Forwarded-For: 172.18.0.1

*Ubuntu* using the regex in validate.sh `X-Forwarded-For: \b\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}\b`

    # echo -e "It's easier to ask forgiveness than it is to get permission.\nX-Forwarded-For: 172.18.0.1\nX-Real-IP: 172.18.0.1\nX-Forwarded-Proto: https"|grep -E "X-Forwarded-For: \b\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}\b"
    # echo $?
    1


A slight change to the regex will result in the values being matched correctly on both mac and ubuntu.

`X-Forwarded-For: ([0-9]{1,3}[\.]){3}[0-9]{1,3}`


    # echo -e "It's easier to ask forgiveness than it is to get permission.\nX-Forwarded-For: 172.18.0.1\nX-Real-IP: 172.18.0.1\nX-Forwarded-Proto: https"|grep -E "X-Forwarded-For: ([0-9]{1,3}[\.]){3}[0-9]{1,3}"
    X-Forwarded-For: 172.18.0.1
    # echo $?
    0

## Usage

#### docker-compose
As long as docker-compose and docker are present on the host system, then the `docker-compose.yml` contains all of the required information to build the app docker container using `app/Dockerfile`. 

To start the containerized development version using docker-compose run 

    docker-compose up

in the `app` services section of `docker-compose.yml` we tell docker-compose to build the container

      app:
        build:
          context: ./app
          dockerfile: Dockerfile

#### manual docker build and run
The Dockerfile for our app container can be found at `app/Dockerfile`. To build this image from `app/` run

    docker build -t app_container -f Dockerile .

The container can be run without any additional configuration with something along the lines of 

    docker run -d -p 0.0.0.0:8000:8000 app_container

You can watch the logs with 

    docker logs --follow $(docker ps | grep app_container | awk '{print $1}')

where `app_container` is the tag you gave the container you built from the `app/Dockerfile` dockerfile.


## Development

The environment is set to development when using `docker-compose up` and the containers `/home/app` folder is mounted to `app/src` allowing changes to be moved across and the reload to be triggered. NOTE: This is using a development server that is not meant for production, this is only for testing development changes.
