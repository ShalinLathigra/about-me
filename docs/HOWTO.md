# How does the website work?

Okay, so figuring out the overall structure with a lot of help from AI. This document
covers how the heck this plan is going to pan out.

## Goal

My goal for this project is to have and deploy a website that I can use for:

- Workout tracking
- D&D notes
- Journaling, etc.

This is also meant to help familiarize myself with web development


## Architecture

User talks to Web browser

Browser talks via HTTP to NGINX

- Here it enters the docker network

NGINX routes to services

- Dashboard is all static HTML, directly retrieves files and images
- Health is a (i.e.) python thing which interacts with a database


Accessing the site goes something like:

http://tower.swallow-crested.ts.net/
http://tower.swallow-crested.ts.net/journal/
http://tower.swallow-crested.ts.net/health/
http://tower.swallow-crested.ts.net/dungeon/

etc.


## Reverse Proxy

Nginx is the entrypoint of the whole website. It is what actually does http connections
and handles the routing between services.

### Config

Nginx config basically boils down to setting up servers and declaring routes.

Main config block will be something that looks like:


```nginx

# Declare the server
server {
	listen 80; # Pick port to listen on
	server_name localhost; # What address the server is for

	# These next routes come in pairs
	# First route MISSING a trailing '/' will match anything that starts
	# with this section. i.e. this one will match: /test, /testing, /test/
	# and will re-route to /test/ using the 301
	location = /test { # location = XXX is an exact match!!! Not anything
		return 301 /test/; 
	}

	# This block handles the routes inside of /test/
	location /test/ {
		proxy_pass http://test:8080/; # These specific addrs are resolved by
			# Docker via internal DNS because they'll all be on the same network
	}
		# Extra note, at this point a route could be like: /test/page.html
		# When it gets past the proxy_pass here nginx will auto-strip the /test
		# so the end service gets /page.html

		# This means that each service doesn't need to worry about trimming
		# the leading section.

	location / {
		proxy_pass http://dashboard:8080
	}
}
```

There is a whole bunch of prox-_set_header and ssl_certificate+_key stuff
that needs to be done, but I really don't want to worry about this right now.

## Docker Networking

So this is what controls how the containers will all talk to one another

```yaml
networks:
	wiz-net:
		driver: bridge
```

This will be a "user-defined" bridge which allows us to use the container names as
aliases. Also means that it's only accessible to the attached containers.

This means that Nginx can use `http://<service>:8080` and be fine.
Need to expose the internal port in the compose:

```yaml
services:
	...
	journal:
		build: ./journal # Not sure what this one does
		expose:
			- "8080"
		networks:
			- wiz-net
		volumes:
			- journal-data:/app/data
		ports:
			- "8080:8080"
	...

	volumes:
		journal-data: # This declares that a special persistent volume exists
```

I could define multiple containers like this, and could run each service out of
an independent repository.

## URL Prefixes

If Journal App is talking with HTML, then it'll probably request things like:

<link rel="stylesheet" href="/static/style.css">

Want this to generally be specific to journal instead of going to the root level.
Need to test to make sure that this actually works, maybe using relative URLs but
I'd rather have a "project" level way to differentiate.

