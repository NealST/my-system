## Preface
[React Testing Library](https://testing-library.com/docs/react-testing-library/intro/) has become the de facto standard for testing React components. Focus on testing from the user's perspective and avoidance of implementation details in tests are some of the main reasons for its success.

Properly written tests not only help prevent regressions and buggy code, but in the case of React Testing Library, they also improve the accessibility of components and the overall user experience. When working with React components, it's important to use proper testing techniques to avoid common mistakes with React Testing Library.

This article will cover some of the most common mistakes made when using React Testing Library, along with the topics like using specific queries to improve test coverage, the importance of *ByRole queries, using userEvent methods over fireEvent for simulating user interactions, and how to use findBy* queries and waitForElementToBeRemoved for asynchronous testing. By the end of this article, you'll be equipped with the knowledge to write better React Testing Library tests, avoid common mistakes, and improve the overall quality of your React applications.

## Default to *ByRole queries
One of the powerful advantages of React Testing Library is that with the right queries, we can ensure not only that components work as expected, but also that they are accessible. So how do we figure out which query is the best? The rule is quite simple - use *ByRole queries by default. These queries work for many elements, even complex select components.

Like most rules, it has exceptions since not all HTML elements have a default role. The list of default roles for HTML elements can be found at w3.org.

## Testing form components
Let's consider the following form component,
```jsx
export const Form = ({ saveData }) => {
  const [state, setState] = useState({
    name: "",
    email: "",
    password: "",
    confirmPassword: "",
    conditionsAccepted: false,
  });

  const onFieldChange = (event) => {
    let value = event.target.value;
    if (event.target.type === "checkbox") {
      value = event.target.checked;
    }

    setState({ ...state, [event.target.id]: value });
  };

  const onSubmit = (event) => {
    event.preventDefault();
    saveData(state);
  };

  return (
    <form className="form" onSubmit={onSubmit}>
      <div className="field">
        <label>Name</label>
        <input
          id="name"
          onChange={onFieldChange}
          placeholder="Enter your name"
        />
      </div>
      <div className="field">
        <label>Email</label>
        <input
          type="email"
          id="email"
          onChange={onFieldChange}
          placeholder="Enter your email address"
        />
      </div>
      <div className="field">
        <label>Password</label>
        <input
          type="password"
          id="password"
          onChange={onFieldChange}
          placeholder="Password should be at least 8 characters"
        />
      </div>
      <div className="field">
        <label>Confirm password</label>
        <input
          type="password"
          id="confirmPassword"
          onChange={onFieldChange}
          placeholder="Enter the password once more"
        />
      </div>
      <div className="field checkbox">
        <input type="checkbox" id="conditions" onChange={onFieldChange} />
        <label>I agree to the terms and conditions</label>
      </div>
      <button type="submit">Sign up</button>
    </form>
  );
};
```
We can test this by simulating the data entry through form elements, submitting the form, and then verifying that the saveData prop received the data we input. We can break it down into 3 steps:

1. Enter the text for fields we want to test (or click on the checkbox)
2. Click the Sign up button
3. Verify that saveData was called with the data we entered.

This workflow closely mirrors how a user would interact with our form (although they might not inspect the saved data in the same manner).

## Querying by placeholder text
Let's begin by entering a name into the first input field. We notice it has an Enter your name placeholder, so why not use that for querying the input?
```jsx
import { render, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import "@testing-library/jest-dom/extend-expect";

const defaultData = {
  conditionsAccepted: false,
  confirmPassword: "",
  email: "",
  name: "",
  password: "",
};

describe("Form", () => {
  it("should submit correct form data", async () => {
    const user = userEvent.setup();
    const mockSave = jest.fn();
    render(<Form saveData={mockSave} />);

    await user.type(screen.getByPlaceholderText("Enter your name"), "Test");
    await user.click(screen.getByText("Sign up"));

    expect(mockSave).toHaveBeenLastCalledWith({ ...defaultData, name: "Test" });
  });
});
```
It works, but we can do better than that. Firstly, this approach may foster the practice of using placeholder text as labels, which is not what they're meant for and is discouraged by [W3C WAI](https://www.w3.org/WAI/tutorials/forms/instructions/#placeholder-text). Secondly, we're not testing with accessibility concerns in mind.

## Querying specific components by role
Instead, let's try replacing our query with getByRole. As the documentation says, we can match the input of type text by the textbox role. However, since we have multiple textboxes in the form, we need to be more specific than that. Luckily, the query accepts a second parameter, which is an options object, where we can narrow down the match using the name attribute.

From the [documentation](https://testing-library.com/docs/queries/byrole/), we can see that name here doesn't refer to the input's name attribute, but rather its accessible name. As a result, for inputs, the accessible name is often the text content of their labels. In our form, the name input has a Name label, so let's use that.

```jsx
user.type(screen.getByRole("textbox", { name: "Name" }), "Test");
```
When running the test we get an error:
```text
TestingLibraryElementError: Unable to find an accessible element with the role "textbox" and name "Name"
```
The help text below shows that our input doesn't have an accessible name:
```text
Here are the accessible roles:

  textbox:

  Name "":
  <input
    id="name"
    placeholder="Enter your name"
  />

```
We do have a label for the input, so why isn't this working? It turns out that the label needs to be associated with the input. To achieve this, the label should have a for attribute matching the associated input's id. Alternatively, the input element can be wrapped within the label. It seems our input already has an id, so we just need to add the for (htmlFor when using React) attribute to it:
```jsx
<label htmlFor="name">Name</label>
<input
  id="name"
  onChange={onFieldChange}
  placeholder="Enter your name"
/>
```
Now, the input is properly associated with its label, and the test passes. This also significantly improves accessibility. Firstly, when clicking or tapping a label, the focus will be passed to the associated input. Secondly, and most importantly, screen readers will read out the label when the input is focused, providing additional information about the input to the user. This demonstrates how switching to getByRole not only enhances test coverage but also offers valuable accessibility improvements to our form component.

## Improving the button test
Upon examining the test again, we see that the getByText query is used for the submit button. In my opinion, *ByText should be the last resort (or perhaps second-to-last before *ByTestId) since they are the most prone to breaking.

In our test, screen.getByText("Sign up") matches the element with a text node that has a Sign up text content. If we later decide to add a paragraph on the same page with the text "Sign up", that element will also be matched, and the test will break due to multiple matching elements. The situation worsens when using a generic regex for the text match instead of the string: screen.getByText(/Sign up/i). This matches any occurrence of the string "sign up" regardless of the case, even if it is part of a larger sentence.

Although we could modify the regex to ensure it matches only this specific string, we can instead use a more precise query and simultaneously verify that our form is accessible with the help of a *ByRole query. In this case, the exact query will be screen.getByRole("button", { name: "Sign up" });. The accessible name this time is the actual text content of the button. Note that if we add an aria-label to the button, the accessible name will be the text content of that aria-label.

Ultimately, the updated test looks like this:
```jsx
describe("Form", () => {
  it("should submit correct form data", async () => {
    const user = userEvent.setup();
    const mockSave = jest.fn();
    render(<Form saveData={mockSave} />);

    await user.type(screen.getByRole("textbox", { name: "Name" }), "Test");
    await user.click(screen.getByRole("button", { name: "Sign up" }));

    expect(mockSave).toHaveBeenLastCalledWith({ ...defaultData, name: "Test" });
  });
});

```
## *ByRole vs *ByLabelText for input elements
You might wonder why we use *ByRole queries for input elements, as their purpose is to match input by its associated label. Wouldn't it be easier to use *ByLabelText queries instead, since they ultimately achieve the same goal and the syntax is a bit lighter? While there might not seem to be a significant difference between the two queries, *ByRole is more robust when matching elements and will still work if you switch e.g. from <label> to aria-label.

On the other hand, not all types of input elements have a default role. For example, for a password or email input, we would use the *ByLabelText query. So, while both methods have their merits, it's essential to consider the specific use case and choose the query that best suits the situation.

## Use userEvent instead of fireEvent
You might have noticed that we don't use the built-in fireEvent method to simulate events, but instead default to the userEvent methods. Although fireEvent works in many cases, it is simply a light wrapper on top of the dispatchEvent API and does not fully simulate user interactions. On the other hand, userEvent manipulates the DOM in the same way a user would in a browser, providing a more reliable testing experience. Its approach also aligns better with the philosophy of React Testing Library, and the syntax is clearer.

Starting from version 13 of testing-library/user-event, it is recommended to set up the user event before simulating the event. This will become mandatory in later versions, and simulating events directly from userEvent will no longer work.

To simplify the setup of userEvent, we can add a utility function that handles both the event setup and the component rendering simultaneously.

```jsx
// setup userEvent
function setup(jsx) {
  return {
    user: userEvent.setup(),
    ...render(jsx),
  };
}

describe("Form", () => {
  it("should save correct data on submit", async () => {
    const mockSave = jest.fn();
    const { user } = setup(<Form saveData={mockSave} />);

    await user.type(screen.getByRole("textbox", { name: "Name" }), "Test");
    await user.click(screen.getByRole("button", { name: "Sign up" }));

    expect(mockSave).toHaveBeenLastCalledWith({ ...defaultData, name: "Test" });
  });
});

```
All the userEvent methods are async, so we need to slightly adjust the tests are async as well. Additionally, since it's a separate package, it needs to be installed via npm i -D @testing-library/user-event.

## Simplify the waitFor queries with findBy*
There are often cases when the element we're trying to match is not available on the initial render, such as when we first fetch items from an API and then display them. In these situations, we need the component to complete all its rendering cycles before querying. As an example, let's modify the ListPage component to wait for the list of items to load asynchronously:

```jsx
export const ListPage = () => {
  const [items, setItems] = useState([]);

  useEffect(() => {
    const loadItems = async () => {
      setTimeout(() => setItems(["Item 1", "Item 2"]), 100);
    };
    loadItems();
  }, []);

  if (!items.length) {
    return <div>Loading...</div>;
  }

  return (
    <div className="text-list__container">
      <h1>List of items</h1>
      <ItemList items={items} />
    </div>
  );
};

```
The current version of the test for this component will no longer work since when the screen.getByRole query is called, only the loading text is displayed. To wait for the component to complete loading we can use the waitFor helper:
```jsx
import { waitFor } from "@testing-library/react";

//...

describe("ListPage", () => {
  it("renders without breaking", async () => {
    render(<ListPage />);

    await waitFor(() => {
      expect(
        screen.getByRole("heading", { name: "List of items" }),
      ).toBeInTheDocument();
    });
  });
});

```
This works, but there is a query type with async behavior built-in: the findBy* queries. These queries are a wrapper on top of waitFor, making the test more readable:
```jsx
describe("ListPage", () => {
  it("renders without breaking", async () => {
    render(<ListPage />);
    expect(
      await screen.findByRole("heading", { name: "List of items" }),
    ).toBeInTheDocument();
  });
});
```
It's worth noting that one await call per test block is usually sufficient, as all async actions have been resolved by that point. In the example above, if we want to additionally test that we have 4 items in the ItemList, we don't need to use the async findBy* query; instead, we can resort to getBy*.

## Testing element's disappearance
This is quite an edge case, but sometimes we want to test that an element, which was present before, has been removed from the DOM after some async action. React Testing Library has a handy helper for this - waitForElementToBeRemoved. For example, in the ListItem component, we might want to wait for the Loading... text to be removed instead of waiting for the list header to appear:
```jsx
it("renders without breaking", async () => {
  render(<ListPage />);
  await waitForElementToBeRemoved(() => screen.queryByText("Loading..."));
});

```
## Use React Testing Library Playground
If you have trouble figuring out the right query for certain elements, the [React Testing Library Playground](https://testing-playground.com/) is a great resource. Simply paste the HTML for the component being tested, and it will provide handy suggestions about which queries would work for each element. This tool is particularly valuable for complex components where it might not always be evident which query is best to use.

## Fixing the "not wrapped in act(...)" warnings
In some cases, particularly when working with components that incorporate asynchronous logic, you might encounter a warning while running tests:
```text
Warning: An update to ComponentName inside a test was not wrapped in act(...).
```
This warning suggests that your tests may not be accurately simulating how React updates components, potentially leading to false positives or negatives in your test results. This happens when your test triggers a state update or a side effect outside the act function provided by React Testing Library, causing React to update the component asynchronously.

Often, the cause of the warning is using getBy* queries instead of findBy* queries, as the element or component in question updates after an asynchronous action.

However, there are instances where the "act" warning is justified and needs to be addressed to fix false positives or negatives in the tests. One such case occurs when working with Jest timers. When testing components with timers, use fake timers (e.g., Jest's useFakeTimers) to control the flow of time and prevent inconsistencies. To run timers, you can use jest.runAllTimers();, which also needs to be wrapped in act.

```js
jest.useFakeTimers();
// ... Set up the tests

act(() => {
  jest.runAllTimers();
});

// ... Do assertions

jest.useRealTimers();

```
This warning can appear more frequently in React 18, as it introduced some changes to how the useEffect is executed. This will require more tests to be wrapped in act.

For a more comprehensive guide on addressing the "act" warning, refer to this article: [Fix the "not wrapped in act(...)" warning](https://kentcdodds.com/blog/fix-the-not-wrapped-in-act-warning).

## Writing smoke tests
Sometimes we want to have basic sanity tests to ensure that a component doesn't break during rendering. Let's consider this simple component:

```jsx
export const ListPage = () => {
  return (
    <div className="text-list__container">
      <h1>List of items</h1>
      <ItemList />
    </div>
  );
};
```
We could check that it renders without issues with a test like this:
```jsx
import { render } from "@testing-library/react";
import React from "react";

import { ListPage } from "./ListPage";

describe("ListPage", () => {
  it("renders without breaking", () => {
    expect(() => render(<ListPage />)).not.toThrow();
  });
});

```
This works fine for our purposes; however, we're underutilizing the power of React Testing Library. Instead, we could do this:
```jsx
import { render, screen } from "@testing-library/react";
import React from "react";

import { ListPage } from "./ListPage";

describe("ListPage", () => {
  it("renders without breaking", () => {
    render(<ListPage />);

    expect(
      screen.getByRole("heading", { name: "List of items" }),
    ).toBeInTheDocument();
  });
});

```
Although this is quite a simplified example, with this slight change, we're not only testing that the component doesn't break during rendering but also that it has a header element with the name List of items, which is properly accessible by screen readers.

## Conclusion
In this article, we've explored some techniques and best practices for improving React Testing Library tests and listed the most common mistakes to avoid. By using getByRole queries, we can ensure that our tests not only provide good coverage but also offer valuable accessibility improvements. We've also learned about the benefits of using userEvent methods over fireEvent and how to set it up for optimal use. Finally, we've seen how using findBy* queries and waitForElementToBeRemoved can help us to write more robust and reliable tests.

By following these tips and tricks, we can write better React tests that are easier to read, maintain, and debug. Remember, testing is an essential part of the development process, and investing time in writing good tests can pay off in the long run. With these techniques and best practices, we can ensure that our React applications are thoroughly tested and of high quality.



