# INTEGRATION TESTING

- Should mostly use integration tests for most bang for the buck
- More setup, but worth it and saves work in the long run (also preventing tons of unit tests for the same coverage)

### Tradeoffs:
- We can test what is rendered but if you really need to test style changes for visual regression testing and things like that you need to use tools like percy.io and app load tools(sp?)

-----------

## Rendering App with all Providers

- you will need to wrap your App with necessary providers to render in tests - i.e. Redux store provider etc.

- If you keep your providers in one  component that takes children (i.e. your <App>), then you can use the wrapper  option with render():

```javascript
test('renders all the book information', async () => {
  render(<App />, {wrapper: AppProviders}) // AppProviders is a component with children

  // use screen debug helper to get feedback on what test is rendering
  screen.debug()
}
```

## Wait for Loading to finish on loading app:

- use waitForElementToBeRemoved() from @testing-library/react which takes a callback which will be called when a DOM update occurs or at a regular interval:

```javascript
test('renders all the book information', async () => {
  render(<App />, {wrapper: AppProviders})
  // get the loading spinner by it's aria-label and wait for it to be removed on app loads
  await waitForElementToBeRemoved(() => screen.getByLabelText(/loading/i)
}
```

## Mocking out AUTH:

- Ideally you want to do the side effect that the auth provider does so the app thinks you're logged in
- you could mock out your provider and the method to get tokens etc. alternatively

Example buildUser in test/generate module:

```javascript
function buildUser(overrides) {
  return {
    id: faker.random.uuid(),
    username: faker.internet.userName(),
    password: faker.internet.password(),
    ...overrides
  }
}
```

```javascript
test('renders all the book information', async () => {
  // this is the side effect for ex that occurs when authed:
  window.localStorage.setItem(auth.localStorageKey, 'SOME_FAKE_TOKEN')

  //Mock out HTTP requests to the auth service:
  // safe set fetch before patching:
  const originalFetch = window.fetch
  window.fetch = async(url, config) => {
    // log out url to see what req is failing in the terminal
    console.log(url, config)

    // custom handle what we want - i.e. the success response needed after authing to load app
    if (url.endsWith('/bootstrap')) {
      return { // mock out a success response returned by fetch...
        ok: true, 
        json: async () => ({user: {username: 'bob'}, listItems: []})
      }
    }

    // use real fetch by default if we don't handle it here
    return originalFetch(url, config)
  }

  render(<App />, {wrapper: AppProviders})

  await waitForElementToBeRemoved(() => screen.getByLabelText(/loading/i)
}
```


## NAVIGATE TO PAGE AND RENDER INFO:

```javascript
test('renders all the book information', async () => {
  const user = buildUser() // from a test/generate module with our builders
  window.localStorage.setItem(auth.localStorageKey, 'SOME_FAKE_TOKEN')

  const book = buildBook() // from generate module made by us - use faker etc. for model
  // use history pushstate to navigate to details page:
  window.history.pushState({}, 'Test Page', `/book/${book.id}`)

  const originalFetch = window.fetch
  window.fetch = async(url, config) => {
    if (url.endsWith('/bootstrap')) {
      return { // mock out a success response returned by fetch...
        ok: true, 
        json: async () => ({...user, token: 'FAKETOKEN', listItems: []})
      }
    } else if (url.endsWith(`/books/${book.id}`)) {
	// intercept request to backend for details page and return data expected
        return {
          ok: true,
  	  json: async () => ({book})
    }
    return originalFetch(url, config)
  }

  render(<App />, {wrapper: AppProviders})

  await waitForElementToBeRemoved(() => screen.getByLabelText(/loading/i) 
  screen.debug()


  // make assertions on information expected on the details page:
  // can look for roles with this to use in assertions:
  // screen.getByRole('blah') // shows you list of roles - scroll down to get specific ones in the terminal 

  expect(screen.getByRole('heading', {name: book.title})).toBeInTheDocument()

  // check image url is stored in the src for the image of the details page:
  expect(screen.getByRole('img', {name: /book cover/i})).toHaveAttribute(
    'src',
    book.coverImageUrl,
  )

  // check that a button expected is there on the details page:
  expect(screen.getByRole('button', {name: /add to list/i})).toBeInTheDocument()

/// verify that you are not showing the wrong things, i.e. things that load later or after an action etc.:
  // remember to use queryByRole and not getByRole because get throws when element not found
  expect(screen.queryByRole('button', {name: /remove from the list/i})).not.toBeInTheDocument()
}
```

## CLEANUP - Clear state and cache between tests


```javascript
// clear caches (react-query for ex.) and reset logged in user state
afterEach(async () => {
  queryCache.clear() // react-query
  await auth.logout() // clear authstate
}

test('renders all the book information', async () => {
  const user = buildUser()
  window.localStorage.setItem(auth.localStorageKey, 'SOME_FAKE_TOKEN')

  const book = buildBook()
  window.history.pushState({}, 'Test Page', `/book/${book.id}`)

  const originalFetch = window.fetch
  window.fetch = async(url, config) => {
    if (url.endsWith('/bootstrap')) {
      return {
        ok: true, 
        json: async () => ({...user, token: 'FAKETOKEN', listItems: []})
      }
    } else if (url.endsWith(`/books/${book.id}`)) {
        return {
          ok: true,
  	  json: async () => ({book})
    }
    return originalFetch(url, config)
  }

  render(<App />, {wrapper: AppProviders})

  await waitForElementToBeRemoved(() => screen.getByLabelText(/loading/i) 
  screen.debug()

  expect(screen.getByRole('heading', {name: book.title})).toBeInTheDocument()
  expect(screen.getByRole('img', {name: /book cover/i})).toHaveAttribute(
    'src',
    book.coverImageUrl,
  )

  expect(screen.getByRole('button', {name: /add to list/i})).toBeInTheDocument()
  expect(screen.queryByRole('button', {name: /remove from the list/i})).not.toBeInTheDocument()
}
```

## USE MOCK SERVICE WORKER:

- Move fetch mocking to one place to re-use throughout test suites

Make sure you have MSW configured (check for usage with create-react-app)


// in test-server.js
```javascript
import {setupServer} from 'msw/node' // msw package
import {handlers} from './server-handlers' // we've setup handler wrappers ourselves

// initialize msw server and pass in our handlers - see server-handlers.js in epic react proj
const server = setupServer(...handlers)

export * from 'msw'
export {server}
```

```javascript
// setupTests.js
import '@testing-library/jest-dom'
import {server} from 'test/server'

beforeAll(() => server.listen()) // start server before all tests
afterAll(() => server.close()) // close the server after all tests are finished
afterEach(() => server.resetHandlers()) // resets runtime handlers in between tests to what was 							set in the first place (...handlers)
```

NOTE: if you setup msw like this you can also use it as a mock back end during local development!
see server-handlers.js in the epic react final project codebase
- Kent Dodds mocks out the databases which you can reuse and interact with in the tests.

```javascript
afterEach(async () => {
  queryCache.clear() // react-query
  // clear or reset mocked or test databases as well between tests!
  Promise.all([
    auth.logout(),
    usersDB.reset(), // resets are custom from the epic react proj
    booksDB.reset(),
    listItemsDB.reset()
  ])
  
}

test('renders all the book information', async () => {
  const user = buildUser()
  // use faked out databases (server-handlers.js) to make a real user
  await usersDB.create(user)
  const authUser = await usersDB.authenticate(user)

  window.localStorage.setItem(auth.localStorageKey, authUser.token)
  // use a mocked out database
  const book = await booksDB.create(buildBook())
  window.history.pushState({}, 'Test Page', `/book/${book.id}`)

  render(<App />, {wrapper: AppProviders})

  /** **NOTE: 
	MSW adds a little tick to the event loop which could cause more loading elements or other things to remain on the screen.
	Make sure to check for removal of all of those items to proceed when the tick has moved on to the next and loading is finished
  */
  await waitForElementToBeRemoved(() => [
    ...screen.queryAllByLabelText(/loading/i), // returns arr so spread it
    ...screen.queryAllByText(/loading/i),
  ]) 
  screen.debug()

  expect(screen.getByRole('heading', {name: book.title})).toBeInTheDocument()
  expect(screen.getByRole('img', {name: /book cover/i})).toHaveAttribute(
    'src',
    book.coverImageUrl,
  )

  expect(screen.getByRole('button', {name: /add to list/i})).toBeInTheDocument()
  expect(screen.queryByRole('button', {name: /remove from the list/i})).not.toBeInTheDocument()
}
```