# Pull our base image
FROM python:3.7-buster

# lets make sure we are up to date
RUN apt-get update -y && apt-get upgrade -y

# non-interactively add a new user
RUN useradd --create-home --shell /bin/bash app

# work exclusively inside of the users home directory
WORKDIR /home/app

# copy over application requirements
COPY src/requirements.txt requirements.txt

# update pip and install requirements
RUN pip install --upgrade pip && pip install -r requirements.txt && pip install gevent

# move over the rest of our source code.
COPY src .

# set all of the files to be owned by the app user and group
RUN chown -R app:app ./

# change to our app user
USER app

# expose port 8000 
EXPOSE 8000

# Define a sane state if we run the container without other configuration
CMD gunicorn --bind :8000 --worker-class gevent --timeout 5 --access-logfile - --error-logfile - --log-level info server:app
