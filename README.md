---
title: "Tutorial2: Combining Nornir with pyATS"
date: "2029-05-03"
tags: ["nornir", "pyATS"]
---
This tutorial aims to provide a simple use-case for combining Nornir and pyATS together in order to profile your current network configurations and implement desired state - as specified in host variable definition files.

A summary of the workflow we will follow in this tutorial is as follows:
1) Setup basic connectivity such as SSH access for Out Of Band Management. This will represent our "clean" configuration.
2) Deploy the ```commit-golden.py``` script to write this configuration into Flash memory where it can be used to easily remove current network configurations whilst still leaving basic OOB SSH connectivity.
3) We shall then deploy the ```configure-network.py``` script which builds the network to our desired state as specified in our host definition files.
4) We shall then deploy the ```capture-golden``` bash script which effectively instruct pyATS to collect a "golden" profile of our desired state for future comparisons.
5) We can now periodically deploy the ```pynir2.py``` to ensure our network is compliant with desired state. If ```pynir2.py``` detects no discrepancy between current and desired state, the script simply terminates and reports all configurations are correct. However, should ```pynir2.py``` detect configuration drift, we are alerted and given the option to rollback. Should be choose No, ```pynir2.py``` terminates but will leave all current-config and diff artefacts for our inspection. Should we choose to rollback, ```pynir2.py``` will erase all current configurations from the device me performing a configuration replace operation and move our captured flash image into the running-configuration - leaving nothing but our OOB SSH connectivity. ```pynir2.py``` will then subsequently rebuild the network from scratch to its desired state as specified in our host definition files.

----------------------------------------------------------------------------------------------------


To begin, let&#39;s first look at the directory structure and setup:
```
ipvzero@MSI:~/Pynir2$ tree
.
├── capture-golden
├── commit-golden.py
├── config.yaml
├── configure_network.py
├── defaults.yaml
├── groups.yaml
├── host_vars
│   ├── S1.yaml
│   ├── S2.yaml
│   ├── S3.yaml
│   ├── S4.yaml
│   ├── S5.yaml
│   ├── S6.yaml
│   └── S7.yaml
├── hosts.yaml
├── pynir2.py
├── templates
│   ├── base.j2
│   ├── etherchannel.j2
│   ├── isis.j2
│   ├── trunking.j2
│   └── vlan.j2
└── testbed.yaml

2 directories, 22 files
```


As you can see we can our basic Nornir yaml files:

```hosts.yaml```
```groups.yaml```
```defaults.yaml```

Notice that there is also a ```testbed.yaml``` file to allow pyATS to connect into and profile the network. Next, you&#39;ll notice we also have two directories. The first one called ```host_vars``` which will house our OSPF host variables (note: you can use the ```hosts.yaml``` file instead, but I have chosen to create a separate directory to perform this task). The second is called ```templates``` which will house our Jinja2 templates.

Importantly, you&#39;ll notice a ```capture-golden``` file. This is a very simple bash script used to capture our &quot;golden&quot; snapshot of our desired network state as configured by the ```configure-network.py``` script. It simply executes a pyATS command. You can type this command by hand should you wish, but since the output directory has to remain the same since it will be referenced by the ```Pynir2.py``` script – for consistency, I have elected to execute it from a bash script to prevent me mistyping the output destination. Let&#39;s look inside to see what&#39;s going on:

```bash
#!/bin/bash
pyats learn config vlan --testbed-file testbed.yaml --output golden-config
```


As you can see the bash script simply tells pyATS to learn the network&#39;s running-configuration as well as vlan configuration and save the output into a directory called ```golden-config```. This directory will act as our future reference point.

Let&#39;s take a look inside the ```host_vars``` directory and see what our host variable definition files look like. For brevity, let&#39;s just look at ```S3.yaml```:

```
---
ISIS:
    nsap: 49.0001.0000.0000.0003.00
    level: level-1-2
    interfaces:
        g0/1:
            sub_iface: g0/1.10
            encapsulation: dot1q 10
        g0/2:
            sub_iface: g0/2.10
            encapsulation: dot1q 10
        loo0:
            ipaddr: 3.3.3.3 255.255.255.255


Etherchannel:
    interfaces:
        - gigabitEthernet1/2
        - gigabitEthernet1/3
    group: 1
    protocol: active

Trunked:
    trunk:
        interfaces:
            gig0/3:
                allowed_vlans: 10,20
            gig1/0:
                allowed_vlans: 30,40
            gig1/1:
                allowed_vlans: 50,60


VLAN:
  - number: 10
    name: TEN
  - number: 20
    name: TWENTY
  - number: 30
    name: THIRTY
  - number: 40
    name: FORTY
```

We have definitions for our network's desired state made up of our IS-IS, Etherchannel, Trunks and VLAN configs. This could,  of course, be expanded to include many more definitions but for the sake of the demo, I felt this was enough.

Next, let&#39;s look inside the templates directory and open our ```isis.j2``` file, as an example:

```
{% if 'ISIS' in host.facts %}
router isis
net {{ host.facts.ISIS.nsap }}
is-type {{ host.facts.ISIS.level }}
interface loopback0
ip address {{ host.facts.ISIS.interfaces.loo0.ipaddr }}
ip router isis
{% set john = host.facts.ISIS.interfaces %}
{% for key in john %}
interface {{ key }}
no switchport
no shut
{% if 'sub_iface' in john[key] %}
interface {{ john[key]['sub_iface'] }}
encapsulation {{ john[key]['encapsulation'] }}
ip unnumbered loopback0
ip router isis
{% endif %}
{% endfor %}
{% endif %}
```

This template will simply reference the Keys specificed in our host\_var yaml files and populate the template with their corresponding Values to build our desired IS-IS configuration.

As stated earlier, we have a ```configure-network.py``` script. This is the script we will use to first initially push our desired state onto the routers. This script simply pulls desired state from our ```host_vars``` and pushes them through our Jinja2 template onto the network. In other words, it does not remove old stale configs (like ```Pynir2.py``` will) so the assumption here is that we are working with a blank slate on the devices other than our basic OOB SSH configurations. Let&#39;s look inside the script:

```python
from nornir import InitNornir
from nornir.plugins.tasks.data import load_yaml
from nornir.plugins.tasks.text import template_file
from nornir.plugins.functions.text import print_result
from nornir.plugins.tasks.networking import netmiko_send_config

nr = InitNornir(config_file="config.yaml")

def load_vars(task):
    data = task.run(task=load_yaml,file=f'./host_vars/{task.host}.yaml')
    task.host["facts"] = data.result

def load_base(task):
    b = task.run(task=template_file, template="base.j2", path="./templates")
    task.host["base_config"] = b.result
    base_output = task.host["base_config"]
    base_send = base_output.splitlines()
    task.run(task=netmiko_send_config, name="Base Commands", config_commands=base_send)

def load_isis(task):
    i = task.run(task=template_file, template="isis.j2", path="./templates")
    task.host["isis_config"] = i.result
    isis_output = task.host["isis_config"]
    isis_send = isis_output.splitlines()
    task.run(task=netmiko_send_config, name="IS-IS Commands", config_commands=isis_send)

def load_ether(task):
    e = task.run(task=template_file, template="etherchannel.j2", path="./templates")
    task.host["ether_config"] = e.result
    ether_output = task.host["ether_config"]
    ether_send = ether_output.splitlines()
    task.run(task=netmiko_send_config, name="Etherchannel Commands", config_commands=ether_send)


def load_trunking(task):
    t = task.run(task=template_file, template="trunking.j2", path="./templates")
    task.host["trunk_config"] = t.result
    trunk_output = task.host["trunk_config"]
    trunk_send = trunk_output.splitlines()
    task.run(task=netmiko_send_config, name="Trunk Commands", config_commands=trunk_send)


def load_vlan(task):
    v = task.run(task=template_file, template="vlan.j2", path="./templates")
    task.host["vlan_config"] = v.result
    vlan_output = task.host["vlan_config"]
    vlan_send = vlan_output.splitlines()
    task.run(task=netmiko_send_config, name="VLAN Commands", config_commands=vlan_send)


yaml_targets = nr.filter(all="yes")
yaml_results = yaml_targets.run(task=load_vars)
base_targets = nr.filter(all="yes")
base_results = base_targets.run(task=load_base)
isis_targets = nr.filter(routing="yes")
isis_results = isis_targets.run(task=load_isis)
ether_targets = nr.filter(etherchannel="yes")
ether_results = ether_targets.run(task=load_ether)
trunk_targets = nr.filter(trunking="yes")
trunk_results = trunk_targets.run(task=load_trunking)
vlan_targets = nr.filter(vlan="yes")
vlan_results = vlan_targets.run(task=load_vlan)
print_result(yaml_results)
print_result(base_results)
print_result(isis_results)
print_result(ether_results)
print_result(trunk_results)
print_result(vlan_results)
print_result(base_results)
```


This script will pull information from the desired state specified in our ```host_vars``` yaml files, save the information and use those values to build our configurations based on the syntax specified in our Jinja2 template. Nornir then invokes Netmiko to push those configurations out to all of our respective devices in the network. Notice each configuration is filtered to target devices only the relevant configurations should be pushed. 

Let's begin the workflow.

After initially configuring our username, password, SSH configuration and management IP addresses, we first execute our ```commit-golden.py``` script to commit those basic reachability configurations to Flash memory:


![alt text](https://github.com/IPvZero/Pynir2/blob/master/images/1.png?raw=true)

With our desired state now present on the network, let&#39;s immediately use pyATS to build a detailed profile of that configuration and grab our &quot;golden&quot; snapshot. Let&#39;s execute the ```capture-golden``` script:

![alt text](https://github.com/IPvZero/Nornir-Blog/blob/master/images/7.png?raw=true)

pyATS has successfully profiled our desired state and you will notice the addition of a new directory called ```desired-ospf``` which houses of all of our detailed OSPF information for each device. 
Now that we have pushed our desired state and successfully created a snapshot for future comparison, let&#39;s look at the main script which we will use for our OSPF management going forward, ```Pynir.py```. The script is relatively long so let&#39;s break it down into sections. First we begin with our imports - and I have also included a Pyfiglet banner for purely aesthetic purposes (who doesn&#39;t like to make their scripts pretty, right?).

```python
import os
import subprocess
import colorama
from colorama import Fore, Style
from nornir import InitNornir
from nornir.plugins.tasks.networking import netmiko_send_command
from nornir.plugins.functions.text import print_result, print_title
from nornir.plugins.tasks.networking import netmiko_send_config
from nornir.plugins.tasks.data import load_yaml
from nornir.plugins.tasks.text import template_file
from pyfiglet import Figlet

nr = InitNornir(config_file="config.yaml")
clear_command = "clear"
os.system(clear_command)
custom_fig = Figlet(font='isometric3')
print(custom_fig.renderText('pyNIR'))
```
Next, we create a custom function called ```clean_ospf```. The first challenge of the script was to find a way to strip away all OSPF configurations, should that be required. The problem with automating over legacy devices with no API capabilities, however, is that we are heavily reliant on screen-scraping – an inelegant and unfortunately necessary solution. To do so, I made the decision to use Nornir to execute a ```show run | s ospf``` on all devices, saved the resulting output, and began screen-scraping to identify digits in the text. The aim here was to identify any OSPF process IDs which could then be extracted and used to negate the process by executing a ```no router opsf```; followed by the relevant process ID. The challenge here is that the show command output would also include area ID information – and OSPFs most common area configuration is for area 0. Of course ```router ospf 0``` is not a legal command, so in order to avoid this I included a conditional statement that would skip over and ```continue``` past any number zeros in the output. The second challenge would be avoiding needless repetition. Should OSPF be configured via the interfaces, the resulting show output could, for example, have multiple: ```ip ospf 1 area 0```
```ip ospf 1 area 0```.
Parsing out this information could lead to the script executing multiple ```no router ospf 1```; commands which is, of course, unnecessary. To avoid this, I elected to push all output into a python list, and from there remove all duplicates. There is still, however, an inefficiency given that the show output could, for example, show a multi-area OSPF configuration all within the same process. This could result in a script seeing an ```ip ospf 1 area 5``` configuration and attempting to execute a superfluous ```no router ospf 5```. However, given that the script has protections against repetitive execution, and that routers will have limited areas configured per device (maybe 3 different areas at most per device, if at all), I made the decision that this was an acceptable inefficiency. Like I say, there is nothing elegant about screen-scraping and sometimes a 90% solution is better than no solution:

```python
def clean_ospf(task):
    r = task.run(task=netmiko_send_command, name="Identifying Current OSPF", command_string = "show run | s ospf")
    output = r.result
    my_list = []
    num = [int(s) for s in output.split() if s.isdigit()]
    for x in num:
        if x == 0:
            continue
        my_list.append("no router ospf " + str(x))
    my_list = list(dict.fromkeys(my_list))
    for commands in my_list:
        task.run(task=netmiko_send_config, name="Removing Current OSPF", config_commands=commands)

    desired_ospf(task)
```

Now we have the ability to remove all current OSPF configuration, we create a custom function called ```desired_ospf```. This is almost identical to the earlier script and simply builds our configuration from ```host_vars``` definition files and pushes them through our jinja2 OSPF template and out to the devices:



```python
def desired_ospf(task):
    data = task.run(task=load_yaml, name="Pulling from Definition Files", file=f'./host_vars/{task.host}.yaml')
    task.host["OSPF"] = data.result["OSPF"]
    r = task.run(task=template_file, name="Building Desired State", template="ospf.j2", path="./templates")
    task.host["config"] = r.result
    output = task.host["config"]
    send = output.splitlines()
    task.run(task=netmiko_send_config, name="Implementing OSPF Desired State", config_commands=send)
```


The next part of the script is effectively what executes first and precedes our two custom functions (with will only execute upon certain conditions). Let&#39;s look at it. First we use the OS and Subprocess python modules to first execute the shell command ```pyats learn ospf --testbed-file testbed.yaml --output ospf-current``` to relearn the current state of the network&#39;s OSPF configs, and then run a diff between the current configs, and our previously saved golden config – ```pyats diff desired-ospf/ ospf-current –output ospfdiff```. We then read the output and search for the string ```Diff can be found```. If a difference is found, we are alerted to the discrepancy and offered the choice to rollback to our desired state.

```python
current = "pyats learn ospf --testbed-file testbed.yaml --output ospf-current"
os.system(current)
command = subprocess.run(["pyats", "diff", "desired-ospf/", "ospf-current", "--output", "ospfdiff"], stdout=subprocess.PIPE)
stringer = str(command)
if "Diff can be found" in stringer:
    os.system(clear_command)
    print(Fore.CYAN + "#" * 70)
    print(Fore.RED + "ALERT: " + Style.RESET_ALL + "CURRENT OSPF CONFIGURATIONS ARE NOT IN SYNC WITH DESIRED STATE!")
    print(Fore.CYAN + "#" * 70)
    print("\n")
    answer = input(Fore.YELLOW +
            "Would you like to reverse the current OSPF configuration back to its desired state? " + Style.RESET_ALL + "<y/n>: "
)
```

Should we answer &quot;y&quot; and affirm our decision to rollback, the script will first remove all current ospf and diff artefacts before calling our ```clean_ospf``` custom function, which, in turn, calls our ```desired-ospf``` function and prints the output.

```python
    if answer == "y":
        def main() -> None:
            clean_up = "rm -r ospfdiff ospf-current"
            os.system(clean_up)
            os.system(clear_command)
            nr = InitNornir(config_file="config.yaml")
            output = nr.run(task=clean_ospf)
            print_title("REVERSING OSPF CONFIGURATION BACK INTO DESIRED STATE")
            print_result(output)

        if __name__ == '__main__':
                main()
```

Should we choose **not** to rollback, however, and instead want to inspect those changes in detail - by selecting &quot;n&quot; the script simply terminates and leaves all artefacts for our inspection.

Lastly, should the script detect no changes between the current state of our OSPF network and the configurations in our golden capture – the script simply ends and informs us that all OSPF configurations are matching desired state:


```python
else:
    clean_up = "rm -r ospfdiff ospf-current"
    os.system(clean_up)
    os.system(clear_command)
    print("*" * 75)
    print(Fore.GREEN + "Good news! OSPF configurations are matching desired state!" + Style.RESET_ALL)
    print("*" * 75)

```
------------------------------------------------------------------------------------------------------------------------------
## Demo - Let's Break The Network!

Now that we understand the logic of the script, let&#39;s perform a demo and see the workflow in action. Now remember, we have already deployed our initial desired state and captured that snapshot using both the ```nornir-ospf.py``` and ```capture-golden``` scripts. Let&#39;s first &quot;break&quot; the network by adding some unwanted OSPF configurations on R8:

![alt text](https://github.com/IPvZero/Nornir-Blog/blob/master/images/14.png?raw=true)

Now the current state of our OSPF network does not match the configuration specified in our desired state. Let&#39;s run the ```Pynir.py``` script and see if it detects the change. ```Pynir.py``` first starts learning the current OSPF configurations:

![alt text](https://github.com/IPvZero/Nornir-Blog/blob/master/images/15.png?raw=true)

The change is detected and we are both notified and given the option to rollback. This time, we first want to inspect the changes, so let&#39;s answer &quot;n&quot; for No:

![alt text](https://github.com/IPvZero/Nornir-Blog/blob/master/images/16.png?raw=true)

The script terminates and leaves the relevant artefacts which we are free to inspect (notice the new directories ```ospf-current``` and ```ospfdiff```):

![alt text](https://github.com/IPvZero/Nornir-Blog/blob/master/images/17.png?raw=true)

We can now freely examine these changes and decide if we want to erase them by performing a rollback, or leave them and updating our OSPF definitions:

![alt text](https://github.com/IPvZero/Nornir-Blog/blob/master/images/18.png?raw=true)

Upon examination it is clear now that these configuration are certainly not meant to be present in the network. We then rerun Pynir, this time choosing &quot;y&quot; to rollback to our desired state:

![alt text](https://github.com/IPvZero/Nornir-Blog/blob/master/images/19.png?raw=true)

This selection triggers Nornir to execute our custom functions that remove all current OSPF configs and artefacts before redeploying OSPF as specified in our ```host_vars``` definition files.

First the OSPF configurations are identified by the show output, and then negated:

![alt text](https://github.com/IPvZero/Nornir-Blog/blob/master/images/20.png?raw=true)

Pynir then pulls out desired state from our host varables, builds our configuration using the Jinja2 template, and pushes out the config: ![alt text](https://github.com/IPvZero/Nornir-Blog/blob/master/images/21.png?raw=true)

For our final validation, let&#39;s rerun the script and ensure that we are now in compliance with our desired state:

![alt text](https://github.com/IPvZero/Nornir-Blog/blob/master/images/22.png?raw=true)

Excellent! Everything is back to the way it should be.

As you can see, combining Nornir with pyATS can allow us to easily monitor and rollback our network to ensure we are compliant with our desired state of the network. As demonstrated, the largest challenge was finding a workaround to remove all undesired OSPF configurations. With modern devices with APIs, with options for candidate configurations, etc, this problem vanishes. Unfortunately, however, we still need to deal with older devices that were not built with automation in mind, and we have to create inventive and sometimes inefficient workarounds to solve a particular problem. This script is an attempt at doing that.

You can download the script and all subsequent configurations at:
[https://github.com/IPvZero/pynir](https://github.com/IPvZero/pynir)


### Contact

[Twitter](https://twitter.com/IPvZero)

[Youtube](https://youtube.com/c/IPvZero)

