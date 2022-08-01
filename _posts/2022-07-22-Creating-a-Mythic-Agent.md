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
                "action":
                "ip":
                "os":
                "user":
                "host":
                "domain":
                "pid":
                "uuid":,
                "architecture":
        }
```
This message will populate metadata about the agent on the server, to provide more information about what commans are available and allow better use of OPSEC utilities.  

###  C2 Profiles/HTTP
The way that the agent and server sends these messages back and forth is dictated by the C2 profile selected when generating the payload.  Examples of C2 profiles include HTTP, raw TCP, or even methods as strange as steganography in twitter image uploads.  The freedom here is almost limitless.  C2 profiles have their own development process and they must be implemented in the agents to be used in.  Since there is much complexity included in this, I am going to use the standard HTTP C2 profile for this article.  
The HTTP profile uses POST requests to communicate.  On the server's side, it runs a web server that listens for POST requests, which the agent makes to check in, with the agent to server message being the data in the POST body.  The server takes that information, and responds with the server to agent message in the POST response.  This way, no ports need to be opened or servers spun up on the agent's host, providing quieter and more universal execution. 





 
