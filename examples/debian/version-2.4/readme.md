We use the following stack in this example:

-   Ruby 2.6.3
-   PostgreSQL 11
-   NodeJS 11 & Yarn (for Webpacker-backed assets compilation)

## [`Dockerfile`](https://github.com/evilmartians/terraforming-rails/blob/master/examples/dockerdev/.dockerdev/Dockerfile)

`Dockerfile`  defines the _environment_  for our Ruby application: this is where we run servers, console (`rails c`), tests, Rake tasks, interact with our code in any way  _as developers_:

```
ARG RUBY_VERSION
# See explanation below
FROM ruby:$RUBY_VERSION-slim-buster

ARG PG_MAJOR
ARG NODE_MAJOR
ARG BUNDLER_VERSION
ARG YARN_VERSION

# Common dependencies
RUN apt-get update -qq \
  && DEBIAN_FRONTEND=noninteractive apt-get install -yq --no-install-recommends \
    build-essential \
    gnupg2 \
    curl \
    less \
    git \
  && apt-get clean \
  && rm -rf /var/cache/apt/archives/* \
  && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/* \
  && truncate -s 0 /var/log/*log

# Add PostgreSQL to sources list
RUN curl -sSL https://www.postgresql.org/media/keys/ACCC4CF8.asc | apt-key add - \
  && echo 'deb http://apt.postgresql.org/pub/repos/apt/ buster-pgdg main' $PG_MAJOR > /etc/apt/sources.list.d/pgdg.list

# Add NodeJS to sources list
RUN curl -sL https://deb.nodesource.com/setup_$NODE_MAJOR.x | bash -

# Add Yarn to the sources list
RUN curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | apt-key add - \
  && echo 'deb http://dl.yarnpkg.com/debian/ stable main' > /etc/apt/sources.list.d/yarn.list

# Application dependencies
# We use an external Aptfile for that, stay tuned
COPY .dockerdev/Aptfile /tmp/Aptfile
RUN apt-get update -qq && DEBIAN_FRONTEND=noninteractive apt-get -yq dist-upgrade && \
  DEBIAN_FRONTEND=noninteractive apt-get install -yq --no-install-recommends \
    libpq-dev \
    postgresql-client-$PG_MAJOR \
    nodejs \
    yarn=$YARN_VERSION-1 \
    $(cat /tmp/Aptfile | xargs) && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/* && \
    truncate -s 0 /var/log/*log

# Configure bundler
ENV LANG=C.UTF-8 \
  BUNDLE_JOBS=4 \
  BUNDLE_RETRY=3

# Uncomment this line if you store Bundler settings in the project's root
# ENV BUNDLE_APP_CONFIG=.bundle

# Uncomment this line if you want to run binstubs without prefixing with `bin/` or `bundle exec`
# ENV PATH /app/bin:$PATH

# Upgrade RubyGems and install required Bundler version
RUN gem update --system && \
    gem install bundler:$BUNDLER_VERSION

# Create a directory for the app code
RUN mkdir -p /app

WORKDIR /app

```

This configuration contains the essentials only and could be used as a starting point. Let me show what we are doing here.

The first two lines could look a bit strange:

```
ARG RUBY_VERSION
FROM ruby:$RUBY_VERSION-slim-buster

```

Why not just  `FROM ruby:2.6.3`, or whatever Ruby stable version du jour it is? We want to make our environment configurable from the outside using Dockerfile as a sort of a template:

-   the exact versions of runtime dependencies are specified in the `docker-compose.yml`  (see below);
-   the list of `apt`-installable dependencies is stored in a separate file (also see below).

We also specify the Debian release (`buster`) explicitly to make sure we add correct sources for other dependencies (such as PostgreSQL).

The following three lines define arguments for PostgreSQL, NodeJS, Yarn, and Bundler versions:

```
ARG PG_MAJOR
ARG NODE_MAJOR
ARG BUNDLER_VERSION
ARG YARN_VERSION

```

Since we do not expect anyone to use this Dockerfile without  [Docker Compose](https://docs.docker.com/compose/), we do not provide default values.

Then the actual image building is happening. First, we need to install some common system dependencies manually (Git, cURL, etc.), because we’re using the _slim_  base Docker image to reduce the size:

```
# Common dependencies
RUN apt-get update -qq \
  && DEBIAN_FRONTEND=noninteractive apt-get install -yq --no-install-recommends \
    build-essential \
    gnupg2 \
    curl \
    less \
    git \
  && apt-get clean \
  && rm -rf /var/cache/apt/archives/* \
  && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/* \
  && truncate -s 0 /var/log/*log

```

We explain all the details of installing system dependencies below, when talking about the application-specific ones.

Installing PostgreSQL, NodeJS, Yarn via  `apt`  requires adding their deb packages repos to the sources list.

For PostgreSQL (based in the [official documentation](https://www.postgresql.org/download/linux/debian/)):

```
RUN curl -sSL https://www.postgresql.org/media/keys/ACCC4CF8.asc | apt-key add - \
  && echo 'deb http://apt.postgresql.org/pub/repos/apt/ buster-pgdg main' $PG_MAJOR > /etc/apt/sources.list.d/pgdg.list

```

**NOTE**: that’s where we use the fact that our OS release is `buster`.

For NodeJS (from  [NodeSource repo](https://github.com/nodesource/distributions/blob/master/README.md#debinstall)):

```
RUN curl -sL https://deb.nodesource.com/setup_$NODE_MAJOR.x | bash -

```

For Yarn (from the [official website](https://yarnpkg.com/en/docs/install#debian-stable)):

```
RUN curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | apt-key add - \
  && echo 'deb http://dl.yarnpkg.com/debian/ stable main' > /etc/apt/sources.list.d/yarn.list

```

Now it’s time to install the dependencies, i.e. run  `apt-get install`:

```
COPY .dockerdev/Aptfile /tmp/Aptfile
RUN apt-get update -qq && DEBIAN_FRONTEND=noninteractive apt-get -yq dist-upgrade && \
  DEBIAN_FRONTEND=noninteractive apt-get install -yq --no-install-recommends \
    libpq-dev \
    postgresql-client-$PG_MAJOR \
    nodejs \
    yarn \
    $(cat /tmp/Aptfile | xargs) && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/* && \
    truncate -s 0 /var/log/*log

```

First, let’s talk about the Aptfile trick:

```
COPY .dockerdev/Aptfile /tmp/Aptfile
RUN apt-get install \
    $(cat /tmp/Aptfile | xargs)

```

I borrowed this idea from  [heroku-buildpack-apt](https://github.com/heroku/heroku-buildpack-apt), which allows installing additional packages on Heroku. If you’re using this buildpack, you can even re-use the same Aptfile for local and production environment (though the buildpack’s one provides more functionality).

Our  [default Aptfile](https://github.com/evilmartians/terraforming-rails/blob/master/examples/dockerdev/.dockerdev/Aptfile)  contains only a single package (we use Vim to edit Rails Credentials):

```
vim

```

In one of the previous project I worked on, we generated PDFs using LaTeX and [TexLive](https://www.tug.org/texlive/). Our Aptfile might look like this (those days I didn’t use this trick):

```
vim
texlive
texlive-latex-recommended
texlive-fonts-recommended
texlive-lang-cyrillic

```

This way, we keep the task-specific dependencies in a separate file, making our Dockerfile more universal.

With regards to `DEBIAN_FRONTEND=noninteractive`, I kindly ask you to take a look at [answer on Ask Ubuntu](https://askubuntu.com/a/972528).

The  `--no-install-recommends`  switch helps us to save some space (and make our image slimmer) by not installing recommended packages. See more  [here](http://xubuntugeek.blogspot.com/2012/06/save-disk-space-with-apt-get-option-no.html).

The last part of this  `RUN`  (`apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/* && truncate -s 0 /var/log/*log`) also serves the same purpose—clears out the local repository of retrieved package files (we installed everything, we don’t need them anymore) and all the temporary files and logs created during the installation. We need this cleanup to be in the same  `RUN`  statement to make sure this particular  [Docker layer](https://docs.docker.com/storage/storagedriver/#images-and-layers)  doesn’t contain any garbage.

The final part is mostly devoted to Bundler:

```
# Configure bundler
ENV LANG=C.UTF-8 \
  BUNDLE_JOBS=4 \
  BUNDLE_RETRY=3 \

# Uncomment this line if you store Bundler settings in the project's root
# ENV BUNDLE_APP_CONFIG=.bundle

# Uncomment this line if you want to run binstubs without prefixing with `bin/` or `bundle exec`
# ENV PATH /app/bin:$PATH

# Upgrade RubyGems and install required Bundler version
RUN gem update --system && \
    gem install bundler:$BUNDLER_VERSION

```

The  `LANG=C.UTF-8`  sets the default locale to UTF-8. Otherwise Ruby uses US-ASCII for strings and bye-bye those sweet sweet emojis 👋

Setting  `BUNDLE_APP_CONFIG`  is required if you use  `<root>/.bundle`  folder to store project-specicic Bundler settings (e.g., credentials for private gems). The default Ruby image  [defines this variable](https://github.com/docker-library/ruby/issues/129#issue-229195231)  making Bundler not to fallback to the local config.

You can optinally add your  `<root>/bin`  folder to the `PATH`  to run commands without  `bundle exec`. We do not do this by default,  
because it could break in multi-project environment (e.g., when you have  _local_  gems or engines in your Rails app).

## [`docker-compose.yml`](https://github.com/evilmartians/terraforming-rails/blob/master/examples/dockerdev/docker-compose.yml)

[Docker Compose](https://docs.docker.com/compose/)  is a tool to orchestrate our containerized environment. It allows us to link containers to each other, define persistent volumes and services.

Below is the compose file for a typical Rails application development with PostgreSQL as a database, and Sidekiq background job processor:

```
version: '2.4'

services:
  app: &app
    build:
      context: .
      dockerfile: ./.dockerdev/Dockerfile
      args:
        RUBY_VERSION: '2.6.3'
        PG_MAJOR: '11'
        NODE_MAJOR: '11'
        YARN_VERSION: '1.13.0'
        BUNDLER_VERSION: '2.0.2'
    image: example-dev:1.0.0
    tmpfs:
      - /tmp

  backend: &backend
    <<: *app
    stdin_open: true
    tty: true
    volumes:
      - .:/app:cached
      - rails_cache:/app/tmp/cache
      - bundle:/usr/local/bundle
      - node_modules:/app/node_modules
      - packs:/app/public/packs
      - .dockerdev/.psqlrc:/root/.psqlrc:ro
    environment:
      - NODE_ENV=development
      - RAILS_ENV=${RAILS_ENV:-development}
      - REDIS_URL=redis://redis:6379/
      - DATABASE_URL=postgres://postgres:postgres@postgres:5432
      - BOOTSNAP_CACHE_DIR=/usr/local/bundle/_bootsnap
      - WEBPACKER_DEV_SERVER_HOST=webpacker
      - WEB_CONCURRENCY=1
      - HISTFILE=/app/log/.bash_history
      - PSQL_HISTFILE=/app/log/.psql_history
      - EDITOR=vi
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy

  runner:
    <<: *backend
    command: /bin/bash
    ports:
      - '3000:3000'
      - '3002:3002'

  rails:
    <<: *backend
    command: bundle exec rails server -b 0.0.0.0
    ports:
      - '3000:3000'

  sidekiq:
    <<: *backend
    command: bundle exec sidekiq -C config/sidekiq.yml

  postgres:
    image: postgres:11.1
    volumes:
      - .psqlrc:/root/.psqlrc:ro
      - postgres:/var/lib/postgresql/data
      - ./log:/root/log:cached
    environment:
      - PSQL_HISTFILE=/root/log/.psql_history
    ports:
      - 5432
    healthcheck:
      test: pg_isready -U postgres -h 127.0.0.1
      interval: 5s

  redis:
    image: redis:3.2-alpine
    volumes:
      - redis:/data
    ports:
      - 6379
    healthcheck:
      test: redis-cli ping
      interval: 1s
      timeout: 3s
      retries: 30

  webpacker:
    <<: *app
    command: ./bin/webpack-dev-server
    ports:
      - '3035:3035'
    volumes:
      - .:/app:cached
      - bundle:/usr/local/bundle
      - node_modules:/app/node_modules
      - packs:/app/public/packs
    environment:
      - NODE_ENV=${NODE_ENV:-development}
      - RAILS_ENV=${RAILS_ENV:-development}
      - WEBPACKER_DEV_SERVER_HOST=0.0.0.0

volumes:
  postgres:
  redis:
  bundle:
  node_modules:
  rails_cache:
  packs:

```

We’re using Docker Compose version 2 intentionally: it suites development better. Read more in [this issue](https://github.com/docker/docker.github.io/pull/7593).

We define  **eight**  services. Why so many? Some of them only define shared configuration for others (_abstract_  services, e.g.,  `app`  and `backend`), others are used to specific commands using the application container (e.g.,  `runner`).

With this approach, we do not use  `docker-compose up`  command to run our application, but always specify the exact service we want to run (e.g.,  `docker-compose up rails`). That makes sense: in development, you rarely need all of the services up and running (Webpacker, Sidekiq, etc.).

Let’s take a thorough look at each service.

### `app`

The main purpose of this service is to provide all the required information to build our application container (the one defined in the `Dockerfile`  above):

```
build:
  context: .
  dockerfile: ./.dockerdev/Dockerfile
  args:
    RUBY_VERSION: '2.6.3'
    PG_MAJOR: '11'
    NODE_MAJOR: '11'
    YARN_VERSION: '1.13.0'
    BUNDLER_VERSION: '2.0.2'

```

The  `context`  directory defines the [build context](https://docs.docker.com/compose/compose-file/#context)  for Docker: this is something like a working directory for the build process, it’s used by the `COPY`  command, for example.

We explicitly specify the path to Dockerfile since we do not keep it in the project root, packing all Docker-related files inside a hidden  `.dockerdev`  directory.

And, as we mentioned earlier, we specify the exact version of dependencies using  `args`  declared in the Dockerfile.

One thing that we should pay attention to is the way we tag images:

```
image: example-dev:1.0.0

```

One of the benefits of using Docker for development is the ability to synchronize the configuration changes across the team automatically. You only need to upgrade the local image version every time you make changes to it (or to the arguments or files it relies on). The worst thing you can do is to use  `example-dev:latest`  as your build tag.

Keeping an image version also helps to work with two different environments without any additional hassle. For example, when you work on a long-running “chore/upgrade-to-ruby-3” branch, you can easily switch to `master`  and use the older image with the older Ruby, no need to rebuild anything.

> The worst thing you can do is to use  `latest`  tags for images in your  `docker-compose.yml`.

We also  _tell_  Docker to [use tmpfs](https://docs.docker.com/v17.09/engine/admin/volumes/tmpfs/#choosing-the-tmpfs-or-mount-flag)  for  `/tmp`  folder within a container to speed things up:

```
tmpfs:
  - /tmp

```

### `backend`

We reached the most interesting part of this post.

This service defines the shared behavior of all Ruby services.

Let’s talk about the volumes first:

```
volumes:
  - .:/app:cached
  - bundle:/usr/local/bundle
  - rails_cache:/app/tmp/cache
  - node_modules:/app/node_modules
  - packs:/app/public/packs
  - .dockerdev/.psqlrc:/root/.psqlrc:ro

```

The first item in the volumes list mounts the current working directory (the project’s root) to the `/app`  folder within a container using the `cached`  strategy. This  `cached`  modifier is the key to efficient Docker development on MacOS. We’re not going to dig deeper in this post (we’re working on a separate one on this subject 😉), but you can take a look at [the docs](https://docs.docker.com/docker-for-mac/osxfs-caching/).

The next line  _tells_  our container to use a volume named  `bundle`  to store  `/urs/local/bundle`  contents (this is where gems are stored  [by default](https://github.com/infosiftr/ruby/blob/9b1f77c11d663930f4175c683b1c5f268d4d8191/Dockerfile.template#L47)). This way we persist our gems data across runs: all the volumes defined in the `docker-compose.yml`  stay put until we run  `docker-compose down --volumes`.

The following three lines are also there to get rid of the “Docker is slow on Mac” curse. We put all the generated files into Docker volumes to avoid heavy disk operations on the host machine:

```
- rails_cache:/app/tmp/cache
- node_modules:/app/node_modules
- packs:/app/public/packs

```

> To make Docker fast enough on MacOS follow these two rules: use  `:cached`  to mount source files and use volumes for generated content (assets, bundle, etc.).

The last line adds a specific  `psql`  configuration to the container. We mostly need it to persist the commands history by storing it in the app’s  `log/.psql_history`  file. Why  `psql`  in the Ruby container? It’s used internally when you run  `rails dbconsole`.

Our  [`.psqlrc`](https://github.com/evilmartians/terraforming-rails/blob/master/examples/dockerdev/.dockerdev/.psqlrc)  file contains the following trick to make it possible to specify the path to the history file via the env variable (allow specifying the path to history file via  `PSQL_HISTFILE`  env variable, and fallback to the defaukt  `$HOME/.psql_history`  otherwise):

```
\set HISTFILE `[[ -z $PSQL_HISTFILE ]] && echo $HOME/.psql_history || echo $PSQL_HISTFILE`

```

Let’s talk about the environment variables:

```
environment:
  - NODE_ENV=${NODE_ENV:-development}
  - RAILS_ENV=${RAILS_ENV:-development}
  - REDIS_URL=redis://redis:6379/
  - DATABASE_URL=postgres://postgres:postgres@postgres:5432
  - WEBPACKER_DEV_SERVER_HOST=webpacker
  - BOOTSNAP_CACHE_DIR=/usr/local/bundle/_bootsnap
  - HISTFILE=/app/log/.bash_history
  - PSQL_HISTFILE=/app/log/.psql_history
  - EDITOR=vi
  - MALLOC_ARENA_MAX=2
  - WEB_CONCURRENCY=${WEB_CONCURRENCY:-1}

```

There are several things here, and I’d like to focus one.

First, the `X=${X:-smth}`  syntax. It could be translated as “For X variable within the container use the host machine X env variable value if present and another value otherwise”.  _Thus, we make it possible to run a service in a different environment provided along with the command, e.g.,  `RAILS_ENV=test docker-compose up rails`_.

The  `DATABASE_URL`,  `REDIS_URL`, and `WEBPACKER_DEV_SERVER_HOST`  variables  _connect_  our Ruby application to other services. The `DATABASE_URL`  and `WEBPACKER_DEV_SERVER_HOST`  variables are supported by Rails (ActiveRecord and Webpacker respectively) out-of-the-box. Some libraries support  `REDIS_URL`  as well (Sidekiq) but not all of them (for instance, Action Cable must be configured explicitly).

We use  [bootsnap](https://www.github.com/Shopify/bootsnap)  to speed up the application load time. We store its cache in the same volume as the Bundler data because this cache mostly contains the gems data; thus, we should drop everything altogether in case we do another Ruby version upgrade, for instance.

The  `HISTFILE=/app/log/.bash_history`  is the significant setting from the developer’s UX point of view: it tells Bash to store its history in the specified location, thus making it persistent.

The  `EDITOR=vi`  is used, for example, by  `rails credentials:edit`  command to manage credentials files.

Finally, the last two settings,  `MALLOC_ARENA_MAX`  and `WEB_CONCURRENCY`, are there to help you keep Rails memory handling in check.

The only lines in this service yet to cover are:

```
stdin_open: true
tty: true

```

They make this service  _interactive_, i.e., provide a TTY. We need it, for example, to run Rails console or Bash within a container.

It is the same as running a Docker container with the `-it`  options.

### `webpacker`

The only thing I want to mention here is the `WEBPACKER_DEV_SERVER_HOST=0.0.0.0`  setting: it makes Webpack dev server accessible from the _outside_  (by default it runs on `localhost`).

### `runner`

To explain what is this service for, let me share the way I use Docker for development:

-   I start a Docker daemon running a custom  `docker-start`  script:

```
#!/bin/sh

if ! $(docker info > /dev/null 2>&1); then
  echo "Opening Docker for Mac..."
  open -a /Applications/Docker.app
  while ! docker system info > /dev/null 2>&1; do sleep 1; done
  echo "Docker is ready to rock!"
else
  echo "Docker is up and running."
fi

```

-   Then I run  `dcr runner`  (`dcr`  is an alias for  `docker-compose run`) in the project directory to log into the container’s shell; this is an alias for:

```
$ docker-compose run --rm runner

```

-   I run (almost) everything from within this container: tests, migrations, Rake tasks, whatever.

As you can see, I do not spin a new container every time I need to run a task, and I’m always using the same one.

Thus, I’m using  `dcr runner`  the same way I used  `vagrant ssh`  years ago.

The only reason why it’s called  `runner`  and not  `shell`, for example, is that it also could be used to _run_  arbitrary commands within a container.

**Note**: The `runner`  service is a matter of taste, it doesn’t bring anything new comparing to the `web`  service, except from the default  `command`  (`/bin/bash`); thus,  `docker-compose run runner`  is exactly the same as  `docker-compose run web /bin/bash`  (but shorter 😉).

## Health checks

When running common Rails commands such as  `db:migrate`, we want to ensure that DB is up and ready to accept connections. How to tell Docker Compose to wait for a dependent service to be ready? We can use  [health checks](https://docs.docker.com/compose/compose-file/compose-file-v2/#healthcheck)!

You’ve probably noticed that our  `depends_on`  definition is not just a list of services:

```
backend:
  # ...
  depends_on:
    postgres:
      condition: service_healthy
    redis:
      condition: service_healthy

postgres:
  # ...
  healthcheck:
    test: pg_isready -U postgres -h 127.0.0.1
    interval: 5s

redis:
  # ...
  healthcheck:
    test: redis-cli ping
    interval: 1s
    timeout: 3s
    retries: 30

```

**NOTE:**  health checks are only supported by Docker Compose file format v2.1 and up; that’s why we use it for development.
