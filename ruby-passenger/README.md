# Introduction

This container is designed to be a starting point for development and
deployment of ruby based web apps.

## Quick start (for simple apps without background processing)

1. Choose your ruby version (2.1, 2.2, 2.3, or 2.4)
2. Set your container to be `FROM` `instructure/ruby-passenger:<ruby version>`
3. Copy app and assets to `/usr/src/app` (nginx will serve static assets from public)
making sure to change the ownership of these files to docker:docker (`RUN chown -R docker:docker /usr/src/app`)
4. No need to set CMD/ENTRYPOINT, this base image already has it set
5. Build your new container and run it
6. View your app on port 80 (already exposed for you)

## App Env (RAILS_ENV)
RAILS_ENV and RACK_ENV default to production. To override this for
development and test, set the RAILS_ENV env var in your `Dockerfile` or
`docker-compose.yml` file, and that will also override RACK_ENV and the
other related vars in passenger.

## nginx worker process count (NGINX_WORKER_COUNT)
The default number of workers (1) in this container is well suited for use
in development environments, this will likely be insufficient in production
environments. To increase the number of workers set `NGINX_WORKER_COUNT` in
the environment before calling the entrypoint script.

## Upload Size Limits (NGINX_MAX_UPLOAD_SIZE)
The nginx default `client_max_body_size` of 1 MB is overly restrictive for
most applications, to combat this we've supplied a default of 10 MB with
the option to override via the `NGINX_MAX_UPLOAD_SIZE` variable. This must
be a valid string that the `client_max_body_size` directive will accept.

## Passenger max pool size (PASSENGER_MAX_POOL_SIZE)
The passenger default is 6. You may override this with the
`PASSENGER_MAX_POOL_SIZE` variable.

## Passenger min instances (PASSENGER_MIN_INSTANCES)
The passenger default is 0. We set a default of 1. You may override this with the
`PASSENGER_MIN_INSTANCES` variable.

## Passenger max request queue size (PASSENGER_MAX_REQUEST_QUEUE_SIZE)
The passenger default is 100. We set a default of 100. You may override this
with the `PASSENGER_MAX_REQUEST_QUEUE_SIZE` variable.

## Passenger startup timeout (PASSENGER_STARTUP_TIMEOUT)
The passenger default is 90. You may override this with the
`PASSENGER_STARTUP_TIMEOUT` variable.

## main.d (/usr/src/nginx/main.d/*.conf)
Additional global configuration settings can be included in the
`/usr/src/nginx/main.d/` directory.

### Environment Variables
One such global configuration that is included by default is an auto-whitelist
of environment variables passed into the container. This is because Nginx will
only pass environment variables that are explicitly whitelisted to Passenger.
If you wish to use an explicit whitelist instead, remove or replace the
`/usr/src/nginx/main.d/env.conf.erb` file in your derived image.

## conf.d (/usr/src/nginx/conf.d/*.conf;)
You may want to add some additional NGINX parameters. This can now
be done by adding a file to `/usr/src/nginx/conf.d/`
Example: web.conf

```
gzip on;
gzip_types text/css text/xml text/plain application/x-javascript application/atom+xml application/rss+xml;
```

## location.d (/usr/src/nginx/location.d/*.conf;)
You may want to add some additional NGINX location parameters. This can now
be done by adding a file to `/usr/src/nginx/location.d/`
Example: location.conf

```
try_files $uri $uri/ /index.html;
```
## SSL enforcement and X-Forwarded-For  (CG_ENVIRONMENT)
By default `CG_ENVIRONMENT` is set to `local`, which does NOT enable SSL or turn on `X-Forwarded-For`.
When deploying with CloudGate `CG_ENVIRONMENT` will be set and both SSL and `X-Forwarded-For` Will be enabled.

## application root path (APP_ROOT_PATH)
Occasionally you may need to change the root path. This currently defaults to
`/usr/src/app/public`. This can be overridden by setting the `APP_ROOT_PATH` variable.

## Sample Dockerfile

```Dockerfile
FROM instructure/ruby-passenger:2.4

ENV APP_HOME "/usr/src/app/"

USER root

COPY nginx/conf.d/* /usr/src/nginx/conf.d/
COPY nginx/location.d/* /usr/src/nginx/location.d/
COPY nginx/main.d/* /usr/src/nginx/main.d/

# Install PostgreSQL 9.6 libraries and client.
RUN echo "deb https://apt.postgresql.org/pub/repos/apt/ $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list \
 && curl -s https://www.postgresql.org/media/keys/ACCC4CF8.asc | apt-key add - \
 && apt-get update && apt-get install -y postgresql-client-9.6 \
 && apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

COPY Gemfile Gemfile.lock $APP_HOME
RUN chown -R docker:docker $APP_HOME

USER docker
RUN bundle install --quiet --jobs 8
USER root

COPY . $APP_HOME
RUN mkdir -p log tmp && chown -R docker:docker $APP_HOME

USER docker
```

# Making changes

All of the Dockerfiles in this directory are generated using a Rake task
(generate:ruby-passenger), this task also copies all of the source files
from the `template` directory. Make changes to any of these files, run the Rake
task and the updates will propagate to the sub folders for each version.
