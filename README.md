## Documentation for the CAFE project.

A user accessible verion of the site is not online yet, however the up to date code is hosted under this organization on github.

The two major code components of the CAFE project are the django based server component and the Angular web client.

The web client is hosted in the [cafe-app](https://github.com/cafe-trauma/cafe-app) repository.  It is compiled down to small number of javascript files which can be statically hosted somewhere.

The server client is hosted in the [cafe-server](https://github.com/cafe-trauma/cafe-server) repository.  It is a django app designed to run on python3.

The installation instructions for both can be found in the respective repositories.  The server component will also attempt to connect to an RDF triple store over the SPARQL REST endpoints.  To enable HTTP communication between all of the various components we currently use [nginx](https://nginx.org/en/) as a proxy server.  These are the relevant aliases and proxy_passes we currently use for the development server.

```
location /static {
    alias /location/of/cafe-server/static;
}

location /graphs {
    alias /location/of/cafe-server/graphs;
}

location /api/ {
    proxy_pass http://localhost:8000/;
}

location /admin/ {
    proxy_pass http://localhost:8000/admin/;
}

location /rdf {
    proxy_pass http://localhost:8080/rdf4j-server/repositories/cafe;
}
```
