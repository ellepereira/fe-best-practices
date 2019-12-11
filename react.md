## React Architecture Proposal

Goals of this proposal:

* Eliminate the cause of many common problematic patterns.
* Reduce the productivity cost of maintenance and designing new features.
* Reduce the amount of decisions a developer must make before typing any code. 
* Answer common questions for those that are new to front-end development.
* Create code that can easily and effectively be tested.

**Note: this guide is not meant to replace any style guides. The focus of this document will be on overall architecture rather than minute syntax details.**

## To Do
This document will eventually cover the entire architecture of a react application. There are still items that need to be defined:
- [ ] CSS / SCSS structure (if not entirely in javascript)
- [ ] Theming (react context? object or string?)
- [ ] Routing (config based?)
- [ ] Form Handling
- [ ] Unit Tests Coverage
- [ ] PWA
- [ ] Translation/Internationalization

## Directory Structure
This suggested directory structure is meant to help developers quickly locate logic and assets they wish to modify or debug.
You may find that this structure doesn't make sense for your use case and change accordingly. However, you should 
**always** be consistent within your project.

| Directory  | Description |
| ------------- | ------------- |
| [src/api](APIs) |  All `APIs`, `endpoints` and `interceptors` for communicating with the API should be housed within this folder.|
| src/assets  |Static assets (images, manifest files, etc) |
| [src/components](#components) | **Sharable and Base** components only. This folder should only house components that can be used more than once.
| src/layouts | Wrapper components that render its children within a layout. An application may contain multiple layout styles (ex: the login page only has the center div (no nav bar, etc) while the other pages have nav bars, title bars, full screen content.. etc).|
| [src/pages](#pages) | This is where you house components that represent an entire page. Routes should *only* be referencing these components. |
| src/router | Contains all routes in the site. (if managed through config files)|
| src/store | The main redux store + submodules are contained here.|
| src/formatters | Each file here is a function that takes in some input and formats it into a string (dates, phones, etc) |
| src/validators | Each file is a set of validators that take in a value and return if it's valid (ex: isEmail, isPhoneNumber...) |

## Redux
Communication with the API and other components should be done through Redux:
![redux pattern](https://github.com/luciano7/fe-best-practices/blob/master/redux-pattern-1.png?raw=true)

## Components
Having clear set definitions of component types and an easy understanding of their functionality significantly lowers
the cognitive cost of debugging and speeds up the creation of new functionality. The suggested structure of components makes it simple to split work amongst developers. If developers focus on a particular component type, other developers can agree on a contract of functionality between them. Having unclear expectations can easily 
cause misunderstandings, duplicate logic and bugs. 

Ideally, there should be at _least_ three types of components: **"Base", "Page" and "Shared"**.

### Page/Container Components

Page components should be completely aware of the application context. 
These components are (as the name implies) the representation of each "page" within the application.
The idea is that a single route will map to a single Page component (if the route is to a modal then it also counts as a page).
A Page component may be composed of its own logic/html, Shared components and Base components.

* It's **highly** encouraged that Page components handle all API communications (through a redux store). Communication with Shared components should use custom events (callbacks) in the Shared component ("onSave", "onUpdate", "onSearch", etc). There are a lot of reasons to do this:
  * Shared components are never guaranteed to be used only once per page, this could cause duplicate/wasteful API calls.
  * It's easier to optimize API calls in the Page component- you can for example cache results or get an entire list at once.
   * You can always still make the requests separately and display the Shared components as their data become available. You have all the choices.
  * It will be difficult to track down bugs when calls are made in many places. It's likely that you'll only have page loaded at a time- making it much easier to know where to start debugging.
 * It's unlikely that Page components will grow much larger because of this as most of the API logic will still be handled by Vuex, API models and the fact that Pages can be mostly composed of other components.
* Examples of Page components:
  * A `CartItemsPage`. A cart could display a list of items (shared component). A list of items is loaded through Vuex and the results sorted/filtered (whatever else) and passed down to shared components.
  * A `SearchResultsPage` that contains a `SearchInput` and a list of `SearchResult`s. When the user types a search term into the `SearchInput` component it fires a `onUpdate` event that the `SearchResultsPage` listens to and makes a request to redux to perform the search. Using `mapStateToProps` the page then gets its props updated and renders `SearchResults`.

### Base (UI) Components
Base components are meant to be reusable by all other components (including other base components). These components should be limited to very specific functionality and it should be possible to use the component outside a single specific application. Avoid doing too much of ANYTHING within a single Base component.

* Examples of base components:
  * Example: `BasePrimaryButton`, `BaseDropdown`, `BaseSwitch`, `BaseWhiteCard`.
  * It is recommended that you prefix Base components with the word `Base` or something similar.
* This does not mean you can't use ui libraries (like bootstrap, material design, etc) in these components.

### Shared Components
Shared components are essentially an extension of Base components but context aware in order to build larger pieces of functionality. These components should be reusable across Page components and should still be as focused as possible. If necessary, these components can be Higher-Order Components as well.

* Example of a Shared components:
  * Say you have two Page components: `CreateProfilePage` and `EditProfilePage`. These two pages contain the same shared component `ProfileEditor` which lists all available fields to be edited with pre-existing values if the profile already exists. Since the two pages share a lot in common, our shared component saves us a lot of duplication. 
  * Each Page component sends a list of fields to that can be edited and validation depends on that list. The shared component must know the fields list so that it can enable/disable the specific inputs in the HTML file and handle different validation making the component context aware (it edits a profile).
  * If you're wondering why we wouldn't perform the validation at the "Page" component - 
  having the validation happen at the Page components would cause duplicate logic between the two pages.
  * **However** do note that this component doesn't care about what "edit" and "create" means- it's possible that a new Page component called `view-profile-page` configures the fields so that they're ALL disabled. This is a great example of allowing re-usability and focusing on specific functionality allowing different Page components to determine usage.
  
### Other Possible Types
- As suggested on the "directories" section you could also have a `Layout` type of component which is essentially contains the layout ui and takes in a Page component to display within it. 
- If you'd like to avoid confusion it is also ok to have `Modal` components which are exactly like Page components but render out a modal.


## APIs
The Data/API layer of the application handles data fetching, orchestration and formatting. This layer has at least 4 types of components: APIs, Endpoints, Models and Interceptors.

### API Object/Module
The application may have multiple APIs it communicates with so it's important to have an API object defining its own base URLs, cookies, headers, interceptors (more on this later), etc. I strongly recommend using a library such as [axios](https://github.com/axios/axios) instead of rolling out your own solution.

An example of an API using [Axios](https://github.com/axios/axios):

```js
const myAPI = axios.create({
  baseURL: 'https://api.example.com'
});

// GET call to https://api.example.com/cars
myAPI.request({method: 'GET', url: 'cars'});
```


### Endpoints / Models
Each Endpoint takes in an API (will be used to perform the requests) and has logic specific to a single url/resource in that API.  By default an endpoint should expose all the supported http methods (get, post, patch, etc).

```js
const CarsEndpoint = Endpoint.create(MyAPI, '/cars/:id');

// Makes a GET request to https://api.example.com/cars/5
CarsEndpoint.get({id: 5});
```


### Interceptors / Middleware 
Interceptors are functions that act on API requests or responses. They are meant to be the glue between application details and the APIs while keeping both sides independent of each other. 

Examples:
An auth request interceptor can inject inject our login token to all requests:

```js
import authStore from './store/auth';

// note: you could also simply change the headers on the APi directly- this is just an example.
function AuthRequestInterceptor(request) {
 request.headers.token = authStore.state.token;
 return request;
}
```

Likewise you may want to redirect an user to login if they get a 403 from the API:
```js
import Router from 'router';

function AuthResponseInterceptor(response) {
 if (response.status === 403) {
  Router.go('unauthorized');
 }
}

```

### Typescript Example
If you want more specifics, here's a suggested typescript interface for the API layer:

```ts

// the "meta" field gives the ability for middleware to have handle extra options

export interface IBaseAPI {
  baseURL: string;

  defaults: IAPIDefaults;
  interceptors: IApiInterceptors;
  request<T>(requestConfig: IRequestConfig, meta?: any): ApiPromise<T>;
}

export interface IAPIDefaults {
  baseURL?: string;
  headers?: any;
  timeout?: number;
  options?: any;
}

// Essentially exposes get/post/patch/etc by default - 
// ths class can be extended for endpoints needing more than those methods
export interface IBaseEndpoint<T, Q> {
  API: IBaseAPI;

  get(params: Q, requestConfig?: IRequestConfig<Q>, meta?: any): ApiPromise<T>;
  query(params: Q, requestConfig?: IRequestConfig<Q>, meta?: any): ApiPromise<T[]>;
  remove(params: Q, requestConfig?: IRequestConfig<Q>, meta?: any): ApiPromise<any>;
  add(data?: any, requestConfig?: IRequestConfig<Q>, meta?: any): ApiPromise<T>;
  update(params: Q, data?: any, requestConfig?: IRequestConfig<Q>, meta?: any): ApiPromise<T>;
}

export interface IRequestConfig<Q = IRequestParams> extends IAPIDefaults {
  url?: string;
  method?: string;
  params?: Q;
  data?: any;
  responseType?: string;
}

// Defines how a response from the API will look like
export interface IResponseSchema<T = any> {
  data: T;
  status: number;
  statusText: string;
  headers: any;
  config: any;
  request?: any;
}

export interface IInterceptorManager<T = any> {
  handlers: Array<IInterceptor<T>>;
  use: (interceptor: IInterceptor<T>) => number;
  eject: (id: number) => number;
  ejectAll: () => void;
}

export interface IInterceptor<T> {
  id?: number;
  // runs if the request/response is successful
  onFulfilled?: (value: T) => T | Promise<T>;
  // runs if the request/response errors
  onRejected?: (error: any) => any;
}

export interface IApiInterceptors {
  request: IInterceptorManager<IInterceptorRequestConfig>;
  response: IInterceptorManager<IResponseSchema>;
}

export interface IInterceptorRequestConfig {
  config: IRequestConfig;
  meta?: any;
}

export interface IRequestParams {
  [name: string]: any;
}

export type RequestInterceptor = IInterceptor<IInterceptorRequestConfig>;

export type ResponseInterceptor<T> = IInterceptor<IResponseSchema<T>>;
```
