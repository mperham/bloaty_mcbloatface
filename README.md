# Bloaty McBloatface

This Rails app tries to reproduce the memory bloat that seems to be
edemic with 64-bit Linux deployments.

### Setup

* Install Docker/OSX.
* Run `docker-compose up`
* Run `docker-compose exec bloat bundle exec rails db:seed`
* Run `ab -k -c 25 -n 1000 http://localhost:3000/posts`
