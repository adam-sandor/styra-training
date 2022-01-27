# Tutorial: Basic Rules with API Authorization and Styra DAS

The Open Policy Agent (OPA) is an open source, general-purpose policy engine that enables unified, context-aware policy enforcement across the entire stack.  Styra Declarative Authorization Service (DAS) is a commercial product that provides a single pane of glass (aka a control plane or management plane) for all of the OPAs running throughout your infrastructure.

OPA provides a high-level **declarative language** for authoring policies and
simple APIs to answer policy queries. Using OPA, you can offload policy
decisions from your service such as:

* Should this API call be allowed? E.g., `true` or `false`.
* How much quota remains for this user? E.g., `1048`.
* Which hosts can this container be deployed on? E.g., `["host1", "host40", ..., "host329"]`.
* What updates must be applied to this resource? E.g., `{"labels": {"team": "products}}`.

This tutorial shows how to get started writing OPA policies using the Styra DAS.

## Goals

At the end of this hands-on-lab (HOL), you will have written, debugged, and tested
a policy for authorizing HTTP API calls into the CarInfoStore sample application.

The CarInfoStore application supports the following API calls:

| URL | Methods |
| --- | --- |
|/cars | GET/POST|
|/cars/{carid} |  GET/PUT/DELETE |
| /cars/{carid}/status | GET/POST |

The policy we want to enforce is:

* Anyone in the world may see the list of all cars
* Employees can read car details and car statuses
* Managers can create/modify/delete cars

Once you finish this tutorial, you will be familiar with:

* Writing HTTP API policies
* Running policies with the Styra DAS
* Debugging policy with the Styra DAS
* Testing policy



## Prerequisites

An account at TENANT.styra.com.

## Steps

### 1. Experiment with Preview

Go to TENANT.styra.com and add a new System on the left-hand navigation by clicking the Plus.  Use the following values:

* **System type**: Custom
* **System name**: <your name> microservices  (should be unique so you can identify it as yours)

Once this finishes, the DAS shows you some install instructions.  You can ignore those since in this tutorial we are only interested in policy authoring.  There are other tutorials for running an OPA on a live system and downloading policy to it.

On the left-hand side, open the system and the `Rules` file underneath it.  Replace what's there with the following rules:

```rego
package rules

default hello = false

hello {
  input.message == "world"
}
```

Provide an `input` by clicking the Preview button and copying in the following JSON:

```json
{
  "message": "world"
}
```

Click the Preview button again to see the result of evaluating the rules.

```json
{
  "hello": true
}
```

Add another statement to the policy.

```ruby
at_least_world {
  startswith(input.message, "wor")
  endswith(input.message, "ld")
}
```

Evaluate everything again by clicking the Preview button.  (Make sure to de-select what you copied in before re-evaluating.)  The Output window displays values for all of the variables in the policy.

```json
{
  "at_least_world": true,
  "hello": true
}
```

Now change the Input to the following JSON and Refresh the evaluation.

```json
{
  "message": "world is not enough"
}
```

Neither of the rules evaluated to true.

```json
{
  "hello": false
}
```

The reason `hello` is false is obvious.  But if we pretend we don't know why `at_least_world` failed, we need to discover which of the two conditions failed.  To do that, first highlight `startswith(input.message, "wor")` and click Preview to re-evaluate.

```json
true
```

Now highlight both lines in the body and evaluate again.  This time the result is empty, meaning `undefined`.  You can highlight anything OPA treats as a query and ask for an evaluation of it.  That includes global variables/rule names, individual expressions like `startswith(input.message, "wor")`, and blocks of expressions like the body of a rule.  This ability to evaluate parts of a rule is invaluable for debugging.



### 2. Create a data file and a policy module

Replace everything in that `Rules` file with the beginnings of an authorization policy for the cars application.  By default the policy rejects all requests.

```shell
package rules

default allow = false
```

Now create a variable `users` assigned to JSON data that has information about the employees in the organization,
who are the ones using this application.  Add it to the same file.

```shell
users := {
    "alice":   {"manager": "charlie", "title": "salesperson"},
    "bob":     {"manager": "charlie", "title": "salesperson"},
    "charlie": {"manager": "dave",    "title": "manager"},
    "dave":    {"manager": null,      "title": "ceo"}
}
```

Sanity check that everything is working by clicking the Preview button to ask for the entire module.  You will see that `allow` is `false` and the `users` are as defined above:

```json
{
  "allow": false,
  "users": {
    "alice": {
      "manager": "charlie",
      "title": "salesperson"
    },
    "bob": {
      "manager": "charlie",
      "title": "salesperson"
    },
    "charlie": {
      "manager": "dave",
      "title": "manager"
    },
    "dave": {
      "manager": null,
      "title": "ceo"
    }
  }
}
```


### 3. Start CarInfoStore Policy

The policy we want to write for the CarInfoStore application limits who can run each of the RESTful APIs.  Below is the list of who is authorized to run which API.

| API | User |
| --- | --- |
| GET /cars | Anyone |
| POST /cars | Only Managers |
| GET /cars/{carid} | Any Employee |
| PUT /cars/{carid} | Only Managers |
| DELETE /cars/{carid} | Only Managers |
| GET /cars/{carid}/status | Any Employee |
| POST /cars/{carid}/status | Any Employee |

Imagine that the input passed into OPA has the fields `method`, `path`, and `user`, e.g.

```json
{
  "method": "GET",
  "path": ["cars", "car17", "status"],
  "user": "alice"
}
```

Write a rule that allows everyone to GET /cars.  We've gotten you started; you just need to fill in the missing condition.

```ruby
allow {
  input.path == ["cars"]
  ...
}
```

Now supply the following as Input and click Preview to see that `allow` evaluates to `true`.

```json
{
  "method": "GET",
  "path": ["cars"],
  "user": "alice"
}
```

Try a `POST` for Input to ensure you get `false`.

```json
{
  "method": "POST",
  "path": ["cars"],
  "user": "alice"
}
```

Now that you're satisfied with your policy, click the Publish icon (the left-arrow next to the trashcan) to make it live.


### 4. Write some tests

As you continue writing policy you'll want to rerun the tests you did manually to verify they all pass.  Use the unit-test framework to record and run your tests.  On the left-hand-side click on the `Tests` file and replace what's there with the following statements.

```shell
package test
import data.rules

test_car_read_positive {
    in = {
       "method": "GET",
       "path": ["cars"],
       "user": "alice"
    }
    rules.allow == true with input as in
}

test_car_read_negative {
    in = {
       "method": "GET",
       "path": ["nonexistent"],
       "user": "alice"
    }
    rules.allow == false with input as in
}
```

Notice that in the test file we are importing the package where we have all of our `allow` statements.
The main part of each test is a line like `rules.allow == true with input as in`.  That's saying to run the `allow` statements from the `data.rules` package and check if the result is true.

Evaluate the tests by going back to the `Rules` file and clicking the `Validate` button.  Once you're happy, click the Publish button (the left-arrow next to the trashcan) on both the `Rules` file and the `Tests` file.


### 5. Finish the Policy

So far we've implemented the first row in the table below.  Now it's time to fill out the remainder of the policy.  In all of the remaining rows in the table, we need to grant access either to `Only Managers` or to `Any Employee`.  So before we write more `allow` statements, we will create helpers that tell us when the user is a manager or an employee.

| API | User |
| --- | --- |
| GET /cars | Anyone |
| POST /cars | Only Managers |
| GET /cars/{carid} | Any Employee |
| PUT /cars/{carid} | Only Managers |
| DELETE /cars/{carid} | Only Managers |
| GET /cars/{carid}/status | Any Employee |
| POST /cars/{carid}/status | Any Employee |

First we define a helper that tells us when the user is an employee.  To do that, we need to reference the user data from earlier, which is shown below.

```json
users := {
    "alice":   {"manager": "charlie", "title": "salesperson"},
    "bob":     {"manager": "charlie", "title": "salesperson"},
    "charlie": {"manager": "dave",    "title": "manager"},
    "dave":    {"manager": null,      "title": "ceo"}
}
```

Let's say that every person who shows up in the `users` data is an employee.  To decide whether a user is an employee, we simply check that `input.user` is a key in the `users` data.   Fill out the missing logic below.

<!-- To check that `input.users` is a key in the dictionary `users`, we simply try to look up its value: `users[input.users]`.  If the key does not exist, the result is `undefined`.  To give the name `user_is_employee` to this bit of logic, we write the statement below. -->

```rego
user_is_employee {
  ...
}
```

`user_is_employee` is like a function or variable in a traditional programming language.  Anytime we need to use that logic, we use the name `user_is_employee` instead.

Likewise, let's say that every user whose title is not `salesperson` is a manager.  Fill out the missing logic below.

```rego
user_is_manager {
   ...
}
```

Add both those helper definitions to your policy, and then write the logic that implements the second row in the table above: that only managers should be able to create new cars.

```rego
allow {
    # only managers can create a new car
    ...
}
```

Add a couple of tests that the new statement works and then run them by clicking `Validate` on the `Rules` file.

```ruby
test_car_create_negative {
    in = {
        "method": "POST",
        "path": ["cars"],
        "user": "alice",
    }
    rules.allow == false with input as in
}

test_car_create_positive {
    in = {
        "method": "POST",
        "path": ["cars"],
        "user": "charlie",
    }
    rules.allow == true with input as in
}
```

To finish the policy we need to write the logic for the remaining rows in the table above along with tests.  Write the logic that allows only employees to `GET /cars/{carid}`.  You can assume that `input.path` comes in as an array, e.g. `["cars", "id789-932"]`.

```rego
allow {
    # only employees can GET /cars/{carid}
    ...
}
```



# Wrap Up

At this point, you've written a policy for the CarInfoStore application and have a complementary test-suite to go along with it.  You've learned

* How to write HTTP authorization policies with the Styra DAS
* How to write tests and run them with the Styra DAS

