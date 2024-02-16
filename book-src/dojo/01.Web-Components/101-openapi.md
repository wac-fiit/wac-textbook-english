# API Specification Proposal

---

>info:>
Template for a pre-built container ([Details here](../99.Problems-Resolutions/01.development-containers.md)):
`registry-1.docker.io/milung/wac-ufe-101`

---

In the previous section, you must have noticed that our editor always appears empty, and changes cannot be saved. While we could edit records directly in memory, it would be more practical to start working with data that we will acquire and save using a [RESTful Web API][REST API]. Currently, we don't have any service providing the necessary API. We have two options on how to proceed - create a service providing the required REST API and then implement this API in our single-page application, or define how the API should look using the [OpenAPI] specification and then generate the necessary implementation using [OpenAPI tools][openapi-generator]. The advantage of the second approach is that we can continue developing our application, define the API based on real client requirements, and automate the generation of service implementation on both the client and server sides. This technique is generally referred to as [_API First Design_][api-first]. For a better understanding of the principle, we will gradually create this API and incrementally modify our application to use the resulting API.

>info:> To work with [openapi] files, we recommend installing the [openapi-lint](https://marketplace.visualstudio.com/items?itemName=mermade.openapi-lint) and [openapi-designer](https://marketplace.visualstudio.com/items?itemName=philosowaffle.openapi-designer) extensions in Visual Studio Code.

1. Create a file `${WAC_ROOT}/ambulance-ufe/api/ambulance-wl.openapi.yaml`. Insert the following code into it:

```yaml
openapi: 3.0.0
servers:
  - description: Cluster Endpoint
    url: /api @_important_@
info:
  description: Ambulance Waiting List management for Web-In-Cloud system
  version: "1.0.0"
  title: Waiting List Api
  contact:
    email: <your_email>
  license:
    name: CC BY 4.0
    url: "https://creativecommons.org/licenses/by/4.0/"
tags:
- name: ambulanceWaitingList  @_important_@
  description: Ambulance Waiting List API
```

In this step, we defined the basic structure of the [openapi] file. In the `servers` section, we specified where our service will be available - this value can be changed later in the implementation, and the one mentioned here will be used as the default value. In the `info` section, we defined basic information about our service. In the `tags` section, we defined a list of tags that we will use to categorize individual endpoints. Tags are important for code generation; typically, a separate class containing all paths and methods assigned to a specific tag is generated for each tag.

2. Next, we will add the specification for the path `/waiting-list/{ambulance-id}` to the file:

```yaml
paths:
"/waiting-list/{ambulanceId}/entries":
  get:
    tags:
      - ambulanceWaitingList @_important_@
    summary: Provides the ambulance waiting list
    operationId: getWaitingListEntries  @_important_@
    description: By using ambulanceId you get list of entries in ambulance waiting list
    parameters:
      - in: path @_important_@
        name: ambulanceId @_important_@
        description: pass the id of the particular ambulance
        required: true
        schema:
          type: string
    responses:
      "200":
        description: value of the waiting list entries
        content:
          application/json:
            schema:
              type: array
              items:
                $ref: "#/components/schemas/WaitingListEntry" @_important_@
            examples:
              response:
                $ref: "#/components/examples/WaitingListEntriesExample" @_important_@
      "404":
        description: Ambulance with such ID does not exist
```

This specification states that on the path `/waiting-list/{ambulanceId}/entries`, where `{ambulanceId}` is a variable of type string, we can make an [HTTP GET](https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods/GET) request, and the response can have a status of [200](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/200) or [404](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/404). In the first case, it will contain an array of objects of type `WaitingListEntry`. We also specified the operation name as `getWaitingListEntries`. This name determines the names of methods and functions during code generation. The `operationId` must be unique within the specification. Note that we used references to the `components` section. Now, let's insert the [JSON schema][jsonschema] for the `WaitingListEntry` object and the objects on which it depends:


```yaml
components:
schemas:
  WaitingListEntry: @_important_@
    type: object
    required: [id, patientId, waitingSince, estimatedDurationMinutes]  @_important_@
    properties:
      id:
        type: string
        example: x321ab3
        description: Unique id of the entry in this waiting list
      name:
        type: string
        example: Jožko Púčik
        description: Name of patient in waiting list
      patientId:
        type: string
        example: 460527-jozef-pucik
        description: Unique identifier of the patient known to Web-In-Cloud system
      waitingSince:
        type: string
        format: date-time
        example: "2038-12-24T10:05:00Z"
        description: Timestamp since when the patient entered the waiting list
      estimatedStart:
        type: string
        format: date-time
        example: "2038-12-24T10:35:00Z"
        description: Estimated time of entering ambulance. Ignored on post.
      estimatedDurationMinutes:
        type: integer
        format: int32
        example: 15
        description: >-
          Estimated duration of ambulance visit. If not provided then it will
          be computed based on condition and ambulance settings
      condition:
        $ref: "#/components/schemas/Condition" @_important_@
    example: 
      $ref: "#/components/examples/WaitingListEntryExample" @_important_@
  Condition: @_important_@
    description: "Describes disease, symptoms, or other reasons of patient   visit"
    required:
      - value  @_important_@
    properties:
      value:
        type: string
        example: Teploty
      code:
        type: string
        example: subfebrilia
      reference:
        type: string
        format: url
        example: "https://zdravoteka.sk/priznaky/zvysena-telesna-teplota/"
        description: Link to encyclopedical explanation of the patient's condition
      typicalDurationMinutes:
        type: integer
        format: int32
        example: 20
    example: 
      $ref: "#/components/examples/ConditionExample" @_important_@
```

In this specification, we defined the `WaitingListEntry` object, which contains all the necessary information about the waiting list of patients, and the description of the patient's health problem is defined by the `Condition` type. Technically, we could define the schema of the nested `Condition` type directly within the `WaitingListEntry` object. However, for code generation purposes and clarity, it is recommended to always use references to separately defined types. An important aspect of the specification is indicating required fields using the `required` keyword. This keyword is crucial for generating code that will validate input data. If the input data does not contain required fields, the request will typically be rejected with a [400 - Bad Request](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/400) status code.

Consistently, in the specification, we provide examples using the `example` keyword. These examples are essential for generating documentation and are also crucial for creating an experimental service (mock) that we will use during local development. Add the following examples to the `components.examples` section of the file:


```yaml
components: 
...
  examples:
    WaitingListEntryExample: 
      summary: Ľudomír Zlostný waiting
      description: |
        Entry represents a patient waiting in the ambulance prep room with
        defined symptoms
      value:
        id: x321ab3
        name: Ľudomír Zlostný
        patientId: 74895-ludomir-zlostny
        waitingSince: "2038-12-24T10:05:00.000Z"
        estimatedStart: "2038-12-24T10:35:00.000Z"
        estimatedDurationMinutes: 15
        condition:
          value: Nevoľnosť
          code: nausea
          reference: "https://zdravoteka.sk/priznaky/nevolnost/"
    ConditionExample:
      summary: Conditions and symptoms
      description: list of few symptoms that can be chosen by patients
      value: 
        valuee: Teploty
        code: subfebrilia
        reference: >-
          https://zdravoteka.sk/priznaky/zvysena-telesna-teplota/
    WaitingListEntriesExample:
      summary: List of waiting patients 
      description: |
        Example waiting list containing 2 patients
      value:
      - id: x321ab3
        name: Jožko Púčik
        patientId: 460527-jozef-pucik
        waitingSince: "2038-12-24T10:05:00.000Z"
        estimatedStart: "2038-12-24T10:35:00.000Z"
        estimatedDurationMinutes: 15
        condition:
          value: Teploty
          code: subfebrilia
          reference: "https://zdravoteka.sk/priznaky/zvysena-telesna-teplota/"
      - id: x321ab4
        name: Ferdinand Trety
        patientId: 780907-ferdinand-tre
        waitingSince: "2038-12-24T10:25:00.000Z"
        estimatedStart: "2038-12-24T10:50:00.000Z"
        estimatedDurationMinutes: 25
        condition:
          value: Nevoľnosť
          code: nausea
          reference: "https://zdravoteka.sk/priznaky/nevolnost/"
```

Our specification now describes how we can retrieve the list of patients waiting in the ambulance. Let's proceed with the integration into our application.

>info:> Our `getWaitingListEntries` API directly returns an array of records, which is considered a design flaw in the case of WebAPI. Our API should be prepared for a larger data scope and support fetching only a [partial range](https://learn.microsoft.com/en-us/azure/architecture/best-practices/api-design#filter-and-paginate-data). For simplicity, we won't address this aspect in our application, but when designing a specific API in practice, familiarize yourself with the [principles of designing RESTful APIs](https://book.restfulnode.com/).

3. In the first step, we will set up an experimental mock server providing the specified API based on the examples in the specification. Install the necessary dependencies for the application development by executing the following command in the `${WAC_ROOT}/ambulance-ufe` directory:


```ps
npm install --save-dev js-yaml open-api-mocker npm-run-all 
```

In the file `${WAC_ROOT}/ambulance-ufe/package.json`, add the new scripts and modify the `start` script:
  
```json
...
"scripts": {
  "convert-openapi": "js-yaml  ./api/ambulance-wl.openapi.yaml > .openapi.json", @_add_@
  "mock-api": "open-api-mocker --schema .openapi.json --port 5000", @_add_@
  "start:app": "stencil build --dev --watch --serve", @_add_@
  "start:mock": "run-s convert-openapi mock-api", @_add_@
  "start": "run-p -r start:mock start:app", @_add_@
  "build": "stencil build --docs",
  "start": "stencil build --dev --watch --serve", @_remove_@
  ...
```

>$apple:> On some Mac devices, an AirPlay server may run on port 5000. In such a case, change the port of the mock server to another available port.

The `convert-openapi` script converts the YAML-format specification to JSON - `open-api-mocker` requires the specification in JSON format. The `mock-api` script starts a mock server that will provide the API according to the specification. The `start:mock` script sequentially executes both previous commands. The `start:app` script contains the command originally used in the `start` script, which compiles our application and starts the development server. We modified the `start` script to concurrently start our mock API server and the development server for our application. These modifications allow us to develop our application locally without connecting to a real API server.

Modify the `${WAC_ROOT}/ambulance-ufe/.gitignore` file and add the line `.openapi.json`.


```text
...
.openapi.json @_add_@
```

In the `${WAC_ROOT}/ambulance-ufe` directory, execute the following command:

```ps
npm run start:mock
```

Open a new command line and enter the following command:

```ps
curl http://localhost:5000/api/waiting-list/bobulova/entries
```

The output will display a JSON-format listing of waiting patients in the ambulance. This output is generated based on the example in the specification. Now, we have a service capable of simulating our REST API. Stop the running mock API (press _CTRL+C_).

4. In the next step, we will generate client code using the [openapi-generator]. Install this tool into the project:

```ps
npm install --save-dev @openapitools/openapi-generator-cli
```

Create a file `${WAC_ROOT}/ambulance-ufe/openapitools.json` and insert the following content:

```json
{
  "$schema": "./node_modules/@openapitools/openapi-generator-cli/config.schema.json",
  "generator-cli": {
      "useDocker": true,
      "version": "6.6.0",
      "generators": {
          "ambulance-wl": {
              "generatorName": "typescript-axios", @_important_@
              "glob": "api/ambulance-wl.openapi.yaml", @_important_@
              "output": "#{cwd}/src/api/ambulance-wl", @_important_@
              "additionalProperties": {
                  "supportsES6": "true",
                  "withInterfaces": true,
                  "enablePostProcessFile": true
              }
          }
      }
  }
}
```

This file configures the behavior of the code generator. We will use the Docker version of the generator, so it is necessary to have an active Docker daemon when generating code, for example, [Docker for Desktop][docker-desktop]. An alternative solution requires having the JAVA SDK installed. Among other parameters, it specifies the path to our specification - `glob` - and also the path - `output` - where the `typescript-axios` client will be generated. Next, create a file `${WAC_ROOT}/ambulance-ufe/src/api/ambulance-wl/.openapi-generator-ignore` and insert the following content:

```text
.npmignore
*.sh
```

This file specifies which files will not be saved to the target directory during code generation. In our case, these are auxiliary files `.npmignore` and `git-push.sh` that we don't need.

Add a new script to the file `${WAC_ROOT}/ambulance-ufe/package.json`:

```json
...
"scripts": {
  ...
  "openapi": "openapi-generator-cli generate" @_add_@
},
...
```

Execute the following command in the `${WAC_ROOT}/ambulance-ufe` directory:

```ps
npm run openapi
```

In the directory `${WAC_ROOT}/ambulance-ufe/src/api/ambulance-wl`, you will now find new code that contains the implementation of a client for our specified API in the [TypeScript] programming language using the [Axios] library.

>info:> In our case, we generate client code as part of our application. However, client code is often generated as libraries so that the API can be used across different applications, especially if our API is generally applicable. In such cases, we would create a separate project whose content is generated from the specification, such as from the URL link to this specification, and publish this project to [npmjs.com]. With suitable automation, we would be able to automatically generate various libraries in different languages for the same specification.

&nbsp;

>info:> It would be appropriate to modify the `build` script as well, so that each time before compiling the application, client code is generated. We leave this modification for your independent work.

5. We will use the generated API in our application. First, install the [Axios] library used by the generated client code:

```ps
npm install --save axios@1.6.0
```

Open the file `${WAC_ROOT}/ambulance-ufe/src/components/<pfx>-ambulance-wl-list/<pfx>-ambulance-wl-list.tsx` and modify the code:

```tsx
import { Component, Event, EventEmitter, Host, Prop, State, h } from '@stencil/core';  @_important_@
import { AmbulanceWaitingListApiFactory, WaitingListEntry } from '../../api/ambulance-wl'; @_add_@
...
export class <Pfx>AmbulanceWlList { 

  @Event({ eventName: "entry-clicked" }) entryClicked: EventEmitter<string> 
  @Prop() apiBase: string; @_add_@
  @Prop() ambulanceId: string; @_add_@
  @State() errorMessage: string; @_add_@

  waitingPatients: WaitingListEntry[]; @_important_@

  private async getWaitingPatientsAsync(): Promise<WaitingListEntry[]> {   @_important_@
    ... odstránte pôvodný kód ... @_remove_@
    // be prepared for connectivitiy issues
    try { @_add_@
      const response = await @_add_@
        AmbulanceWaitingListApiFactory(undefined, this.apiBase). @_add_@
          getWaitingListEntries(this.ambulanceId) @_add_@
      if (response.status < 299) { @_add_@
        return response.data; @_add_@
      } else { @_add_@
        this.errorMessage = `Cannot retrieve list of waiting patients: ${response.statusText}`  @_add_@
      } @_add_@
    } catch (err: any) { @_add_@
      this.errorMessage = `Cannot retrieve list of waiting patients: ${err.message || "unknown"}`  @_add_@
    } @_add_@
    return []; @_add_@
  } 
...
```

In this step, we modified the way of obtaining the list of patients - now we retrieve the list of patients using the client code from our API. We introduced new variables - attributes of the element - `apiBase` and `ambulanceId`, which we will use to connect to the API. In the code, we also handle the possibility that we fail to connect to the API or fail to retrieve the list of patients. In such cases, we display an error message. In the same file, modify the `render` method:

```tsx
render() {
  return (
    <Host>
      {this.errorMessage @_add_@
        ? <div class="error">{this.errorMessage}</div>  @_add_@
        :  @_add_@
          <md-list> 
          ...
        </md-list>
      }  @_add_@
    </Host>
  );
}
```

6. Modify the file `${WAC_ROOT}/ambulance-ufe/src/components/<pfx>-ambulance-wl-app/<pfx>-ambulance-wl-app.tsx` and add similar attributes of the element that will be passed to the `<pfx>-ambulance-wl-list` element:

```tsx
...
export class <Pfx>AmbulanceWlApp { 
  @State() private relativePath = "";
  @Prop() basePath: string="";
  @Prop() apiBase: string; @_add_@
  @Prop() ambulanceId: string; @_add_@
  ...  
  render() {
    ...
    return (
      <Host>
        { element === "editor" 
        ? <pfx-ambulance-wl-editor entry-id={entryId}
          oneditor-closed={ () => navigate("./list")}
        ></pfx-ambulance-wl-editor>
        : <pfx-ambulance-wl-list  ambulance-id={this.ambulanceId} api-base={this.apiBase} @_important_@
          onentry-clicked={ (ev: CustomEvent<string>)=> navigate("./entry/" + ev.detail) } >
          </pfx-ambulance-wl-list>
        }
      </Host>
  ...
```

Finally, modify the file `${WAC_ROOT}/ambulance-ufe/src/index.html` and add attributes to the `<pfx>-ambulance-wl-app` element:

```html
<body style="font-family: 'Roboto'; ">
  <<pfx>-ambulance-wl-app ambulance-id="bobulova" api-base="http://localhost:5000/api" base-path="/ambulance-wl/"></<pfx>-ambulance-wl-app> @_important_@
</body>
```

7. Start the development server by executing the command in the `${WAC_ROOT}/ambulance-ufe` directory:

```ps
npm run start
```

and go to the page [http://localhost:3333](http://localhost:3333). You should see a page with a list of patients corresponding to our example in the [OpenAPI] specification. You can also try using the application without access to the WEB API using the command `npm run start:app`. In this case, an error message will be displayed that it was not possible to connect to the API. To make the message clearer, modify the file `${WAC_ROOT}/ambulance-ufe/src/components/<pfx>-ambulance-wl-list/<pfx>-ambulance-wl-list.css`:

```css
:host {
  display: block;
  width: 100%;
  height: 100%;
}

.error {
  margin: auto;
  color: red;
  font-size: 2rem;
  font-weight: 900;
  text-align: center;
}
```

8. Before archiving the changes, we still need to fix our tests. If you execute the command in the `${WAC_ROOT}/ambulance-ufe` directory now:

```ps
npm run test
```

If you see an error in the output, such as _SyntaxError: Cannot use import statement outside a module_, pointing to code in the [axios] library, it is similar to the issue encountered when using the [@material/web][md-webc] library. This is caused by different assumptions about the ECMAScript language version and support for the latest module loading methods between these libraries and the testing library [Jest]. [Jest] can be configured using so-called [Code Transformers](https://jestjs.io/docs/code-transformation).

First, install the necessary packages. Execute the following command in the `${WAC_ROOT}/ambulance-ufe` directory:

```ps
npm install --save-dev @babel/preset-env babel-jest
```

Create a file `${WAC_ROOT}/ambulance-ufe/babel.config.cjs` and insert the following content:

```js
module.exports = {
presets: [ '@babel/preset-env' ]
}
```

[Babel] is a [JavaScript transpiler](https://en.wikipedia.org/wiki/Source-to-source_compiler) that allows translating ECMAScript code into various target language versions. It is used, for example, when during development, we want to use the latest language features and at the same time want to support older language versions - for instance, older, not yet modernized browsers.

Next, modify the file `${WAC_ROOT}/ambulance-ufe/stencil.config.ts` and add new code transformation rules to the `testing` section:

```js
...
export const config: Config = {
...
testing: {      @_add_@
    transformIgnorePatterns: ["/node_modules/(?!axios)"], @_add_@
    transform: { @_add_@
      "^.+\\.(js|jsx)$": "babel-jest", @_add_@
    },  @_add_@
}
...
```

Run the command `npm run test` again, and this time the tests should run successfully.

9. If we review the test in the file `${WAC_ROOT}/ambulance-ufe/src/components/pfx-ambulance-wl-list/test/pfx-ambulance-wl-list.spec.tsx`, we understand that our test will pass even when the connection to the server fails, resulting in an empty patient list. Therefore, it would be more appropriate if we could simulate a connection to the API server. For this, we will use the [axios-mock-adapter](https://github.com/ctimmerm/axios-mock-adapter) library. Install this library into the project:

```ps
npm install --save-dev axios-mock-adapter
```

Open the file `${WAC_ROOT}/ambulance-ufe/src/components/pfx-ambulance-wl-list/test/pfx-ambulance-wl-list.spec.tsx` and modify its content:

```tsx
import { newSpecPage } from '@stencil/core/testing';
import { <Pfx>AmbulanceWlList } from '../<pfx>-ambulance-wl-list';
import axios from "axios"; @_add_@
import MockAdapter from "axios-mock-adapter"; @_add_@
import { WaitingListEntry } from '../../../api/ambulance-wl'; @_add_@

describe('<pfx>-ambulance-wl-list', () => {

  const sampleEntries: WaitingListEntry[] = [ @_add_@
    { @_add_@
      id: "entry-1", @_add_@
      patientId: "p-1", @_add_@
      name: "Juraj Prvý", @_add_@
      waitingSince: "20240203T12:00", @_add_@
      estimatedDurationMinutes: 20 @_add_@
    }, { @_add_@
      id: "entry-2", @_add_@
      patientId: "p-2", @_add_@
      name: "James Druhý", @_add_@
      waitingSince: "20240203T12:05", @_add_@
      estimatedDurationMinutes: 5 @_add_@
    }]; @_add_@
    @_add_@
  let mock: MockAdapter; @_add_@
  @_add_@
  beforeAll(() => { mock = new MockAdapter(axios); }); @_add_@
  afterEach(() => { mock.reset(); }); @_add_@

  it('renders sample entries', async () => { @_important_@
    // simulate API response using sampleEntries 
    mock.onGet().reply(200, sampleEntries); @_add_@

    // set proper attributes
    const page = await newSpecPage({
      components: [<Pfx>AmbulanceWlList],
      html: `<<pfx>-ambulance-wl-list ambulance-id="test-ambulance" api-base="http://test/api"></<pfx>-ambulance-wl-list>`, @_important_@
    });
    const wlList = page.rootInstance as <Pfx>AmbulanceWlList;
    const expectedPatients = wlList?.waitingPatients?.length;

    const items = page.root.shadowRoot.querySelectorAll("md-list-item");
    // use sample entries as expectation
    expect(expectedPatients).toEqual(sampleEntries.length); @_important_@
    expect(items.length).toEqual(expectedPatients);
  });
...
```

In this step, we modified the test to simulate a response from the API server. We created a list of patients that we will use as the expected result. In the test, we used the `reply` method to simulate the API server response. We also adjusted the expected result in the test to expect the number of `md-list-item` elements equal to the number of patients in the simulated response list `sampleEntries`.

We will expand the test to also check the component's reaction in the case of an error response. In the same file, add the new test:

```tsx
...
describe('<pfx>-ambulance-wl-list', () => {
  ...
  it('renders sample entries', async () => { 
    ...
  });

  it('renders error message on network issues', async () => {  @_add_@
    mock.onGet().networkError();  @_add_@
    const page = await newSpecPage({  @_add_@
      components: [<Pfx>AmbulanceWlList],  // @_add_@
      html: `<pfx-ambulance-wl-list ambulance-id="test-ambulance" api-base="http://test/api"></pfx-ambulance-wl-list>`,  // @_add_@
    });  @_add_@
      @_add_@
    const wlList = page.rootInstance as <Pfx>AmbulanceWlList; // @_add_@
    const expectedPatients = wlList?.waitingPatients?.length  @_add_@
    @_add_@
    const errorMessage =  page.root.shadowRoot.querySelectorAll(".error");  @_add_@
    const items = page.root.shadowRoot.querySelectorAll("md-list-item");  @_add_@
      @_add_@
    expect(errorMessage.length).toBeGreaterThanOrEqual(1)  @_add_@
    expect(expectedPatients).toEqual(0);  @_add_@
    expect(items.length).toEqual(expectedPatients);  @_add_@
  });  @_add_@
...
```

Notice how we simulate an error response from the API server using the `networkError` method. In the test, we expect an error message to be displayed, and no record to be shown in the list of patients.

>info:> Testing negative scenarios - so-called _rainy days use cases_ - requires proper attention in practice. Development often takes place in artificial environments, under ideal network connection conditions and with sufficient system objects. Limitations in real environments may result in delayed product delivery or complete rejection of the product by users. As mentioned, in this exercise, we only briefly touch on testing, and the scope of the tests provided here would be insufficient in practice.

10. (_Optional_) If you want to use the [Jest] testing environment directly, for example, you want to use some popular extensions like [vscode-jest](https://marketplace.visualstudio.com/items?itemName=Orta.vscode-jest), add configuration for the correct behavior of [jest cli](https://jestjs.io/docs/cli) tools to the project. Create a file `${WAC_ROOT}/ambulance-ufe/jest.config.js` with the following content:

```js
module.exports = {
    "roots": [
        "<rootDir>/src"
    ],
    "transform": {
        "^.+\\.(ts|tsx)$": "<rootDir>/node_modules/@stencil/core/testing/jest-preprocessor.js",
        "^.+\\.(js|jsx)$": "babel-jest",
    },
    transformIgnorePatterns: ["/node_modules/(?!axios)"],
    "testRegex": "(/__tests__/.*|\\.(test|spec))\\.(tsx?|jsx?)$",
    "moduleFileExtensions": [
        "ts",
        "tsx",
        "js",
        "json",
        "jsx"
    ]
}
```

In the file `${WAC_ROOT}/ambulance-ufe/package.json`, add a new script:

The functionality of the command is similar to the `npm run test` command. The advantage is that we can use extensions for [Jest] in our development environment or some pre-configured [GitHub Actions](https://github.com/marketplace?type=actions&query=jest+) in continuous integration.

11. Archive your code using the following commands:

```ps
git add .
git commit -m "Added ambulance waiting list API"
git push
```

After creating a new version of the image, it will be deployed to the server. If you have the cluster started, you can verify the functionality on the page [http://localhost:30331](http://localhost:30331). In this case, we have not yet set the correct attributes for our application. Open the file `${WAC_ROOT}/ambulance-gitops/apps/<pfx>-ambulance-ufe/webcomponent.yaml` and add the attributes `api-base` and `ambulance-id`:

```yaml
...
navigation:
  - element: <pfx>-ambulance-wl-app
    attributes:                    @_add_@
    - name: api-base @_add_@
      value: http://localhost:5000/api @_add_@
    - name: ambulance-id @_add_@
      value: bobulova @_add_@
    ...
    hash-suffix: v1alpha2 @_important_@
```

>$apple:> If you previously changed the port number, don't forget to set the correct port here as well.

Then, in the directory `${WAC_ROOT}/ambulance-gitops`, commit and push:

```ps
git add .
git commit -m "Added ambulance waiting list API"
git push
```

After the time needed for the changes to be applied to the cluster, verify the functionality at [http://localhost:30331](http://localhost:30331). On the screen, you see an error message. The reason is that the server call is not going through. Verify this by checking the 'Network' tab in the 'Developer Tools' panel, where you can see the unsuccessful API call to `http://localhost:5000/api/waiting-list/bobulova/entries`. The call fails due to a violation of CSP rules (we will explain and address this in the next chapters).

However, even if we resolve the CSP issue, the server call won't be functional. The reason is that there is no server running at `localhost:5000`. We can simulate it by running the command `npm run start:mock`.
