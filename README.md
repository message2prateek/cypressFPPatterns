# Functional Programming Test Patterns with Cypress

## No more Page Objects

Page Object design pattern has two benefits
1. They keep all page element selectors in one place and thus separation of Test code from Locators of the AUT
1. They standardize how tests interact with the page and thus avoid duplication of code and ease code maintenance.

OO in JS is a little awkward. Introduction on `Class` in ES6 helps but Classes, specifically `this` keyword, can still surprise people used to Java because they work differently than. [Here is a great blog from Kent C. Dodds which highlights this point](https://kentcdodds.com/blog/classes-complexity-and-functional-programming)

## Enter Page Modules

In Java land it's pretty common to find Page Objects which inherit from the Base Page. e.g. of this in JS will be

```JS
import { HomePage } from './BasePage'

class HomePage extends BasePage  {
  constructor() {
    super();
    this.mainElement = 'body > .banner';
  }

//... More code

export const mainPage = new MainPage();

}
```

With the move to FP we are going to loose not only Inheritance but the Class itself. Therefore we need to use `Modules` to arrange our code. Each  `module` exports public functions that can be imported into other modules and used.

```js
// HomePage Module  - HomePage.js
export function login(email, password){
  //...
}

export function logout(){
  //...
}

export function search(criterion){
  // ...
}
```

This module could be then imported into your tests or other modules and used as below.
```js
// HomePage Test - HomePageTest.js
import * as homePage from './HomePage.js'

describe('Home Page', ()=> {
  it('User can login', ()=> {
      cy.visit('/')
      homePage.login('prateek','123456')
  })
})

```

or We could selectively import individual functions from a module

```js
import {login} from './HomePage.js'
describe('Home Page', ()=> {
  it('User can login', ()=> {
    cy.visit('/')
    login('prateek','123456')
  })
})

```

### What about Inheritance

```java
public class HomePage extends BasePage {

}

```
A lot of times we come around Test suites where Page Objects extend a `BasePage` or every test file extends a `BaseTest` class. The intention behind doing this is often code reuse. Most often the `BaseTest` class has methods related to login, logout and logging etc. Please don't do that. Bundling unrelated functionality into a parent class for the purpose of reuse is an abuse of Inheritance.

Common functionality that is to be made available to Specs could be added as Cypress `Custom commands`. Custom Commands are available to be used globally with the `cy.` prefix. e.g. we can add a method called `login` as a custom command as below.

```js
Cypress.Commands.add('login', (username, password) => {
    cy.get('#username').type(username)
    //...
})
```

The `Cypress.Commands.add` takes the name of the custom command as the first argument and a closure as the second argument.

Now we could use the custom command in any spec.

```js
describe('Login Page', ()=>{
  it('User can login', ()=>{
    cy.login('prateek','123456')
    // ...
  })
})

```

Functionality that is shared between few Specs but not all should be added to utility modules. 

#### Favour composition over Inheritance

Why? Watch this video

<iframe width="560" height="315" src="https://www.youtube.com/embed/wfMtDGfHWpA" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

Consider the below code which uses Inheritance 

```js
class Person {
  constructor(firstName, lastName) {
    this.firstName = firstName;
    this.lastName = lastName;
  }
  getInfo(greetStr) {
    return `${greetStr}, I am ${this.firstName} ${this.lastName}.`;
  }
}

class Employee extends Person {
  constructor(firstName, lastName, employeeId) {
    super(firstName, lastName);
    this.employeeId = employeeId;
  }

  getId(greetStr) {
    return `${greetStr}, my employee id is ${this.employeeId}.`;
  }
}
const employee = new Employee('John', 'Doe', 123);
console.log(employee.getInfo('Hi')); // Hi, I am John Doe.
console.log(employee.getId('Hello')); // Hello, my employee id is 123.
```

The same functionality could be achieved using Composition like so

```js
// We first define all the functions that the classes will have included e.g. getInfo() and getId()

function getInfo(firstName, lastName, greetStr){
  return `${greetStr}, I am ${firstName} ${lastName}.`
}

function getId(emplyeeId, greetStr){
  return `${greetStr}, my employee id is ${emplyeeId}.`;
}

// Instead of a Person Class we create a function which returns an Object which represents the person. This object is "Composed" of the bindings and functions available to us.

function CreatePerson(firstName, lastName){
  return {
    firstName: firstName,
    lastName: lastName,
    getInfo: (greetStr) => getInfo(firstName, lastName, greetStr) 
  }
}
 
function CreateEmployee(firstName, lastName, employeeId){
  return {
    employeeId: employeeId,
    getId : (greetStr) => getId(employeeId, greetStr),
    getInfo: (greetStr) => getInfo(firstName, lastName, greetStr) 
  };
}

// Notice that the objects returned by CreatePerson and CreateEmployee functions are independent of each other(i.e. not bounded by a relation aka inheritance). Both the returned objects have a property on them who's value happens to be the same function.

let person = CreatePerson('Bla', 'Bla')
let employee = CreateEmployee('John', 'Doe', 123)
console.log( employee.getInfo('Hi')) // Hi, I am John Doe.
console.log( employee.getId('Hello')) // Hello, my employee id is 123.
```

Functions which return objects are called `Factory functions`. Watch the below video for simpler and more convinsing argument for using *Factory Functions* over classes.

<iframe width="560" height="315" src="https://www.youtube.com/embed/ImwrezYhw4w" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>


### Readability

The main benefit of using Page Object is that it encapsulates the complexity of the UI and locators and thus helping with reusability and making the tests more readable.

> For these examples I am using the [Cypress TodoMVC Example Repo](https://github.com/cypress-io/cypress-example-todomvc) and refactoring few tests.

Compare the below tests
```js
describe('Todo Application', ()=> {

  it('Can add a new Todo', ()=>{
     cy.get('.new-todo')
      .type('First Todo')
      .type('{enter}') 

     cy.get('.todo-list li')
      .eq(0)
      .find('label')
      .should('contain', 'First Todo')  
  })
})

```

VS

```js
import {addTodo, getTodoName} from './TodoUtil'

describe('Todo Application', ()=> {

  it('Can add a new Todo', ()=>{
     addTodo('First Todo')
     .then(getTodoName)
     .should('equal', 'First Todo')
  })
})

```

The second test requires less cognitive load to understand because it's declarative and don't makes us read through the steps of how to add new todo or get it's name.

The `addTodo` and `getTodoName` come from the `TodoUtil` module

```js
// TodoUtil.js
export const addTodo = (name) => {
  cy.get('.new-todo')
    .type('First Todo')
    .type('{enter}') 

  return cy
    .get('.todo-list')
    .eq(0)
}

export const getTodoName = (todo) => {
  return todo.find('label)
}
```

While the first approach is fine for small and simple scenarios but as the scenarios become more complex or longer a more declarative approach can be a lifesaver.

```js
// The scenario to test that we are able to update the newly created Todo will look like
import {addTodo, getTodoName, updateTodo} from './TodoUtil'

describe('Todo Application', ()=> {

  const TODO_NAME = 'First todo'

it('Can update a newly created todo', ()=>{
    addTodo(TODO_NAME)
    .then(updateTodo(TODO_NAME + ' new'))
    .then(getTodoName)
    .should('equal', TODO_NAME + ' new')

  })
})

```

The new method `updateTodo` from `TodoUtils.js` looks like

```js
export const updateTodo = (name) => ($todo) => {
  cy.wrap($todo).within(() => {
    cy.get('label').dblclick()
    cy.get('.edit').clear().type(`${name}{enter}`)
  })

  return cy.wrap($todo)
```

## But I still love my Page Objects.

Most common arguments against Page Objects are 

1. Page Objects introduce additional state in addition to the AUT's state which makes tests hard to understand.
1. Using Page object means that all our tests are going to go through the Application's GUI.
1. Page objects try to fit multiple cases into a uniform interface, falling back to conditional logic.

While the above arguments are all true in my experience, but biggest problem with Page Objects arise due to Selenium's recommendation that "methods should return Page Objects". 

[Read all recommendations here](https://github.com/SeleniumHQ/selenium/wiki/PageObjects)

Let's try to look at some common situations we find ourselves in when using Page Objects and how to solve them.

### Single Responsibility Principle is not met by Page Objects
PO bind unrelated functionality together in one class. e.g. in the below code `searchProduct()` functionality is not related to `login` or `logout` actions.

```java
public class HomePage {
    private final WebDriver webDriver;

    public HomePage(WebDriver webDriver) {
        this.webDriver = webDriver;
    }

    public SignUpPage signUp() {
        webDriver.findElement(By.linkText("Sign up")).click();
        return new SignUpPage(webDriver);
    }

    public void logOut() {
        webDriver.findElement(By.linkText("Log out")).click();
    }

    public LoginPage logIn() {
        webDriver.findElement(By.linkText("Log in")).click();
        return new LoginPage(webDriver);
    }
    
    public ProductPage searchProduct(String product){
        webDriver.findElement(By.linkText(product)).click();
        return new ProductPage(webDriver);
    }
    
}

```

This means that or class does not follow Single Responsiility Princial (SPR).

> The **Single Responsibility Principal (SRP)** states that every module or class should have responsibility over a single part of the functionality provided by the software, and that responsibility should be entirely encapsulated by the class.

Breaking Page Objects like above into multiple smaller Page Objects does not help with `SRP` either. 

e.g. We could take the `Login` action outside the `HomePage` and create a new `LoginPage` object and use it like below.

```Java
LoginPage loginPage = new HomePage().navigateToLoginPage();
loginPage.login("username", "password");
``` 

As the actions belong to 2 different pages, this code will repeat in all test cases that use login. The responsibility is not entirely encapsulated.

One way of fixing this is be to define our Class/Module not by the page but by the intent.

```js
// login.js

export const loginAsCustomer = (name, password) => {
}

```
The `loginAsCustomer` method can then work through both the Home and Login screens of the application to complete login as a single user action.

> :pencil: If possible, prefer basing your modules on user intent rather than basing them strictly by Page.

### Page Order != User Flows

Another place where PO may complicate things is when User flows are not be same as the page order. 

Consider the example of a shopping website. Here the user can add an item to the cart either using the Product Page or using the search functionality on the Search page. 

From the Cart page the user maybe taken to either the Home page or the Search page on clicking "continue to shop", depending on if the last item was added using the Product Page or the Search Page.

The code for the `CartPage` class might now look something like this 

```java
public class CartPage{       
    Page continueShopping(){
     if(state) {  // determines using which page the last item was added
       return new SearchPage();
     }
     else {
       return new HomePage();
     }    
}

```

**Why is this a problem?**

Not only is this code more complex to understand and we have to maintain additional state, it also makes it harder to modify the CartPage if in future another user flow was introduced. This violates the _Open/Closed principle (OCP)._

> **The open/closed principle(OCP)** states “software entities (classes, modules, functions, etc.) should be open for extension, but closed for modification”

One way to remove the state logic from our `Cart` Module is by changing the `continueShopping` method to not return references to other Classes/Modules and limit it to only clicking the continue shopping link.

```js
// cart.js

export const continueShopping(){
  cy.get('#continue').click();
}

// Test.js

it('user can add item to cart from home page post selecting "continue to shop"', (){
    //.... code to add product to the cart from Product Page
    cartPage.continueShopping();
    homePage.navigateToProductPage();
    productPage.addItemToCart('item2');
})

it('user can add item to cart from search page post selecting "continue to shop"', (){
    //.... code to add product to the cart using Search
    cartPage.continueShopping();
    searchPage.addItemToCart('item');
})
```

In this example our Test builds user flows just by choosing the right order of calling loosely coupled steps. This means our individual modules did not have to maintain state and state based was removed.

### Loosely coupled steps

Need another example of how loosely coupled steps reduce complexity? Consider the below typical `LoginPage` class.

Business requirement is that on successful login use is taken to "Home Page" and on unsuccessful login use stays on the "Login" page.

```java
class LoginPage {
    HomePage validLogin(String userName,String password){...}
    LoginPage invalidLogin(String userName,String password){...}
  }
}
```

Now let's introduce roles in the mix. An "Admin" on login is taken to the "Admin Dashboard" instead of the Home Page. So we now need to add another method to the LoginPage class and return an instance of the "Admin Dashboard" Page.

```java
class LoginPage {
    HomePage validLogin(String userName,String password){...}
    LoginPage invalidLogin(String userName,String password){...}

    AdminDashboardPage adminValidLogin(String userName,String password){...}
  }
}
```

More roles will mean even more methods because there is a tight coupling between the pages and the return type. 

Let's fix this by not returning references to different pages by the login action.

```js
// login.js
export default const login = (username, password) => {
  cy.get('.username').type(username)
  cy.get('.password').type(password)
  cy.click('.loginButton')
}
```

Our test will now look like

```js
// Test.js
it('User is taken to Home Page on valid login', ()=> {
   login('prateek', '12345')
   cy.title().should('equal', 'Home Page');
})

it('Admin is taken to Admin Dashboard on valid login', ()=> {
   login('admin', '12345')
   cy.title().should('equal', 'Admin Dashboard');
})

```

Hopefully you can see that preferring Loosely Coupled steps leads to us writing less lines of code with reduced complexity.

> :bulb: ASS tests in NIB when through a big refactor 2 years back to not have methods return Page Objects.
