# Containerlab configuration engine

This little demo is designed to show how the embedded configuration engine works in containerlab 0.18.0.

The goal of the config engine is to enable containerlab users not only to deploy the labs, but also to provision the nodes with the actual configuration so that the use case can be executed immediately without using 3rd party config management tools.

## 1 Demo highlights
The demo will demonstrate how containerlab can deploy a pair of SR Linux nodes and provision an eBGP peering between them.

Throughout the course of the demo we will get to know what `containerlab config` command offers, how to define the templates and variables which are used to create the actual CLI configuration snippets, and how to test and execute them.

## 2 Config engine overview
Containerlab' embedded config engine is based on a well known templating functionality. It is expected that the users will create a template(s) with the actual config snippets that config engine will augment with the user-defined variables and push to the nodes of the lab.

At the very high level, the following two pieces make the core of the config engine:

* **templates** - text files written in [Go template syntax](https://learn.hashicorp.com/tutorials/nomad/go-template-syntax) which contain CLI snippets
* **variables** - user-defined variables that are provided in the topology file to parametrize the snippets used in templates.

## 3 Variables
With variables we provide the user-defined values to a template. Variables can be set in the `config.vars` block on the three levels of the topology:

* defaults
* kinds
* node

### 3.1 Node-scoped variables
Most commonly the variables are defined on a per-node basis, consider the following snippet where a single variable named `my_var` is set for two nodes of a lab with different values:

```yaml
nodes:
  node1:
    config:
      vars:
        my_var: somevalue1

  node2:
    config:
      vars:
        my_var: somevalue2
```

> Note, that variables can't have dashes `-` in their name, instead use underscores `_`

### 3.2 Link-scoped variables
In addition to the node-scoped variables (these are the variables that are defined for a node/kind) containerlab has another place where you can define the variables - in the `links` section.

This is done to leverage the topology information that we, as users, provide in the links section. We will have an example later demonstrating how we can use it. For now, just look at how we define the link-scoped variables:

```yaml
links:
  - endpoints: ["srl1:e1-1", "srl2:e1-1"]
    vars:
      port: [ethernet-1/1, ethernet-1/1]
      clab_link_ip: 192.168.0.1/30
```

## 4 Templates and Roles
The template is basically a text file with your usual CLI commands where you inject variables to render the template as a list of commands that you want to execute on a certain node.

For example, here is a valid template snippet that will configure the location of the system, based on the value of the variable named `location`.

```tmpl
/ system information location {{ .location }}
```

when such template is used in conjunction with the following topology file, this will make this template to render to a different command accordingly.

```yaml
nodes:
  node1:
    config:
      vars:
        location: Brussels

  node2:
    config:
      vars:
        location: Amsterdam
```

## 4.1 Roles
Since your lab can have multiple kinds (for example SR Linux and SR OS), you need to somehow make a distinction which template is to use with which node.

This is when the Roles come into the picture. By default, the role is your kind. So SR Linux nodes will have a role `srl` and SR OS nodes will have a role `vr-sros`.

The role must be used in the template naming convention when you create the templates. If you write a template for SR Linux nodes, you need to name it as `<template-name>__<role>.tmpl`. As in this repo, the single template we have is named `bgp__srl.tmpl`.

If we would have SR OS nodes as part of this topology, we would create a template `bgp__vr-sros.tmpl` accordingly. The SR OS template would then have had SR OS commands to configure the location.

## 5 Magic variables
Containerlab does a lot in the backgrounds to simplify the template creation. One of the many things that containerlab does is that it creates magic variables that we can use in our templates.

Remember that link-scoped vars from 3.2? Let's dig a bit deeper and look at them once again:

```yaml
links:
  - endpoints: ["srl1:e1-1", "srl2:e1-1"]
    vars:
      port: [ethernet-1/1, ethernet-1/1]
      clab_link_ip: 192.168.0.1/30
```

Here we defined two variables under the single link our nodes have between each other.

The `port` variable is a list of two strings, where each element in this list corresponds to each endpoint of the link. Namely here we say that I want `srl1` node to have variable named `port` to have `ethernet-1/1` value for its first interface.

Subsequently we used the same port name for the `srl2` node.

Next we used a magic variable called `clab_link_ip` that when set will be used by containerlab to automatically calculate the IP addresses for both ends of the link using the provided subnet.

> Magic variables are variables which have special meaning or may lead to some additional computation done by containerlab. They are prefixed with `clab_`.

When we set the value of this var to `192.168.0.1/30` we make containerlab to create calculate the remote IP to be `192.168.0.2` and it will make a certain variable under the hood which we will use in our template.

At this moment you might think that there is a lot of stuff happening in the background which is hard to grasp, but don't you worry, it will grow on your when you try it.

First, you can easily make containerlab to show you all the variables that it knows about. Given the files that we have in this repo

* `srlbgp.clab.yml` - topology file which has vars defined inside of it
* `bgp__srl.tmpl` - template

we can use `clab config` command with `--vars` flag to dump all the variables:

```
clab config -t srlbgp.clab.yml template --vars
```

This will output all the variables per each node of the topology:

```
INFO[0000] Parsing & checking topology file: srlbgp.clab.yml 
INFO[0000] srl1 vars = bgp:
  as: 65100
  groups:
    ebgp:
      peer_as: 65200
  neighbors:
    192.168.0.2:
      peer_group: ebgp
clab_links:
- clab_far:
    clab_link_ip: 192.168.0.2/30
    clab_link_name: to_srl1
    clab_node: srl2
    port: ethernet-1/1
  clab_link_ip: 192.168.0.1/30
  clab_link_name: to_srl2
  port: ethernet-1/1
clab_role: srl
system_ip: 10.0.0.1/32 

INFO[0000] srl2 vars = bgp:
  as: 65200
  groups:
    ebgp:
      cluster_id: 1.2.3.4
      peer_as: 65100
  neighbors:
    192.168.0.1:
      peer_group: ebgp
clab_links:
- clab_far:
    clab_link_ip: 192.168.0.1/30
    clab_link_name: to_srl2
    clab_node: srl1
    port: ethernet-1/1
  clab_link_ip: 192.168.0.2/30
  clab_link_name: to_srl1
  port: ethernet-1/1
system_ip: 10.0.0.2/32 
```

as you see, there are quite some variables that are exposed per each node, and you can access them all in your templates.

## 6 Writing the templates
It is quite easy to define variables, it is just YAML. Writing the templates is something that takes a few tries. The [Go template syntax](https://pkg.go.dev/text/template) is not that complex and has similarities to Jinja2, but still going through some basic examples may help get you started.

So let's see the very basics of Template language that we use in config engine using the template provided within this repo.

### 6.1 Accessing the variables
Now that you know how to display all the variables that you have for each node, let's see how we can use them in the templates.

Given the following `srl1` vars we will see how to access different variables in the template:

```yaml
config:
  vars:
    system_ip: 10.0.0.1/32
    bgp:
      as: 65100
      groups:
        ebgp:
          peer_as: 65200
      neighbors:
        192.168.0.2:
          peer_group: ebgp
```

The top level variables with scalar values, such as `system_ip: 10.0.0.1/32` can be accessed in the template with the `.<variable-name>` syntax:

```tmpl
/ interface system0 admin-state enable subinterface 0 ipv4 address {{ .system_ip }}
```

In the same spirit, you can access variables nested under the yaml objects. This is how you get access to the BGP AS:

```tmpl
/ network-instance default protocols bgp autonomous-system {{ .bgp.as }}
```

If you need to range over the list or map/object, you need to use the `range` function. For example, if you happen to have multiple BGP neighbors defined and you want to create all of them in the template, you would need to use the following construct in your template:

```tmpl
{{ range $ip, $nei := .bgp.neighbors }}
/ network-instance default protocols bgp neighbor {{ $ip }} peer-group {{ $nei.peer_group }}
{{ end }}
```

Note, how we assign two new variables (`$ip` and `$nei`) when ranging over the map accessed by `.bgp.neighbors`. The `$ip` variable will hold the key of the map of neighbors (`192.168.0.2` in our example), and `$nei` will store the content of the map addressed by this key.

Now in our variables section we have only one neighbor defined, but the range construct is ready to accommodate for multiple neighbors definition.

### 6.2 If conditions
To make a certain portion of the template to execute if a certain condition is true/false you use `if` condition expression.

In our example we configure a BGP route reflection only if the `cluster_id` variable was defined by a user in the vars section of the topology. This is how it is done in the template:

```tmpl
{{/* visit all the groups defined per node and capture the group name and its contents */}}
{{ range $name, $group := .bgp.groups }}
  {{/* check if cluster_id was defined in a group */}}
      {{ if $group.cluster_id }}
          / network-instance default protocols bgp group {{ $name }} route-reflector client false
          / network-instance default protocols bgp group {{ $name }} route-reflector cluster-id {{ $group.cluster_id }}
      {{ end }}
  {{ end }}
{{ end }}
```

### 6.3 Static configuration
As expected, when you just want a template to have a static configuration block/line, this is a block/line without any variables, you can just have it directly in the template body.

For example, to create a policy which has no parameters, you can simply paste the config lines in the template:

```tmpl
{{/* configuring the policy first */}}
/ routing-policy
/ routing-policy policy all
/ routing-policy policy all default-action
/ routing-policy policy all default-action accept
```

## 7 Rendering the templates
It makes a lot of sense to check if the template that you created renders correctly given the variables you have defined.

In containerlab' config engine you can display the rendered template without actually pushing the config lines towards the nodes. This is also known as a dry-run.

Given the topology file and the template you see in this repo, use the following command to render the template for both nodes:

```
containerlab config -t srlbgp.clab.yml template -p . -l bgp
```

* The `-p` flag sets the directory which containerlab will look at for template files
* With `-l` flag is used to provide the template names to use. We have a single template named `bgp` in the current directory.

The output will display the rendered templates for all the nodes in a lab. Here is a trimmed example:

```
INFO[0000] Parsing & checking topology file: srlbgp.clab.yml 
INFO[0000] srl1
  Template bgp__srl.tmpl for srl1 = [[
     / interface system0 admin-state enable subinterface 0 ipv4 address 10.0.0.1/32
     
     / network-instance default {
         interface system0.0 {
         }
     }
  ]] 
INFO[0000] srl2
  Template bgp__srl.tmpl for srl2 = [[
     / interface system0 admin-state enable subinterface 0 ipv4 address 10.0.0.2/32
     
     / network-instance default {
         interface system0.0 {
         }
     }
  ]] 
```

Rendering your template is a good way to check that your resulting config looks correct.

## 8 Sending the configs
Now if the rendered template meets your expectation you can safely push the configuration commands to the nodes, using the `containerlab config` command.

Here is how you configure the nodes of your lab using the vars and template in this repo:

```
containerlab config -t srlbgp.clab.yml -p . -l bgp
```

The `config` subcommand takes the same flags `-p` and `-l` to indicate the path to the templates directory, as well as the template name.

Containerlab then will render the template and will send those commands over to the srl nodes. It will automatically enter the candidate mode and will automatically commit the changes, so you don't need to provide these commands yourself in the template.

The successful config execution will produce the following result:

```
INFO[0000] Parsing & checking topology file: srlbgp.clab.yml 
WARN[0000] Skipping host key verification for clab-srlbgp-srl1:22 
WARN[0000] Skipping host key verification for clab-srlbgp-srl2:22 
INFO[0000] Connected to clab-srlbgp-srl1:22             
INFO[0000] Connected to clab-srlbgp-srl2:22             
INFO[0002] bgp__srl.tmpl COMMIT - 25 lines              
INFO[0002] bgp__srl.tmpl COMMIT - 27 lines
```

You can then check that the BGP has been established on both neighbors:

```
clab exec -t srlbgp.clab.yml --cmd "sr_cli show network-instance default protocols bgp neighbor *"
```

This command will display the output of the command executed on both nodes of the lab:

```
INFO[0000] Parsing & checking topology file: srlbgp.clab.yml 
INFO[0002] Executed command 'sr_cli show network-instance default protocols bgp neighbor *' on clab-srlbgp-srl1. stdout:
---------------------------------------------------------------------------------------------------------------------------------------
BGP neighbor summary for network-instance "default"
Flags: S static, D dynamic, L discovered by LLDP, B BFD enabled, - disabled, * slow
---------------------------------------------------------------------------------------------------------------------------------------
---------------------------------------------------------------------------------------------------------------------------------------
+----------------------+--------------------------------+----------------------+--------+------------+------------------+------------------+----------------+--------------------------------+
|       Net-Inst       |              Peer              |        Group         | Flags  |  Peer-AS   |      State       |      Uptime      |    AFI/SAFI    |         [Rx/Active/Tx]         |
+======================+================================+======================+========+============+==================+==================+================+================================+
| default              | 192.168.0.2                    | ebgp                 | S      | 65200      | established      | 0d:0h:3m:5s      | ipv4-unicast   | [2/1/2]                        |
+----------------------+--------------------------------+----------------------+--------+------------+------------------+------------------+----------------+--------------------------------+
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Summary:
1 configured neighbors, 1 configured sessions are established,0 disabled peers
0 dynamic peers 
INFO[0003] Executed command 'sr_cli show network-instance default protocols bgp neighbor *' on clab-srlbgp-srl2. stdout:
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
BGP neighbor summary for network-instance "default"
Flags: S static, D dynamic, L discovered by LLDP, B BFD enabled, - disabled, * slow
---------------------------------------------------------------------------------------------------------------------------------------
---------------------------------------------------------------------------------------------------------------------------------------
+----------------------+--------------------------------+----------------------+--------+------------+------------------+------------------+----------------+--------------------------------+
|       Net-Inst       |              Peer              |        Group         | Flags  |  Peer-AS   |      State       |      Uptime      |    AFI/SAFI    |         [Rx/Active/Tx]         |
+======================+================================+======================+========+============+==================+==================+================+================================+
| default              | 192.168.0.1                    | ebgp                 | S      | 65100      | established      | 0d:0h:3m:7s      | ipv4-unicast   | [2/1/2]                        |
+----------------------+--------------------------------+----------------------+--------+------------+------------------+------------------+----------------+--------------------------------+
---------------------------------------------------------------------------------------------------------------------------------------
Summary:
1 configured neighbors, 1 configured sessions are established,0 disabled peers
0 dynamic peers 
```
