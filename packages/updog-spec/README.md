# UPDOG Specification

**U**nified **P**WA **D**efinitions **O**f **G**raphs are simple files describing how a web server delivers a [Progressive Web Application][pwa def]. They specify server behavior in a platform-independent way, so that a PWA client application expecting a certain backend can be deployed on any type of tech stack that implements the UPDOG specification.

## Quickstart

This repository is a test suite for UPDOG compliance, testing several scenarios and features on a live web server. It requires NodeJS v8 LTS or later. To test an UPDOG server:

1. Install the `npx` utility to run global NPM commands:

    ```sh
    npm install -g npx
    ```

2. Write or obtain a POSIX shell script which:

   - gets the path to an `updog.yaml` file from the environment variable `UPDOG_YAML_PATH`
   - launches and/or binds the UPDOG server under test and runs it in the foreground (not as a daemon process).
   - prints the hostname and port of the now-running server instance to standard out
   - responds to SIGTERM by gracefully closing the server

   [Example here.][spec-shell-script]

3. Use `npx` to run `updog-spec` on your shell script

    ```sh
    npx updog-spec ./test_my_updog.sh
    ```

4. The shell script will run for each test suite with the environment variable `UPDOG_YAML` set to the path of a fixture `updog.yaml` file for configuring a server instance. The script should launch a server (on a local port or a remote port, but resolvable to the local system) and print its host to standard out, staying in the foreground.

    The tests run in parallel. When each test suite is over, the script will receive a SIGTERM or SIGKILL.

5. The test suite will print results to stdout; the argument `--junit` will make it print JUnit-compatible test result XML to stdout.

## Summary

UPDOG files are XML, YAML or JSON files which declare the behavior of an [application shell] server. An application shell server implements a strict subset of HTTP functionality: it handles an HTTP GET request for a resource and delivers enough code and data to bootstrap a Progressive Web App which displays that resource.

An App Shell is purposefully minimal, and so is an UPDOG server. It is meant to initialize or refresh sessions, deliver small HTML documents with enough server-side rendering for initial display and SEO, and then hand off subsequent request handling to the PWA running in the client.

The declarative format of UPDOG means that an UPDOG-compliant server may be written in any programming language and run on any tech stack; therefore, a PWA can declare its own runtime network dependency by including an UPDOG file.

### Example `updog.yaml` which echoes request data

```yaml
root:
  add:
    status: 200
    headers:
      content-type: text/plain
    body:
      render:
        engine: mustache
        template: |
          {{#request}}
            Method: {{method}}
            HTTP Version: {{httpVersion}}
            Headers:
              {{#headerEntries}}
                {{key}}: {{value}}
              {{/headerEntries}}
            URL:
              {{#url}}
                protocol: {{protocol}}
                host: {{host}}
                hostname: {{hostname}}
                port: {{port}}
                pathname: {{pathname}}
              {{/url}}
            URL Query:
              {{#url.queryEntries}}
                {{key}}: {{value}}
              {{/url.queryEntries}}
          {{/request}}
```

This trivial example describes a server which always returns status 200 with a single header, `content-type`, and a text body which is a plaintext summary of the GET request properties.

### The configuration file

A spec-compliant UPDOG server should be launchable with a single runtime parameter: the location of the `updog.yaml` file. The file may reference external resources using [Source Specifiers](#sourcespecifier), which should always be resolved to filesystem paths relative to the `updog.yaml` file. A simple, short template can be provided inline, but most of the time, a template should be specified with a Source Specifier.

### The root responder

**An `updog.yaml` file must contain a property `respond` at the top level.** (Other properties are allowed but ignored, to permit later expansion of the specification, and to allow for [YAML anchors][yaml anchors].).

The `respond` value must be an array of objects, often just a single object that represents the root of a decision tree. Each object is a [`ResponseLayer`](#responselayer) and the array of them is a [`ResponsePlan`](#responseplan). **The root `respond` must always eventually produce a context object with `status`, `headers`, and `body` properties.** Once the context satisfies this contract, it is coerced into an HTTP response and flushed to the client. No streaming or buffering interface should be provided; UPDOG servers should not deal in data of significant size.

### ResponsePlan

A `ResponsePlan` is an ordered list set of `ResponseLayers` which build a context object that can be used as an HTTP response. ResponseLayers can have child ResponsePlans, so a ResponsePlan can be thought of as a node in a tree. Each ResponseLayer can add properties to the `context`, which is global for each request.

```yaml
# A ResponsePlan that adds constant values to the context,
# with one conditional.
# At the end of execution, the context will match the object:
# {
#   foo: 'bar',
#   status: 403,
#   monkey: ['see']
# }
#

 - when:
    - match:
        status: '^4' # All values are coerced to strings for regex.
      respond:
        - add:
          monkey:
           - see
    - respond: # The default case has no `match` property.
        - add:
          monkey:
           - do
 - add:
    status: 403
    body: ook ook ook
```

**ResponsePlans execute their ResponseLayers as concurrently as possible**. The maximum concurrency is left to the implementation. A compliant server detects when a ResponseLayer uses a context value, and delay its execution until that context value becomes available. To manually design serial workflows, you must use `after` rules in ResponsePlans to delay their execution until a given context value is available.

For instance, in the above plan, the `when` ResponseLayer uses the `status` context value in its `match` configuration. It should not run until the `status` value has been assigned, so it runs second.

All paths through such a tree must result in context objects with `status`, `headers`, and `body` properties. The expected behavior when a `ResponsePlan` ends without producing such a context is to immediately return a generic 500 error; the UPDOG server should never time out.

### ResponseLayer

A `ResponseLayer` is an object which specifies how to get or create a value in the response. It has different properties depending on the type of operation it runs. These properties are mutually exclusive. The following operations must be supported:

#### Add

The `add` operation specifies an object which will be merged into the context. It can overwrite values (though there should never be a reason to). The merge is a "shallow" merge. Values can be declared literals or specified through [Source Specifiers](#sourcespecifiers).

#### Call

The `call` operation specifies a GraphQL call to be placed. It requires an endpoint. Variables can be extracted from context; queries should be separate files in most cases, but can be literals.

#### When

The `when` operation does branch logic via pattern matching.

### A more complex example

```yaml

# Store an anchor to a reusable 404 config.
common:
  - &notfound
    - add:
        status: 404
        body: '<html><body><h1>404: Not Found</h1></body></html>'

root:
  when:
    - match:
        url:
          path: '^/.*\.html'
      respond:
        - call: env.M2_GRAPHQL_REST_PROXY
          query: |
            query {
              storeInfo {
                name
              }
            }
        - call: env.M2_GRAPHQL_ENDPOINT
          query: |
            query resolveRoute($path: String!) {
              urlResolver(url: $path) {
                type
                id
              }
            }
          variables:
            path: url.path
        - after:
            - urlResolver
          alias: model
          when:
          - match:
              urlResolver:
                type: PRODUCT
            respond:
              - call: env.M2_GRAPHQL_ENDPOINT
                query:
                  file: ./productDetail.graphql
                variables:
                  sku: urlResolver.id
              - after:
                - products
                add:
                  template:
                    file: './productDetail.mst'
          - match:
              urlResolver:
                type: CATEGORY
            respond:
              - call: env.M2_GRAPHQL_ENDPOINT
                query:
                  file: ./category.graphql
                variables:
                  id: urlResolver.id
              - after:
                - category
                add:
                  template:
                    file: './category.mst'
          - match:
              urlResolver:
                type: CMS
            respond:
              - call: env.M2_GRAPHQL_ENDPOINT
                query:
                  file: ./cmsPage.graphql
                variables:
                  id: urlResolver.id
              - after:
                - ./page
                add:
                  template:
                    file: ./cmsPage.mst
          - respond: *notfound
        - after:
            - model
            - storeInfo
          add:
            status: 200
            body:
              render:
                engine: mustache
                template: template
    - respond: *notfound
```

## Background

Progressive Web Apps represent a new plateau of maturity for the Web as a platform. Like any other high-quality software, PWAs benefit from [Twelve-factor methodology][twelve factor], but they have historically been considered a _part_ of a full-scale deployed application, the "frontend", and not a full application to which all twelve-factor principles should apply.

Until PWAs, "web apps" have not been apps on their own; they could not function without a specific set of backing services. Those services should be decoupled and abstracted as [resource locators][twelve factor resources], so that individual servers can be swapped in and out, but the "frontend" is still tightly coupled to a specific topology of services, culminating in end-to-end functionality that depends on all of those services being present.

![12-factor resource diagram: 12factor.net](http://localhost:8080/backing_services.png)
_Image courtesy of 12factor.net_

A PWA must run independently from any backing services as much as possible; therefore, it must have alternate strategies to replace the functionality of each backing service. The simplest and most efficient PWA would have as few individual backing services as possible, so a need emerges for a tool which unifies backing services into a single layer that deliver, supports, and synchronizes with a PWA.

![Unified graph over backing services](http://localhost:8080/unified_graph.png)

### The Problem With PWA Runtime Dependencies

A PWA must have at least partial functionality offline; for example, an eCommerce store PWA should be able to navigate its catalog in several dimensions, including categories and configurable products, without placing a server call for each transition. Therefore, at least some of its business logic must be implemented in the HTML/CSS/JS layer traditionally designated as "frontend".

However, duplication of logic is an antipattern, and the spreading of business logic across multiple layers is an antipattern that harms testability. The [hexagonal architecture][hexagonal architecture] describes how business logic must be expressible through multiple "ports and adapters", so that it can be tested in multiple contexts. This seems incompatible with an app architecture that supports "offline mode", where workflows should be independent of the availability of services. It implies that a Web app should implement state management logic in the frontend, or as close to the frontend as possible.

### Edge Definition, Not Service Orchestration

A common solution to the problem of business logic that exists across multiple services is [service orchestration][service orchestration], but a service orchestrator is a separate program with foreknowledge of the workflows that exist, and an imperative programming paradigm that enforces the workflows. UPDOG more closely resembles [service choreography][service choreography]

![12-factor resource diagram: 12factor.net](http://localhost:8080/unified_graph.png)

### Common PWA Backing Service Characteristics

PWA best practices recommend a small subset of the possible things a web server can do. This set of restrictions makes it much easier to describe web server behavior with a limited set of definitions.

A PWA should:

- Serve over [HTTPS only][pwa https]
- Serve an [application shell] with some aspects common to all pages
- Serve static resources from [edge servers][cdn def] whenever possible
- Provide a common strategy for data exchange, such as REST or GraphQL, to avoid excess client responsibilities

A PWA should not:

- Require many resources that cannot be [cached and reused when offline][offline mode]
- Depend on server-side state for [most interactions and workflows][high perf loading]

It follows that a server which delivers PWA resources can count on the following to be true:

- Requests are idempotent
- Responses are small
- All services are GraphQL

These assumptions enable a manageably small declarative specification for an app shell server.

### Objections

Common objections to creating new specifications and standards include:

- **Necessity**: Unclear what problem the standard solves
- **Proliferation**: Too many standards exist already
- **Restrictiveness**: The standard imposes restrictions that hamper desired functionality
- **Erosion risk**: Ensuring compliance with a standard's updates adds to the cost of software development and maintenance

The UPDOG team shares those concerns, and applies them continuously to the standard as it develops. UPDOG exists because it answers these concerns with clear necessity, wise restrictiveness and limited scope.

[application shell]: <https://developers.google.com/web/fundamentals/architecture/app-shell>
[twelve factor]: <https://12factor.net/>
[twelve factor resources]: <https://12factor.net/backing-services>
[hexagonal architecture]: <http://alistair.cockburn.us/Hexagonal+architecture>
[pwa def]: <https://developers.google.com/web/progressive-web-apps/>
[pwa https]: <https://developers.google.com/web/fundamentals/security/encrypt-in-transit/why-https>
[cdn def]: <https://en.wikipedia.org/wiki/Content_delivery_network>
[offline mode]: <https://developers.google.com/web/fundamentals/instant-and-offline/offline-cookbook/>
[high perf loading]: <https://developers.google.com/web/fundamentals/primers/service-workers/high-performance-loading>
[fetch]: <https://developers.google.com/web/ilt/pwa/working-with-the-fetch-api>
[service choreography]: <https://en.wikipedia.org/wiki/Service_choreography#Web_Service_Choreography>
[service orchestration]: <https://en.wikipedia.org/wiki/Orchestration_(computing)>
[service choreography]: <https://en.wikipedia.org/wiki/Service_choreography>
[npx]: <https://github.com/zkat/npx>
[spec-shell-script]: <./test_my_updog.sh>
[yaml anchors]: <https://learnxinyminutes.com/docs/yaml/>
