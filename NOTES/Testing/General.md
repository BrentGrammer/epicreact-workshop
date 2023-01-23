# Testing

## What to test?

- [What to test?](https://kentcdodds.com/blog/how-to-know-what-to-test)
  - Test use cases, not lines of code (coverage is not the best indicator of
    what needs to be tested)
  - note the lines that are missing tests, don't think about the ifs/elses,
    loops, or lifecycles. Instead ask yourself: What use cases are these lines
    of code supporting, and what tests can I add to support those use cases?
- Code coverage is not a perfect metric, but it can be a useful tool in
  identifying what parts of our codebase are missing "use case coverage".
- Sometimes, our code coverage report indicates 100% code coverage, but not 100%
  use case coverage.
- **_the reason we have tests in place is to ensure that code continues to
  support the USE CASES we intend it to support, even as things change._** Not
  just that it tests lines or implementation

Caution: if you think about the code rather than the use cases, it becomes
dangerously natural to start testing implementation details. When you do that,
your code now has a third user.

- rather than thinking about the code, think about the observable effect that
  code has for the end user and developer user, that's your use case, test that.

## Testing in React:

- you have two users that you need to support: End users, and developer users
- Do not test impl details:
  - Lifecycle methods
  - Element event handlers
  - Internal Component State
- things you should be testing concern your two users. Each of these could
  change the DOM, make HTTP requests, call a callback prop, or perform any other
  number of observable side effects which would be useful to test:
  - User interactions (using userEvent from @testing-library/user-event): Is the
    end user able to interact with the elements that the component renders?
  - Changing props (using rerender from React Testing Library): What happens
    when the developer user re-renders your component with new props?
  - Context changes (using rerender from React Testing Library): What happens
    when the developer user changes context resulting in your component
    re-rendering?
  - Subscription changes: What happens when an event emitter the component
    subscribes to changes? (Like firebase, a redux store, a router, a media
    query, or browser-based subscriptions like online status)
- Note: most hooks and components should be tested in integration tests, not
  individualized tests

  - The exception is for highly reusable or highly complex hooks and components

  ### Useful packages for testing with React

  - Testing Library
  - @testing-library/user-event for user interaction
  - @testing-library/jest-dom

## Where to start?

- What is the worst thing that could break or make your user upset if it didn't
  work?
- Start with a e2e test covering the happy path for each use case and add more
  tests from there.
- Once you have a few E2E tests in place, then you can start looking at writing
  some integration tests for some of the edge cases that you are missing in your
  E2E tests and unit tests for the more complex business logic that those
  features are using

## AHA Programming

- Avoid Hasty Abstractions and Optimize for Change
- use code duplication until you feel pretty confident that you know the use
  cases for that duplicate code. What parts of the code are different that would
  make good arguments to your function? After you've got a few places where that
  code is running, the commonalities will scream at you for abstraction and
  you'll be in the right frame of mind to provide that abstraction.
- If you abstract early, you'll think the function or component is perfect for
  your use case and so you just bend the code to fit your new use case. This
  goes on several times until the abstraction is basically your whole
  application in if statements and loops
- Tip: can use jsinspect to analyze code for duplication and abstraction
  opportunitieshttps://github.com/danielstjules/jsinspect
- you shouldn't be dogmatic about when you start writing abstractions but
  instead write the abstraction when it feels right and don't be afraid to
  duplicate code until you get there.
- If you find yourself passing parameters and adding conditional paths through
  shared code, the abstraction is incorrect.
