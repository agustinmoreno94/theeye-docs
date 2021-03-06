# Tasks API

[![theeye.io](../../images/logo-theeye-theOeye-logo2.png)](https://theeye.io/en/index.html)

## API URL

URL: `https://supervisor.theeye.io/task?access_token={token}&customer={organization_name}`

### Properties

| UI Property | Api Property | Type | Description |
| ----- | ----- | ----- | ----- |
| Name | name | string | name your task |
| Bots | host_id | string | select the host where the script will run |
| Script | script_id | string | select the script to be executed by the task |
| Tags | tags | strings array | tag your task so you can find quickly through the application. |
| Task Arguments | task_arguments | array | If the script played by the task is meant to receive parameters you can set them from here. Mind the order as it will be used by the script. _Fixed_, _options_, and _input_ arguments are allowed. _Input_ and _options_ arguments will be asked to the user for execution. _Fixed_ arguments will not be displayed to the user at execution time. |
| Copy Task |  |  | select an already created task as template |
| Run As | run_as | string | write down any extra command or argument needed to run the script. Windows users must declare here which interpreter to use. Linux users could prepend sudo |
| Description | description | text | describe your task. What does it do, what's the expected output after execution |
| ACL's | acls | array | select who can view your task \(what can be done with the task depends on the user role\) |
| Triggered by | triggers | array | If the task is part of a workflow, select what triggers the task. The task will be triggered by the resource you selected here. |
| Trigger on-hold time | grace_time | number | enter the time period TheEye should wait before running the task. _No wait / Cancelation_ can be selected which means the task will run inmediately after triggered. \(only applicable for triggered tasks\). **To cancel the task execution during the grace period, go to tasks panel, expand the task and delete the schedule created by the trigger.** |
| Execution Timeout | timeout | number | This is the number of seconds the Bot will wait for the script to complete the execution. If the timeout is exceeded the Bot will try to terminate(kill) the script, sending SIGTERM/SIGKILL signal |
| Multitasking | multitasking | boolean | enable or disable parallel execution of the task. When this is enable assigned bot will be able to run multiple instances of the Job at same time. this is important to check when running DesktopBots |
| Environment (env) | env | string | Define extra environment variables that will be present during script execution |


## Examples

### Tasks

*Task execution timeout*

The default execution timeout for tasks is 10 minutes.

Now it's not possible to change the timeout via API. To modify the timeout for a task contact us.



### List all

*Resquest*
```bash
customer=$THEEYE_ORGANIZATION_NAME

curl -sS "https://supervisor.theeye.io/${customer}/task?access_token=$THEEYE_TOKEN"
```

### Search task id by name

*Request*
```bash
customer=$THEEYE_ORGANIZATION_NAME
taskid=$(echo $THEEYE_JOB | jq -r '.task_id')
echo "task id: ${taskid}"

result=$(curl -sS "https://supervisor.theeye.io/${customer}/task/${taskid}?access_token=${THEEYE_TOKEN}")

echo $result | jq -c '. | {"name": .name, "id": .id, "hostname": .hostname}'
```

*Response*
Returns a json array with tasks, id and hostname:
```json
[
  {
    "name": "Get IP Address",
    "id": "5b1c65ee3c32bb1100c2920a",
    "hostname": "Apache3"
  },
  {
    "name": "Get IP Address",
    "id": "5b1c65ee3c32bb1100c29210",
    "hostname": "Apache1"
  },
  {
    "name": "Get IP Address",
    "id": "5b1c65efd421031000213bb8",
    "hostname": "Apache4"
  },
  {
    "name": "Get IP Address",
    "id": "5b1c65efd421031000213bc6",
    "hostname": "Apache2"
  }
]
```

### List and timeout
(Timeout = null) means that the timeout is set to default (10 minutes).

```bash
#!/bin/bash

customer=$1
access_token=$THEEYE_TOKEN
supervisor=$2
if [ $2 == "" ]; then supervisor='https://supervisor.theeye.io' ; fi
if [ $# -ne 2 ]; then echo "missing parameters" ; exit ; fi

data=$(curl -s ${supervisor}/${customer}/task?access_token=${access_token})

echo "${data}" | jq -j '.[] | "id: ", .id, "\ttask: ", .name, "\ttimeout: ", .timeout, "\n"'
```


## Run task by id

For a **Task** to run, internally, a new **Job** is created and added to the queue. If you want to run a **task** you only need to create a **Job** manually and supply the task ID \(and task options\) you want to run.

Note that in this example we are executing a task with no arguments, task_arguments is an empty array.
To execute a task with arguments, provide the list of ordered arguments in the "task_arguments" parameter.
Each argument must be a valid JSON escaped string.

This is easily done with a POST request to the Job endpoint API.

There are two methods available.

### Task and Workflow execution payload via API

When creating Task Jobs via API you will have to provide the Task ID and the Task Arguments.

```javascript
{
  // (required)
  task: "task id",
  // (required only if task has arguments)
  task_arguments: []
}
```

### Using task secret key. Integration Feature \(recommended\)

All tasks have a **secret key** which can be used to invoke them directly via API. The secret key provides access to the task it belongs **and only to that task**. **Secret keys** can be revoked any time by just changing them, which makes this the preferred method for it's implicit security.

#### CURL sample Request


```bash
echo $TASK_SECRET
task_id=$TASK_ID
task_secret_key=$TASK_SECRET
customer=$(echo $THEEYE_ORGANIZATION_NAME | jq -r '.')


curl -i -sS \
  --request POST \
  --header "Accept: application/json" \
  --header "Content-Type: application/json" \
  --data "{\"task_arguments\":[]}" \
  "https://supervisor.theeye.io/job/secret/${task_secret_key}?customer=${customer}&task=${task_id}"
```

#### Task HTML Button

This technique could be combined with an HTML form to generate an action button. This results very handy when it is needed to perform actions from email bodies or static web pages.

```html
<html>
  <div>
    <form action="http://api.theeye.io/job/secret/b9d0b89439866987e818d5299ba61df0a32ccb38d81d996f46b9ce7af0720058" method="POST">
      <input type="hidden" name="customer" value="organization_name">
      <input type="hidden" name="task" value="5c49157cdb340a4d0444195a">
      <input type="hidden" name="task_arguments[]" value="arg1">
      <input type="submit" value="ACEPTO">
    </form>
  </div>
</html>

```

### Using workflow secret key.

You can invoke a **Workflow** using its **secret key**.

#### CURL Sample Request

```bash
workflow=$WORKFLOW_ID
secret=$SECRET_WORKFLOW
customer=$(echo $THEEYE_ORGANIZATION_NAME | jq -r '.')

echo $THEEYE_ORGANIZATION_NAME

curl -i -sS -X POST "https://supervisor.theeye.io/workflows/${workflow}/secret/${secret}/job" \
  --header 'Content-Type: application/json' \
  --data '{"customer":"'${customer}'","task_arguments":["'${parametro-1}'","'${parametro-2}'","'${parametro-3}'"]}'
```

#### Tasks Workflow HTML Button

Can also use an HTML form for the same purposes.

```html
<html>
<div>
 <form action="http://api.theeye.io/workflows/5d5ee18e809501000fb1435b/secret/b9d0b89439866987e818d5299ba61df0a32bbb38d81d996f46b9ce7af0720058" method="POST">
   <input type="hidden" name="customer" value="organization_name">
   <input type="hidden" name="task_arguments[]" value="arg1">
   <input type="submit" value="ACEPTO">
 </form>
</div>
</html>

```

### Using API integration token (beta)

Integration Tokens can be obtained only by admin users.

<h2 style="color:red"> WARNING ! Integration Tokens has full admin privileges. Use with care</h2>

Accessing to the web interfaz *Menu > Settings > Credentials > Integration Tokens*.


#### **Request:**

```bash
task_id=$TASK_ID
access_token=$ACCESS_TOKEN
customer=$(echo $THEEYE_ORGANIZATION_NAME | jq -r '.')


curl -X POST \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -d "{\"task_arguments\":[]}" \
  "https://supervisor.theeye.io/job?access_token=${access_token}&customer=${customer}&task=${task_id}"

```

> NOTE 1: customer is REQUIRED. can also be included in the body as "customer"  

> NOTE 2: access token can be provided vía Authorization header \( Authorization: Bearer ${token} \)


#### **Response:**

```json
{
  "task_arguments_values": [],
  "_type": "ScriptJob",
  "output": null,
  "creation_date": "2019-05-15T16:23:56.393Z",
  "last_update": "2019-05-15T16:23:56.403Z",
  "task": {
    "id": "************************",
    "task_arguments": [],
    "output_parameters": []
  },
  "task_id": "************************",
  "host_id": "************************",
  "host": {
    "enable": true,
    "_id": "************************",
    "customer_name": "demo",
    "customer_id": "************************",
    "creation_date": "2018-10-05T12:15:48.875Z",
    "last_update": "2019-05-15T16:23:54.932Z",
    "hostname": "************************",
    "ip": "127.0.0.1",
    "os_name": "Linux",
    "os_version": "4.15.0-22-generic",
    "agent_version": "v0.15.2",
    "integrations": {
      "ngrok": {
        "active": false,
        "url": "",
        "last_job": null,
        "last_job_id": "",
        "last_update": "2018-10-05T12:15:48.877Z"
      }
    },
    "id": "************************"
  },
  "name": "testing task",
  "lifecycle": "ready",
  "state": "in_progress",
  "customer_id": "************************",
  "customer_name": "demo",
  "user_id": "************************",
  "user": {
    "id": "************************",
    "username": "************************",
    "email": "info+************************@theeye.io"
  },
  "notify": true,
  "origin": "user",
  "script_id": "************************",
  "script_runas": "",
  "id": "************************"
}
```

The API response is a new created job. We can save the job id and use it later to query the job status.
In this example the id is "************************"


### Querying job status

Once the job is created we can query it's status using the ID of the job. we can also fetch all the jobs and then filter the response.

#### **Request:**


```shell

access_token=$THEEYE_ACCESS_TOKEN
customer=$(echo $THEEYE_ORGANIZATION_NAME | jq -r '.')
job_id=$1

echo "using: $customer"

curl -sS "https://supervisor.theeye.io/${customer}/job/${job_id}?access_token=${access_token}"| jq -r .

```

#### **Response:**


```json
{
  "task_arguments_values": [],
  "_type": "ScriptJob",
  "result": {
    "code": 0,
    "log": "\nnormal\n",
    "lastline": "normal",
    "signal": null,
    "killed": false,
    "times": {
      "seconds": 0,
      "nanoseconds": 7156981
    }
  },
  "output": null,
  "creation_date": "2019-05-15T15:55:30.914Z",
  "last_update": "2019-05-15T15:55:38.194Z",
  "task": {
    "id": "5cdb424d7dbdc0000fd3112a",
    "task_arguments": [],
    "output_parameters": []
  },
  "task_id": "5cdb424d7dbdc0000fd3112a",
  "host_id": "5bb755f42f78660012bdd9a6",
  "host": "5bb755f42f78660012bdd9a6",
  "name": "testing task",
  "lifecycle": "finished",
  "state": "normal",
  "customer_id": "5562573181a2334537425eaf",
  "customer_name": "demo",
  "user_id": "5bf81d18ca5e7e000f80f8ef",
  "user": {
    "id": "demouserid",
    "username": "demouser",
    "email": "demouser@theeye.io"
  },
  "notify": true,
  "origin": "user",
  "script_id": "5b8d45f31a047f12005fdb61",
  "script_runas": "",
  "trigger_name": "success",
  "id": "5cdc367240f0bb000f8e9653"
}
```
