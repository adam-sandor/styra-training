# Tutorial: Iterative Rules with API Authorization and Styra DAS

The Open Policy Agent (OPA) is an open source, general-purpose policy engine that enables unified, context-aware policy enforcement across the entire stack.

OPA provides a high-level **declarative language** for authoring policies and
simple APIs to answer policy queries. Using OPA, you can offload policy
decisions from your service such as:

* Should this API call be allowed? E.g., `true` or `false`.
* How much quota remains for this user? E.g., `1048`.
* Which hosts can this container be deployed on? E.g., `["host1", "host40", ..., "host329"]`.
* What updates must be applied to this resource? E.g., `{"labels": {"team": "products}}`.

This tutorial is intended for people with a firm grasp of basic policy authoring.
The goal is to demonstrate how to write policy that requires searching through data.


## Goals

This tutorial is an extension of Policy Authoring 1, where the overall goal is the
same: a complete HTTP API policy for the CarInfoStore sample application.  This tutorial
is different because the user data you must use requires search in order to write the
policy.

Remember that the CarInfoStore application supports the following API calls:

| URL | Methods |
| --- | --- |
|/cars | GET/POST|
|/cars/{carid} |  GET/PUT/DELETE |
| /cars/{carid}/status | GET/POST |

The policy we want to enforce (which differs slightly from Policy Authoring 1) is:

* Anyone in the world may see the list of all cars
* Employees can read car details and car statuses, if they are the administrators of the car resource
* Managers can create/modify/delete cars, if they are administrators of the car-status resource


## Prerequisites

An account at TENANT.styra.com.

## Steps

### 1. Get started


Go to TENANT.styra.com and add a new System on the left-hand navigation by clicking the Plus.  Use the following values:

* **System type**: Custom
* **System name**: <your name> microservices  (should be unique so you can identify it as yours)

On the left-hand side, open the system and the `Rules` file underneath it.  Replace what's there with the following rules:

```rego
package rules

###################
# Policy

default allow = false

# /cars
allow {
    # everyone in the world can see the car listing
    input.path == ["cars"]
    input.method == "GET"
}

allow {
    # only managers can create a new car
    input.path == ["cars"]
    input.method == "POST"
    user_is_manager
}

# /cars/{carid}
allow {
    # only employees can see car details
    some carid
    input.path = ["cars", carid]
    input.method == "GET"
    user_is_employee

}

allow {
    # only managers can update an existing car
    some carid
    input.path = ["cars", carid]
    input.method == "PUT"
    user_is_manager
}

allow {
    # only managers can delete a car
    some carid
    input.path = ["cars", carid]
    input.method == "DELETE"
    user_is_manager
}

# /cars/{carid}/status
allow {
    some carid
    input.path = ["cars", carid, "status"]
    input.method == "GET"
    user_is_employee
}

allow {
    some carid
    input.path = ["cars", carid, "status"]
    input.method == "POST"
    user_is_employee
}

######################
# Helpers

user_is_manager {
    false
}
user_is_employee {
    false
}
```

Now create a variable users assigned to JSON data that has information about the employees in the organization, who are the ones using this application.

```shell
users = {
    "manages": {
        "bob":     [],
        "charlie": ["alice", "bob"],
        "dave":    ["charlie"]
    },
    "titles": {
        "salesperson": ["alice", "bob"],
        "ceo": ["dave"],
        "vp": ["charlie"]
    }
}
```

Let's also start with the tests we finished in Policy Authoring 1.

```rego
package test
import data.rules

test_car_read {
    in = {
       "method": "GET",
       "path": ["cars"],
       "user": null,
       "body": null
    }
    rules.allow == true with input as in
}

test_car_status_read_negative {
    in = {
       "method": "GET",
       "path": ["cars", "car789", "status"],
       "user": null,
       "body": null
    }
    rules.allow == false with input as in
}


test_car_create_negative {
    in = {
        "method": "POST",
        "path": ["cars"],
        "user": "alice",
        "body": {"id": "123",
                 "model": "nissan",
                 "registeredVehicleId": "456",
                 "ownerId": "789"}
    }
    rules.allow == false with input as in
}

test_car_create_positive {
    in = {
        "method": "POST",
        "path": ["cars"],
        "user": "charlie",
        "body": {"id": "123",
                 "model": "nissan",
                 "registeredVehicleId": "456",
                 "ownerId": "789"}
    }
    rules.allow == true with input as in
}

test_car_status_create_negative {
    in = {
        "method": "POST",
        "path": ["cars", "car789", "status"],
        "user": "nonexistent",
        "carId": "car7890",
        "body": {"sensedAt": "2018/02/20",  # CarStatus
                "position": {
                    "latitude": 123,
                    "longitude": 456,
                    "height": 789
                },
                "mileage": 14000,
                "speed": 90,
                "fuel": 5.6}
    }
    rules.allow == false with input as in
}

test_car_status_create_positive {
    in = {
        "method": "POST",
        "path": ["cars", "car789", "status"],
        "user": "charlie",
        "carId": "car7890",
        "body": {"sensedAt": "2018/02/20",  # CarStatus
                "position": {
                    "latitude": 123,
                    "longitude": 456,
                    "height": 789
                },
                "mileage": 14000,
                "speed": 90,
                "fuel": 5.6}
    }
    rules.allow == true with input as in
}
```

Evaluate the tests by going back to the `Rules` file and clicking the `Validate` button.


### 2. Update a helper so that more of the original tests pass

The helpers defined above never return true.  Some of the tests will no longer work.  To verify, first check that some of the tests fail.

Let's look at the first helper: `user_is_manager`.  The manager data we have looks like this:

```json
"manages": {
    "bob":     [],
    "charlie": ["alice", "bob"],
    "dave":    ["charlie"]
}
```

This data says that:

* `bob` is the manager of no one
* `charlie` is the manager of `alice` and `bob`
* `dave` is the manager of `charlie`

A user is a manager if they are a key in the `manages` dictionary and they manage at least one person.  Replace the definition of `user_is_manager` appropriately.  You can count the elements in an array with the `count` builtin.

```rego
user_is_manager {
    ...
}
```

Run the tests again to see that an additional test passed.


### 3. Update the other helper so that all of the original tests pass

Now let's fix the other helper `user_is_employee`.  The data we need to define who is an employee is shown below.

```json
"titles": {
    "salesperson": ["alice", "bob"],
    "ceo": ["dave"],
    "vp": ["charlie"]
}
```

A user is an employee if they have some title.  It doesn't matter which title they have.  To check if an employee has some title, we need to search.  In particular, we need to iterate over all possible titles and then iterate over all employees with that title to see if any of them match `input.user`.  Iteration with OPA happens automatically if you replace concrete values (strings, numbers) with variables.

For example, to check if `input.user` has the title salesperson in the 0th array index, you would write: `input.user == users.titles["salesperson"][0]`.

To iterate over all array indexes for the `salesperson` title, you replace the `0` with a variable and use the keyword `some`: `some i; input.user == users.titles["salesperson"][i]`.

At this point, you know enough to write the definition for `user_is_employee` using our new data:

```rego
user_is_employee {
    ...
}
```

It often happens that you want to iterate over all values, but you don't want to worry about inventing a new variable.  OPA has a special variable (underscore `_`) that you can use as many times as you like, and OPA treats every occurrence as a separate variable.  Moreover, there is no need to use `some`.  Update your `user_is_employee` definition to use `_`.

```rego
user_is_employee {
    ...
}
```

Run the tests to see that they all pass.


### 4. Update the policy to handle Administrators, Part 1

The english policy we are trying to enforce requires that you must be an administrator to make changes.  (The previous policy had no requirements to be an administrator.)

* Anyone in the world may see the list of all cars
* Employees can read car details and car statuses, if they are the administrators of the car resource
* Managers can create/modify/delete cars, if they are administrators of the car-status resource

Below are tests that handle this extended policy.  Replace your current tests with this one.

```rego
package test

import data.rules

test_car_read {
    in = {
       "method": "GET",
       "path": ["cars"],
       "user": null,
       "body": null
    }
    rules.allow == true with input as in
}

test_car_status_read_negative {
    in = {
       "method": "GET",
       "path": ["cars", "car789", "status"],
       "user": null,
       "body": null
    }
    rules.allow == false with input as in
}


test_car_create_negative {
    in = {
        "method": "POST",
        "path": ["cars"],
        "user": "alice",
        "body": {"id": "123",
                 "model": "nissan",
                 "registeredVehicleId": "456",
                 "ownerId": "789"}
    }
    rules.allow == false with input as in
}

test_car_create_negative2 {
    in = {
        "method": "POST",
        "path": ["cars"],
        "user": "dave",
        "body": {"id": "123",
                 "model": "nissan",
                 "registeredVehicleId": "456",
                 "ownerId": "789"}
    }
    rules.allow == false with input as in
}

test_car_create_positive {
    in = {
        "method": "POST",
        "path": ["cars"],
        "user": "charlie",
        "body": {"id": "123",
                 "model": "nissan",
                 "registeredVehicleId": "456",
                 "ownerId": "789"}
    }
    rules.allow == true with input as in
}

test_car_status_create_negative1 {
    in = {
        "method": "POST",
        "path": ["cars", "car789", "status"],
        "user": "nonexistent",
        "carId": "car7890",
        "body": {"sensedAt": "2018/02/20",  # CarStatus
                "position": {
                    "latitude": 123,
                    "longitude": 456,
                    "height": 789
                },
                "mileage": 14000,
                "speed": 90,
                "fuel": 5.6}
    }
    rules.allow == false with input as in
}

test_car_status_create_negative2 {
    in = {
        "method": "POST",
        "path": ["cars", "car789", "status"],
        "user": "bob",
        "carId": "car7890",
        "body": {"sensedAt": "2018/02/20",  # CarStatus
                "position": {
                    "latitude": 123,
                    "longitude": 456,
                    "height": 789
                },
                "mileage": 14000,
                "speed": 90,
                "fuel": 5.6}
    }
    rules.allow == false with input as in
}

test_car_status_create_positive {
    in = {
        "method": "POST",
        "path": ["cars", "car789", "status"],
        "user": "charlie",
        "carId": "car7890",
        "body": {"sensedAt": "2018/02/20",  # CarStatus
                "position": {
                    "latitude": 123,
                    "longitude": 456,
                    "height": 789
                },
                "mileage": 14000,
                "speed": 90,
                "fuel": 5.6}
    }
    rules.allow == true with input as in
}

```

Run the tests to see that some of them fail.

To handle the new administrative conditions, add a helper condition to each of the `allow` statements that need it.  For example, for the API that creates a new car, we would add `user_is_car_admin` to the rule as shown below.


```ruby
allow {
    # only managers can create a new car
    input.path == ["cars"]
    input.method == "POST"
    user_is_manager
    user_is_car_admin
}
```

Now we need to define `user_is_car_admin`.  The relevant data is below.

```json
admin = {
    "cars": ["groupB"],
    "cars_status": ["groupA", "groupC"]
}

groups = [
    {"name": "groupA", "members": ["alice"]},
    {"name": "groupB", "members": ["bob", "charlie"]},
    {"name": "groupC", "members": ["charlie", "dave"]}
]
```


To know if a user is an administrator for the car resource, we need to look up the group or groups that are administrators, find that group from the `groups` array, and check if `input.user` is a member.  Below we get you started.  You can figure out the rest.


```ruby
user_is_car_admin {
   group_name := admin["cars"][_]
   <missing line>
   <missing line>
}
```

Here is a test you can run to see if you've got it right.

```ruby
test_user_is_car_admin {
    data.rules.user_is_car_admin with input as {"user": "bob"}
    data.rules.user_is_car_admin with input as {"user": "charlie"}
    not data.rules.user_is_car_admin with input as {"user": "alice"}
    not data.rules.user_is_car_admin with input as {"user": "dave"}
}
```

Once you get `user_is_car_admin` working,

* add the definition of `user_is_car_admin` to your policy file
* update the `allow` rule for `POST` on `["cars"]` to include `user_is_car_admin`
* update the `allow` rules for paths like `["cars", car_id]` and `["cars", car_id, "status"]` to include `user_is_car_admin`
* add `test_user_is_car_admin` to your test suite
* rerun the tests to see that more of them pass

### 5. Update the policy to handle Administrators, Part 2

There are 2 more policy rules that need to be updated: those that allow requests to `["cars", carid, "status"]`.  We will edit those statements so they require the user be an administrator for `cars_status`.

Similar to the last step, we will add a helper condition to each of the two `allow` rules that match the path `["cars", carid, "status"]`.  This time we will create a helper function that takes as an argument the name of the resource the user must be an administrator for.

For example, here is an updated rule that uses the helper function `user_is_admin("car_status")`.

```ruby
allow {
    some carid
    input.path = ["cars", carid, "status"]
    input.method == "POST"
    user_is_employee
    user_is_admin("cars_status")
}
```

We define that helper function just like we defined the helper in the last step, but we provide an argument to it.

```ruby
user_is_admin(resource) {
   group_name := admin[resource][_]
   some i
   groups[i].name == group_name
   groups[i].members[_] == input.user
}
```

Now you can finish updating to this new data format.

* Add the function above to your policy file
* Update the remaining 2 rules to call the `user_is_admin` function
* (Optional) Replace the `user_is_car_admin` helper with `user_is_admin("car")` in all the rules and test from the last step.
* Rerun the tests and see that now they all pass.


### 6. Comprehensions (Optional)

Let's imagine a PUT request to /cars/{id} updates the fields for the specified car ID.  Suppose the body of that request is a simple dictionary that lists all the changes the user is making, e.g. the following payload changes the `description` and `name` fields, leaving all the rest unchanged.

```json
{
 "description": "A very new description",
 "name": "toyota123"
 }
```


Moreover let's suppose that employees and managers are authorized to update different fields, as specified in the data below.

```rego
manager_fields = employee_fields | {"owner", "date-sold"}
employee_fields = {"name", "description"}
```

Now the task is to write a rule that allows a PUT to `/cars/{id}` as long as ALL of the fields in `input.body` are allowed to be updated by the persona running the API.


```rego
allow {
    some carid
    input.path = ["cars", carid]
    input.method == "PUT"
    user_is_employee
    <missing line>
    <missing line>
}
allow {
    some carid
    input.path = ["cars", carid]
    input.method == "PUT"
    user_is_manager
    <missing line>
    <missing line>
}
```

Hint: use a comprehension to compute the set of all fields that the user is changing and compare that to the fields the user is authorized to change.

<!--
    changing := {k | some k; input.body[k]}
    count(changing - employee_fields) == 0
-->



# Wrap Up

Congratulations for finishing the tutorial!  You've learned:

* How to use variables to iterate
* How to use the underscore for wildcard iteration
* How to write functions that take arguments to factor out common logic
* How to use comprehensions


