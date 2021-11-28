# Test Doubles for REST APIs

Where OpenAPI meets KarateDSL Server Side Features for REST API mocking:

- Powerful yet simple **stateful mocks using KarateDSL**.
- Request/response validation from your OpenAPI schemas.
- **Declarative stateless mocks** from your OpenAPI examples with `'x-apimock-when'`.
- Flexible and powerful built-in and custom defined **dynamic data generators** for your OpenAPI examples with `x-apimock-transform`.
- Use openapi examples to **populate your karate mocks** initial data with `x-apimock-karate-var` and `x-apimock-seed`.

Checkout [KarateIDE vscode extension](https://github.com/ivangsa/karate-ide) for a powerful testing and mocking user experience for KarateDSL.

<!-- TOC -->

- [Test Doubles for REST APIs](#test-doubles-for-rest-apis)
    - [Features](#features)
    - [How it works](#how-it-works)
    - [Configuration](#configuration)
        - [Creating *Routable* OpenAPI examples](#creating-routable-openapi-examples)
        - [Populate karate mocks initial data from OpenAPI examples](#populate-karate-mocks-initial-data-from-openapi-examples)
        - [Generating random data store from base OpenAPI examples](#generating-random-data-store-from-base-openapi-examples)
    - [Usage](#usage)
        - [Build from source](#build-from-source)
        - [Maven dependency](#maven-dependency)
        - [Command line](#command-line)
        - [Mock your REST dependencies in JUnit tests](#mock-your-rest-dependencies-in-junit-tests)

<!-- /TOC -->
## Features

This is basically a wrapper around KarateDSL mock server with some useful features from OpenAPI examples.

OpenAPI definitions are used to **validate** request/responses and to serve **dynamic OpenAPI examples**:

- Request/response validation thanks to [org.openapi4j:openapi-operation-validator](https://www.openapi4j.org/operation-validator.html)
- Choose OpenAPI examples with `'x-apimock-when'`: you can use any valid javascript and karate expression that returns a boolean.
- Built-in Data Generators inline in your examples: `uuid()`, `now('dd/MM/yyyy')`, `date('dd/MM/yyy', +/-n[d|h])`, `sequenceNext()`
- Use `'x-apimock-transform'` when generator tags are not compatible with your OpenAPI linters.
- Custom Data Generators: define your own data generators using *Javascript* or even *Java* right inside you karate features. They will be available also for OpenAPI examples 
- Populates karate features data sets from OpenAPI `'#/components/examples/'`.
- Seed (multiply) mock data from a few OpenAPI examples.

With KarateDSL you can:

- Simple yet powerful stateful mocks. See official [KarateDSL Mocks](https://github.com/karatelabs/karate/tree/master/karate-netty) for details.
- Delegating validations to OpenAPI makes karate mocks even simpler.
- Reusing OpenAPI examples as mock data sets. Compatible with any favorite graphical OpenAPI editing tool, makes it easy to **edit** and **lint** your mocking data sets.

## How it works

- All this is just a wrapper and custom hooks around **KarateDSL MockServer** listening for requests and serving responses.
- Before requests are passed into Karate features, a custom hook validates request payload against provided OpenAPI definition file, responding with a `400 status code` error in case it fails validation.
- If request passes validation, then it is routed to KarateDSL server side features.
  - If Karate matches one scenario it produces a response.
  - If no Karate scenario matches for your request, then a custom hook will search for examples in OpenAPI definition file and will filter them using `x-apimock-when`. This when clause uses the same functions and functionality as KarateDSL scenario names selection. If no example matches this **when clause** the it will send a 404 status code.
  - OpenAPI examples are processed for **interpolated function tags** using `{{ }}`. For instance `{{ uuid() }}` will be replaced with the output of `uuid()` generator. You can configure any custom generator defining them in the Background section of you Karate mock features.
- Last, all responses are validated against OpenAPI schema and will send a 400 in case validation fails.

## Configuration

### Creating *Routable* OpenAPI examples

Karate mocks allows you to create stateful mocks with little effort, but you can also create dynamic stateless mocks using just OpenAPI response examples.

You can use just OpenAPI definition to create complex dynamic stateless mocks.

- Use `x-apimock-when` to select which response example you want to serve. You can use any valid Javascript (ES2021) expression that returns a boolean.
- Use `{{ }}` to interpolate generator tags. There are some built-in generators (uuid, now, date, sequenceNext), but you can also add any custom generator using the Background section of your karate mocks.
- Use `x-apimock-transform` when generator tags are not compatible with your OpenAPI linters.

You can use any valid Javascript (ES2021) expression with all karate variables (request, pathParams, requestParams, requestHeader,...) and helpers (paramExists(), paramValue(), headerContains(), bodyPath(),...) available for you.

See [karate documentation]((https://github.com/karatelabs/karate/tree/master/karate-netty#request) for more details .

Just see this example to grasp an idea of what you can do:

```yaml
responses:
'200':
    description: successful operation
    content:
    application/json:
        schema:
        "$ref": "#/components/schemas/Pet"
        examples:
          pet-1:
            summary: Dynamic Pet
            x-apimock-when: pathParams.petId > 0
            x-apimock-transform:
                - id: pathParams.petId
                - category.id: uuid()
                - $[*].creationDate: 'date("dd/MM/yyy", "-1d")'
            value:
                id: 0
                name: 'DOG {{Math.random()}}'
                category:
                    id: 0
                    name: DOG
                tags:
                    - id: 0
                    name: 'name'
                status: sold
```

### Populate karate mocks initial data from OpenAPI examples

You can populate karate mocks intitial data leveraging **#/components/examples/<example>** using `x-apimock-karate-var`, `x-apimock-seed`, and `x-apimock-transform` to generate dynamic data right from your OpenAPI definition.

```yaml
  examples:
    karate-pets:
      summary: Pets for karate mocks dataset
      x-apimock-karate-var: pets
      x-apimock-seed: 20
      x-apimock-transform:
        $[*].id: sequenceNext()
        $[*].status: "Math.random() >= 0.5? 'sold' : 'available'"
      value:
      - id: 0
        name: 'Dog Name {{Math.random()}}'
        category:
          id: 1
          name: DOG
        status: sold
      - id: 0
        name: 'Cat Name {{Math.random()}}'
        category:
          id: 2
          name: CAT
        tags:
          - id: 0
            name: 'Cat'
        status: sold
```

This helps keeping your karate mock features to a minimum logic.

```gherkin
@mock
Feature: PetMock Mock

Background: 
* configure cors = true
* configure responseHeaders = { 'Content-Type': 'application/json' }

# this array will be populated directly from openapi.yml#/components/examples/pets
* def pets = []

@getPetById
Scenario: methodIs('get') && pathMatches('/pet/{petId}')
* def response = pets.find(pet => pet.id == pathParams.petId)
* def responseStatus = response? 200 : 404

@addPet
Scenario: methodIs('post') && pathMatches('/pet')
* def pet = request
* pet.id = sequenceNext()
* pets.push(pet)
* def response = pet
* def responseStatus = 200
```

### Complete Stateful PetStore CRUD Example

Checkout this complete CRUD Example

- [OpenAPI definition with examples data](https://github.com/ivangsa/apimocks/blob/main/src/test/resources/petstore/petstore-openapi.yml#L360)
- [Complete Karate PetMock](https://github.com/ivangsa/apimocks/blob/main/src/test/resources/petstore/mocks/PetMock/PetMock.feature)
- [JUnit CRUD test using an OpenAPI client](https://github.com/ivangsa/apimocks/blob/main/src/test/java/io/github/apimock/RestClientExampleTest.java)


## Usage

### Build from source

```shell
git clone https://github.com/ivangsa/apimock.git
cd apimock
mvn clean install
```

### Maven dependency

```xml
<dependency>
  <groupId>io.github.apimock</groupId>
  <artifactId>apimock</artifactId>
  <version>0.0.1</version>
</dependency>
```

### Command line

```shell
java -cp "apimock.jar;karate-1.2.0.jar" io.github.apimock.Main -o openapi.yml -m Mock.feature -p 3000 -P context/path -W
```

If you see an error like this, make sure you have added karate fat jar dependency in your classpath, make sure it's [karate fat jar](https://github.com/karatelabs/karate/releases).

```log
Exception in thread "main" java.lang.NoClassDefFoundError: picocli/CommandLine
        at io.github.apimock.Main.main(Main.java:46)
Caused by: java.lang.ClassNotFoundException: picocli.CommandLine
```

### Mock your REST integrations/dependencies in JUnit tests

In JUnit tests, you can mock your REST services integrations starting a MockServer on a random port and pointing clients to this local implementation:

```java
public class RestClientExampleTest {

    io.github.apimock.MockServer server;

    @Before
    public void setup() throws Exception {
        server = MockServer.builder()
                .openapi("classpath:petstore/petstore-openapi.yml")
                .features("classpath:petstore/mocks/PetMock/PetMock.feature")
                .pathPrefix("api/v3")
                .http(0).build();
    }

    @Test
    public void testRestWithMockServer() {
        PetApi petApiClient = new PetApi();
        petApiClient.getApiClient().setBasePath("http://localhost:" + server.getPort() + "/api/v3");
        PetDto pet = petApiClient.getPetById(1L);
        Assert.assertNotNull(pet);
    }

    @After
    public void tearDown() throws Exception {
        server.stop();
    }

}
```
