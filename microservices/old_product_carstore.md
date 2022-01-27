# Tutorial: Scaling your application with N Instances of OPA

The Open Policy Agent lets you offload authorization decisions from any kind of software system.  styra.com lets you manage the policies and decisions made by many different OPAs.  For example, OPA can make authorization decisions such as:

* Which microservice APIs are authorized
* Which fields a GUI is authorized to show
* Which operations an application is authorized to perform on behalf of a user
* Which servers a user may connect to
* Which Kubernetes workloads may be deployed to production
* Which database tables users may query or update
* Which files users may read/write
* Which topics may be published/subscribed
* Which changes to the public cloud are permissible

The more instances of OPA you are using, the more important it becomes to use styra.com to centrally administer the policies and context those OPAs require.   While eventually you may end up with many instances of OPA because you use OPA to solve many of the different use cases listed above, in this tutorial we focus on a single use-case where you end up with many instances of OPA: microservice/application authorization.

Microservice/application authorization often requires many different instances of OPA because for high-availability and performance, you want to run OPA next to every application/microservice instance.  For scale-out applications, this can lead to hundreds or thousands of OPA instances.  Styra.com makes it easy to distribute policy and context to all of those OPA instances and gain both management and decision-making visibility into all of those OPAs.


## Goals

OPA instances scale up and down right alongside your application, which presents a challenge for control, visibility, and audit.  The number of instances may be large and may change dynamically.
In this tutorial you will learn how to use Styra.com to help you administer a large and dynamically changing collection of OPAs.  In particular you will learn:

* How to author policy using `TENANT.styra.com` and distribute that policy across many agents.
* How to look up which agents are up-to-date and which are out-of-date.
* How to inspect what policy decisions those agents are making.
* How to use a `styra-agent` as a drop-in replacement for OPA. `styra-agent` is a thin wrapper around OPA that provides additional functionality beyond OPA.  You can use OPA > 0.8.0 if you prefer, though the configuration instructions will be different than are shown in this tutorial.

The tutorial that follows extends the HTTP API tutorial in styra.com's documentation (see [docs](https://TENANT.styra.com/v1/docs/tutorial/http-api.html)).  In particular, this tutorial shows you how to set up a cluster of 2 HTTP application instances, where each app instance will connect to a dedicated  `styra-agent`. The local styra-agents take care of the policy synchronization with `TENANT.styra.com`
and report back any authorization decisions made.

## Prerequisites

1. You will need an account at `TENANT.styra.com`
1. docker and docker-compose


## Step 1: Create an API token for TENANT.styra.com
You will need an API token so that the styra-agent you run as a drop-in replacement for OPA can connect to `TENANT.styra.com`.

1. Go to TENANT.styra.com > ☰ > API Tokens
1. Click "Add API Token" at the bottom of the dialog and enter the following settings
   * **Pathname**: enter a name you haven't used for a service account: `<your-name>/nopa`
1. Click OK
1. Copy the Secret (you won't be able to retrieve it after this step)
1. Click Done


## Step 2: Deploy N styra-agents on your machine

To simulate a cluster of http servers we will start 2 simple http servers via docker compose. One server will run at **port 5000** and the second one at **port 6000**, each will connect to dedicated `styra-agent` (one at **port 8181** and the other at **port 9191**) responsible to making the authorization decisions.  Create a file called **docker-compose.yml** and include the following text.

Below, replace

* `<YOUR_API_TOKEN_FROM_STEP_1>` with the token you copied in Step 1.
* `<YOUR_NAME>` with your first name.

**docker-compose.yml**
```shell
version: '2'
services:
  opa1:
    image: styra/agent:latest
    ports:
      - 8181:8181
    command:
      - "styra-agent"
      - "run"
      - "--server"
      - "--log-level=debug"
      - "--policy=<YOUR_NAME>/httpapi/authzN"
      - "--sync-max-wait=20s"
    environment:
      - STYRA_TOKEN=<YOUR_API_TOKEN_FROM_STEP1>
      - STYRA_CUSTOMER_ID=<YOUR_TENANT>
  api_server1:
    image: openpolicyagent/demo-restful-api:latest
    ports:
      - 5000:5000
    environment:
      - OPA_ADDR=http://opa1:8181
      - POLICY_PATH=/v1/data/<YOUR_NAME>/httpapi/authzN
  opa2:
    image: styra/agent:latest
    ports:
      - 9181:8181
    command:
      - "styra-agent"
      - "run"
      - "--server"
      - "--log-level=debug"
      - "--policy=<YOUR_NAME>/httpapi/authzN"
      - "--sync-max-wait=20s"
    environment:
      - STYRA_TOKEN=<YOUR_API_TOKEN_FROM_STEP1>
      - STYRA_CUSTOMER_ID=<YOUR_TENANT>
  api_server2:
    image: openpolicyagent/demo-restful-api:latest
    ports:
      - 6000:5000
    environment:
      - OPA_ADDR=http://opa2:8181
      - POLICY_PATH=/v1/data/<YOUR_NAME>/httpapi/authzN
```

Then run `docker-compose up` to pull and run the containers.

At this point the local agents will start complaining that they cannot synchronize the `<YOUR_NAME>/httpapi/authzN` policy that we wanted to enforce locally (defined by the `POLICY_PATH` environment variable and the `--policy` styra-agent parameter).

```
...
opa2_1         | time="2018-02-20T00:43:04Z" level=error msg="Synchronization error." err="non-existing policy 'httpapi/authzN'"
opa1_1         | time="2018-02-20T00:43:04Z" level=error msg="Synchronization error." err="non-existing policy 'httpapi/authzN'"
...
```


## Step 3: Create the policy `authzN`

Go to https://TENANT.styra.com/policies and `ADD` a new Blank policy named `<YOUR_NAME>/httpapi/authzN` and copy-and-paste the policy below.

```ruby
package <YOUR_NAME>.httpapi.authzN

default allow = false

# Allow users to get their own salaries.
allow {
  input.method = "GET"
  input.path = ["finance", "salary", user]
  user = input.user
}

# Allow managers to get their subordinates' salaries.
allow {
  input.method = "GET"
  input.path = ["finance", "salary", user]
  manager_of[user] = input.user
}

# Manager of alice is bob...
# Manager of charlie is betty...
manager_of = {"alice": "bob", "charlie": "betty"}
```

## Step 4: Explore the policy editor

The Policy editor lets you evaluate the policy you just copied in.  Let's test a few inputs to see what the policy produces as output.  To do that:

1. Click the left-most icon on the upper-right corner of the policy editor
1. Add a JSON object to the window that pops up.  That JSON object will be assigned to `input` in the policy.
1. Click the eyeball icon from the upper-right corner of the policy editor
1. See the result of evaluating the entire policy.

As a first input, check if `alice` can see her own salary.  Put the following JSON object into the Input pop up window.

**input**

```json
{
  "user": "alice",
  "path": ["finance", "salary", "alice"],
  "method": "GET"
}
```

Now look at the output window.  We can see that `alice` is allowed to see her own salary because `allow` is `true` in the output evaluation.  By default, the policy editor asks OPA to evaluate everything in the policy, which is why the `result` contains `allow`, and `manager_of`.

**output**

```json
{
  "result": {
    "allow": true,
    "manager_of": {
      "alice": "bob",
      "charlie": "betty"
    }
  }
}
```

The policy editor lets you control what gets evaluated.  Using your mouse or cursor, highlight the `allow` keyword in one of the rules in the policy.  Now click the Refresh icon in the upper right corner of the output window (or click the eyeball twice).  Now you will see just the value of `allow`:

```json
true
```

Now use your mouse or the cursor to highlight the 2nd two lines of the first `allow` rule:

```ruby
  input.path = ["finance", "salary", user]
  user = input.user
```

Refresh the output.  The output format this time shows you all the variable assignments that make the text you selected true.  In this example, there is only 1 variable `user` and only 1 variable assignment that makes the selected text true (for the input you provided): `user` must be assigned `alice`.

```ruby
# Found 1 result
variable_bindings_by_result[result] = {"user":user} {
  result = "Result 1"
  user = "alice"
}
```

When you highlight text, the only restriction is that you highlight a well-formed fragment of the OPA policy.  If there are variables in what you highlighted, you will see all the values for those variables that make the highlighted text `true`; otherwise, it returns the value for what you highlighted.

Try other inputs to see if your policy does what you expect.

`styra.com` does not yet support OPA's test runner, but you can still write tests within the policy and run them with the evaluate functionality whenever you modify the policy.  For example, add the following test to the policy and check that the test returns `true`.

```ruby
test_allow {
   i = {"user": "alice", "method": "GET", "path": ["finance", "salary", "alice"]}
   allow = true with input as i
}
```

**Once you finish, click the Publish button to make your policy live.**

## Step 5: Check on agent health

Once the policy is published, the local agents will synchronize and report success.

```
opa2_1         | time="2018-02-20T00:51:19Z" level=info msg="Updating documents and policies (httpapi/authzN)"
...
opa1_1         | time="2018-02-20T00:51:19Z" level=info msg="Updating documents and policies (httpapi/authzN)"
```

Once you have many agents, you need a way to gain visibility into their health: which ones are up to date, are they experiencing heavy load, and so on.  `TENANT.styra.com` provides that kind of information from the `Agents` tab.

1. Go to `TENANT.styra.com` > Agents
1. Inspect your agents' health by answering the following questions:
   * How much memory is each of your agents using?
   * Do your agents have the latest policy?  (Look for a status of `Synced`)


## Step 6: Make policy context-aware with an external datasource

OPA policies let you make authorization decisions based on what is happening in the world.  In styra.com `Datasources` connect to the world to give policies visibility into what is happening.  For example we can create a custom datasource that pulls information from a user-controlled button specifying an outage. In this step we will create such a `Lockdown` datasource on https://TENANT.styra.com.

1. Go to https://TENANT.styra.com > Datasources > Add
1. Choose Custom Lockdown
1. Enter the following field
   * **Pathname**: `<YOUR_NAME>/lockdown`
1. Click Finish

Now we will modify the OPA policy so that the `httpapi/authzN` policy results will change depending the toggled state of the Lockdown datasouce.  In particular, the policy will ensure allow is `false` when the lock is toggled, which happens in case of an outage.

To do so replace the `<YOUR_NAME>/httpapi/authzN` policy we created above with the following Rego code. The new version of the policy includes the following lines:

* `import data.<YOUR_NAME>.lockdown` so that the policy can refer to `lockdown`.
* `lockdown.active = false` so that the rules enforce the lockdown policy.

```ruby
package <YOUR_NAME>.httpapi.authzN

import data.<YOUR_NAME>.lockdown

default allow = false

# Allow users to get their own salaries.
allow {
  lockdown.active = false
  input.method = "GET"
  input.path = ["finance", "salary", user]
  user = input.user
}

# Allow managers to get their subordinates' salaries.
allow {
  lockdown.active = false
  input.method = "GET"
  input.path = ["finance", "salary", user]
  manager_of[user] = input.user
}

# Manager of alice is bob...
# Manager of charlie is betty...
manager_of = {"alice": "bob", "charlie": "betty"}
```


In order to lock or unlock the lockdown datasource:
1. Go to https://TENANT.styra.com > Datasources
1. Click the `lock` symbol in the `Extra` column of the datasource named `<YOUR_NAME>.lockdown` you created in Step 6
1. A pop-up window will appear: press Activate or Deactivate to toggle the state of the lockdown datasource
1. To verify the state: Click the datasource path to read the state of the the lockdown datasource


Using the policy editor test that the policy works as expected

1. Unlock the lockdown datasource (as explained above), and verify the state has been changed: https://TENANT.styra.com/v1/data/<YOUR_NAME>/lockdown
1. Similar to Step 4 evaluate the policy with the following input and verify that the `allow` rule returns `true`
```json
{
  "user": "alice",
  "path": ["finance", "salary", "alice"],
  "method": "GET"
}
```
1. Act as a manager and lock the lockdown datasource (as explained above).
1. Re-evaluate the policy and confirm that `allow` will correctly return `false` for all users.

Spend a few minutes thinking about how you might use the lockdown datasource, or another user-controlled custom datasource in practice to schedule permissions changes.

## Step 7: Use your application

Before you continue, unlock the lockdown datasource, and verify that https://TENANT.styra.com/v1/data/<YOUR_NAME>/lockdown returns `active = false`.

Now let's verify the applications are getting the correct decisions from OPA/styra-agent.

Your results will depend on the external context defined in Step 5, but for this section we assume there are no external events that are blocking everyone's rights to see salary information.

Verify that `alice` can see her own salary.
```shell
> curl --user alice:password localhost:5000/finance/salary/alice
Success: user alice is authorized
> curl --user alice:password localhost:6000/finance/salary/alice
Success: user alice is authorized
```

Verify that `zed` cannot see `alice`'s salary:
```shell
> curl --user zed:password localhost:5000/finance/salary/alice
Error: user zed is not authorized to GET url /finance/salary/alice
> curl --user zed:password localhost:6000/finance/salary/alice
Error: user zed is not authorized to GET url /finance/salary/alice
```

Now let's verify that the lockdown datasource we defined in Stage 6 is also correctly propagated. To do so lock the lockdown datasource and see if it changes the authorizations above. Please note that the OPA agents will pull the policy and required context from TENANT.styra.com at regular intervals and it might take up to a minute for lockdown state to propagate to the agents.

## Step 8: Audit all authorization decisions

Let's now explore how `TENANT.styra.com` keeps a historical record of all the policy decisions made by agents, including the context and policy that was used to make that decision.

1. Go to `TENANT.styra.com` > Log
1. Find an entry for the `httpapi/authzN` policy.
1. Click on that entry and you will see the input and the policy decision made, as shown below.

Note: the decision logs might be uploaded every 2 minutes

```json
request:
{
  "method": "GET",
  "path": [
    "finance",
    "salary",
    "alice"
  ],
  "user": "bob"
}
response:
{
  "allow": true,
  "manager_of": {
    "alice": "bob",
    "charlie": "betty"
  }
}
```

Now imagine that you want to understand more about why that decision was made the way it was.  At some point you will be able to ask for an explanation as to why the decision was made right in the GUI, but right now the best way to understand the decision is to download the policies and load them into OPA to see for example a trace.

1. Click the cloud icon with the downarrow to download a bundle of policies to your desktop.
1. Untar the file
1. Load the file into OPA and run a trace on the query

Note: replace <YOUR_NAME> with your name below

```shell
$ cd path-to-untarred-download
$ docker run -it -v `pwd`:/authzN -w /authzN openpolicyagent/opa run .
> trace
> data.<YOUR_NAME>.httpapi.authzN.allow with input as {"user": "bob", "method": "GET", "path": ["finance", "salary", "alice"]}
...
```

Change the user to test the policy and type `exit` to exit the OPA shell.

## Wrap Up

Congratulations for finishing the tutorial!


You learned how to deploy and scale styra agents alongside your applications, and learned how styra's decentralized policy decisions are made.
