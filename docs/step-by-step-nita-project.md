
- [What you need to know to create your own projects in NITA](#what-you-need-to-know-to-create-your-own-projects-in-nita)
- [Files and Folders](#files-and-folders)
    - [Examples, Examples, Examples](#examples-examples-examples)
- [Device Requirements](#device-requirements)
    - [Inventory File](#inventory-file)
    - [Inventory File Location](#inventory-file-location)
- [Configuration Requirements](#configuration-requirements)
    - [Static Route Configuration Example](#static-route-configuration-example)
- [Automation Steps with NITA](#automation-steps-with-nita)
    - [Implementing the Roles Structure](#implementing-the-roles-structure)
    - [Structure and Example of a tasks yaml file:](#structure-and-example-of-a-tasks-yaml-file)
- [Jinja2 Templates](#jinja2-templates)
    - [Jinja2 Example](#jinja2-example)
    - [Jinja2 file locations](#jinja2-file-locations)
- [Workbook Content:](#workbook-content)
  - [Define Attribute/Value Pairs:](#define-attributevalue-pairs)
  - [Handling Multiple Values](#handling-multiple-values)
- [Using and Modifying the Example Workbook](#using-and-modifying-the-example-workbook)
  - [Base Data Worksheet](#base-data-worksheet)
  - [Management Interface Data Worksheet](#management-interface-data-worksheet)
  - [Integration with NITA:](#integration-with-nita)
- [NITA Files and Their Purpose](#nita-files-and-their-purpose)
    - [Editing Files](#editing-files)
    - [Customizing `project.yaml`](#customizing-projectyaml)
      - [The `shell_command:` Statement](#the-shell_command-statement)
      - [Step-by-Step Instructions for Customizing `project.yaml`](#step-by-step-instructions-for-customizing-projectyaml)
      - [Complete Example of the Updated `project.yaml`](#complete-example-of-the-updated-projectyaml)
    - [Playbook Updates - `sites.yaml`](#playbook-updates---sitesyaml)
      - [Steps to Update `sites.yaml`](#steps-to-update-sitesyaml)
      - [Example of Updated `sites.yaml`](#example-of-updated-sitesyaml)
    - [Manually Testing Static Route Implementation](#manually-testing-static-route-implementation)
      - [Steps for Manual Testing](#steps-for-manual-testing)
- [Building a Project Zip File](#building-a-project-zip-file)
    - [Project Specific Files](#project-specific-files)
    - [Creating the Zip File](#creating-the-zip-file)
- [Reference Example](#reference-example)
  - [Automating Tests](#automating-tests)
- [TOOLS](#tools)
- [VERSION](#version)
- [SEE ALSO](#see-also)

# What you need to know to create your own projects in NITA
NITA ships with two example projects that we use to demonstrate its ability to automate the build and test processes for both a simple [DC WAN topology based on IPCLOS and eBGP](https://github.com/Juniper/nita/tree/main/examples/ebgp_wan) and an [EVPN VXLAN data centre using Juniper QFX devices](https://github.com/Juniper/nita/tree/main/examples/evpn_vxlan_erb_dc). However, what if we wanted to configure NITA to do build a new environment? This document explains how to start building a new project.

>! All the instructions below are based on using Ubuntu 20.04 as the local host for the NITA tool.

---- 

# Files and Folders

Below is a screenshot of the files and folders within one of the NITA project examples that has already been created. Here we are looking at the `ebgp_wan` project, along with a brief explanation of each file and folder. Familiarising yourself with the contents is essential, as we refer back to the contents throughout this guide.

![Files and Folders Image](docs/images/files_folders.PNG)

### Examples, Examples, Examples

To initiate the new project, let's start by using the existing [EBGP_WAN](https://github.com/Juniper/nita/tree/main/examples/ebgp_wan) example from the NITA repository, begin by cloning the NITA repository to the local machine you are using for the NITA server. 

To clone the repository, open a terminal and run the following command:

```bash
git clone https://github.com/Juniper/nita.git
```

Once the repository is cloned, navigate into the `examples` directory and copy the `ebgp_wan` folder to your local NITA Project directory and rename it to the `new_project` name as desired. You can do this using the following command:
>!Replace `new_project' name with the desired name for the new project.


```bash
cd nita/examples/

cp -r ebgp_wan /path/to/new_project
```

With the `new_project folder` created and renamed, we are now positioned to start the process. The first item on the list is to modify the device inventory so that only the relevant hosts/host groups for the `new_project` are updated moving forward.

---- 

# Device Requirements

The host files and inventory data contained within the example folder structures are built around standard Ansible roles. Each group is referenced as NITA understands it, which will help significantly when instructing NITA which devices to work on later.

To generate the host inventory, we suggest to use the Ansible [INI format](https://docs.ansible.com/ansible/latest/user_guide/intro_inventory.html). A straightforward approach to this is to rename the existing `hosts` file from within the previously copied example folder. If you plan to create more than one set of hosts files for different areas of your project, you can follow the same process for those as well and refer to the respective host files later when setting up NITA via the web application.

Below is a populated example of an inventory file in [INI format](https://docs.ansible.com/ansible/latest/user_guide/intro_inventory.html):

### Inventory File
```ini
[all:children]
switches
routers
servers

[switches:children]
spines
leaves

[spines]
edc-spine1
edc-spine2

[leaves]
edc-leaf1
edc-leaf2

[routers]
router1
router2

[servers]
edc-webserver1
edc-webserver2
edc-appserver
edc-dbserver
```

When naming your host groups, consider using a consistent naming scheme that reflects the purpose or type of each device. This ensures easy identification of devices and reduces confusion during configuration and management of the devices.

### Inventory File Location

Once you have modified the inventory file with the new groups and host names for the `new_project` project, proceed to save this file in the root of the `examples/{new_project}` folder. We suggest giving it a unique name so you can easily recognize which host file corresponds to which environment later. A common practice is to append the environment name to the file name, such as `hosts_dc1` or `hosts_dc2`, to facilitate better management of your inventory files.

By keeping your device inventory organized and appropriately named, will keep the process of configuring and managing your network devices within the NITA framework much easier.

---- 

# Configuration Requirements

Next, consider the configurations needed for each device type in the domains/zones and work towards standardizing them by using templates. Templates make building and troubleshooting easier because everything remains standardized and consistent — right?!

### Static Route Configuration Example

In the `new_project` we are working on we intend to concentrate on configuring a static route such that **ALL** devices in `dc1` will utilise the gateway `123.123.123.123`. In conventional network engineering practice, you would have to log into each device manually, edit the configuration via the Junos CLI, enter the respective `set` statements, and commit the configuration. Afterwards you would then verify the newly entered configuration by entering `show routing-options` on the CLI to ensure everything is as expected. 

Below are examples of how that might look:

On `dc1` Devices:
```plaintext
dc1-spine1# set routing-options static route 0.0.0.0/0 next-hop 123.123.123.123

dc1-spine1> show routing-options
static {
    route 0.0.0.0/0 next-hop 123.123.123.123;
}
```

----

# Automation Steps with NITA

To streamline the automation process using NITA, we need to translate the above manual configurations into reusable templates.

**Jinja2 Snippets**

First, we need to create our Jinja2 snippets, these snippets enables us to modify configurations according to device types or environments as required, we do this using [Jinja2 templates](https://jinja.palletsprojects.com) after these snippets have been generated we can focus on the relevant variables we need to populate.

**Excel Workbook Variables**

Next, we use an Excel workbook to store the inventory/variable data for the templates, the workbook contains the inventory data and serve's to populate the variables required from the the Jinja2 snippets we generated above and then NITA uses this file along with the templates to generate the device configurations.

----

### Implementing the Roles Structure

To be able to use the aforementioned Jinja2 snippets and the workbook containing the data variables we need to have the correct directory structure in place so that NITA is able to locate the files and build the configuration required for each role, to do this we have to build the directory structure, proceed to navigate to the existing `new_project` directory, then within this directory, create a new subdirectory called `roles`. 

The structure should look something similar to the below screenshot (_paying attention to the lower half of the image shown_).  Now within the newly created `roles` directory, create subdirectories for each different role as required, each role should typically have its own folders (_shown here as "mx_common"_) that include:

   - `tasks/`: Contains the main task files.
   - `templates/`: This is where the new Jinja2 template should be stored. 

 ![j2_location image](docs/images/j2_location.PNG)

Once we have the `roles` directory structure in place, NITA will first check this designated `roles` location for any roles when executing a playbook. This initial search helps to ensure that the application is aware of all available templates and configurations defined.

### Structure and Example of a tasks yaml file:

The `tasks/main.yaml` file serves a crucial purpose. It is the entry point for defining the tasks that will be executed as part of that role. The main.yaml file contains a list of tasks that are executed in the order they are defined.

As we continue working through our example we can see the `yaml` file contains a list of tasks defined using the `- name:` and `- task:` patterns (_this yaml must contain at least 1 pattern_), so if you need to add any further `tasks` to this role it can be completed by literally adding a new `- name:` and `- task:` pattern to the yaml:

```
---
- name::      Specifies the name of the task. It's a description that explains what the task is doing.
  template::  This is an Ansible module used to process Jinja2 templates. The module takes a source file and generates a destination file.
              The `src` parameter specifies the template file.
              The `dest` parameter` specifies the path where the rendered configuration file will be saved.
              The destination is defined using a variable {{ tmp_dir }}. The variable tmp_dir should be defined elsewhere.
```

```yaml
---
- name: Creating BGP config
  template: src=bgp.j2 dest={{ tmp_dir }}/bgp.cfg

- name: next element name
  template: next template src & dest objects
```
This template can also be written as follows if preferred:
   ```yaml
   - name: Creating BGP config
     template:
       src: bgp.j2
       dest: {{ tmp_dir }}/bgp.cfg
   ```

The `template` module in NITA will render the Jinja2 template based on the variables available and then place the rendered configuration in the specified destination folder on the local machine.
After building the relevant file structure and management files outlined above we can actually now start working on the templates and then later look into populating the variables outlined in the Jinja2 templates with the aforementioned "workbook".

# Jinja2 Templates
For this example, since we are concentrating on how to add a static route using templates, we will proceed with this scenario using the following Jinja2 example template structure which contains a static route -- as you can see in the example the section begins at `{% for route in routing_options %}` and from here we can see how the Jinja2 template builds a static route by "populating" the relevant variables requested -- `{{ route.static }}` and  `{{ route.destination }}` for example:

### Jinja2 Example
```jinja
#jinja2:lstrip_blocks: True
## ****************************************************************************************************** ##
##                                                                                                        ##
## Project: nita-ansible                                                                                  ##
##                                                                                                        ##
## Copyright (c) Juniper Networks, Inc., 2021. All rights reserved.                                       ##
##                                                                                                        ##
## Notice and Disclaimer: This code is licensed to you under the Apache 2.0 License (the "License").      ##
## You may not use this code except in compliance with the License. This code is not an official          ##
## Juniper product. You can obtain a copy of the License at                                               ##
## https://www.apache.org/licenses/LICENSE-2.0.html                                                       ##
##                                                                                                        ##
## SPDX-License-Identifier: Apache-2.0                                                                    ##
##                                                                                                        ##
## Third-Party Code: This code may depend on other components under separate copyright notice and license ##
## terms. Your use of the source code for those components is subject to the terms and conditions of the  ##
## respective license as noted in the Third-Party source code file.                                       ##
##                                                                                                        ##
## ****************************************************************************************************** ##
routing-options {
{% if loopback_ip is defined %}
    router-id {{ loopback_ip }};
{% endif %}
{% if routing_options is defined %}
{% for route in routing_options %}
    {% if route.destination == "discard" %}
    static {
        route {{ route.static }} {{ route.destination }};
    }
    {% else %}
    static {
        route {{ route.static }} next-hop {{ route.destination }};
    }
    {% endif %}
{% endfor %}
{% endif %}
}
endif %}
```

### Jinja2 file locations

After generating the new Jinja2 templates, it is crucial to store these templates in locations that NITA can access. The organisation of these templates and roles in NITA play a significant part in the automation tasks and how each device role is configured.
To utilise the Jinja2 templates created for the `new_project`, we use the standard directory structure used by Ansible, specifically the `roles/templates` directory.  

Now we have completed the Jinja2 (J2) document and looked at the structure all we have left is to actually generate the variable data using the Excel workbook and corresponding spreadsheets within.

---- 

# Workbook Content:

If you've worked with Ansible before, you're probably familiar with its use of YAML files for the inventory data, which populates the variables required in Jinja2 templates. However, when using the NITA application, this process is streamlined through using a workbook. This workbook contains multiple spreadsheets, each dedicated to specific configuration types. For instance, you might create a spreadsheet for management configurations, another for IP/VRF settings, one for BGP configurations, and additional sheets for SYSLOG and VRFs, etc etc... 

NITA takes advantage of these spreadsheets and then automatically generates the necessary YAML files in the background. This means you don't have to create all the YAML files manually for each device, allowing you to harness the power of automation without the tedious workload - _the magic of automation eh!_.  

The [Jinja2 Template](#jinja2-example) snippet shown above contains a basic configuration stanza for routing options. To complete the configuration just for a static route, we need to populate specific variables beginning at `{% for route in routing_options %}`, these varaibles are namely `{{ route.static }}` and `{{ route.destination }}` - among others, if we were using Ansible directly you would generate these variables in a yaml file with content similar to the following:
```yaml
---
# ****************************************** #
# ** START - ROUTING OPTIONS              ** #
# ****************************************** #
routing_options:
  - static: "0.0.0.0/0"
    destination: "192.168.1.1"
  - static: "192.168.2.0/24"
    destination: "discard"
```

## Define Attribute/Value Pairs: 

However, to complete this process for NITA, we use workbooks and within that workbook we have a “`routing-options`” spreadsheet. This spreadsheet must include detailed entries for the attribute/value pairs referenced in the Jinja2 template.

Here’s how to approach this, open the workbook and either add or edit a sheet called "`routing_options+`" and within this sheet, you will need to ensure the columns created represent the required Attribute/Value Pairs relevant to the `routing_options` defined within the example [Jinja2 Template](#jinja2-example) above - in this case it would have Attribute/Value Pairs such as:
   - `route.static`: The static route to be added.
   - `route.destination`: The destination network for the static route.
   - Additional necessary attributes based on configuration requirements.

   For instance, the contents of your "`routing_options+`" spreadsheet might look something like this:

  ![static spreadsheet image](docs/images/static_route.PNG)

## Handling Multiple Values

It is important to note that some column headers within the copied worksheet may contain the "`+`" symbol, such as `neighbors+name` or `neighbors+asn`. This notation is used to indicate that there are multiple values associated with the same attribute. By using the "`+`" nomenclature, NITA is instructed to treat these fields as ordered dictionaries rather than simple strings.

This distinction is particularly useful when you need to define multiple neighbors or attributes that share a common category while maintaining their respective state. If you plan to parse similar data types in your configurations later, understanding this structure is essential for proper functionality and reliability.

![Note + Image 1](docs/images/note_+.PNG)
![Note + Image 2](docs/images/note_+2.PNG)

----

# Using and Modifying the Example Workbook
We suggest to begin modifying the example workbook, start by adding `base` data information for the devices. This `base` data functions similarly to global variables, providing a standard configuration across various Jinja2 templates.

## Base Data Worksheet

The `base` worksheet must contain the following three columns: `host`, `name`, and `value`. The layout should resemble the example shown below:

![Base Data Spreadsheet Image](docs/images/base.PNG)

**Column Definitions**

- **Host**: 
  - **Description**: This column indicates the name of the YAML file where the information will be stored.
  - **Purpose**: By specifying the host name, you ensure that the correct configuration data is written to the appropriate YAML file.
  - **Example**: `router1.yml`, `switchA.yml`

- **Name**: 
  - **Description**: The "`name`" column defines the attribute for which the `value` will be set. This acts as the key in the key-value pair, identifying the configuration setting that will be populated in the Jinja2 templates.
  - **Purpose**: The name should be descriptive enough to validate the parameter it represents, making it easier to understand and manage configurations.
  - **Example**: `netconf_user`, `netconf_passwd`, `management_ip`

- **Value**: 
  - **Description**: The "`value`" column holds the data for the attribute defined in the "name" column. This is the actual value that will be used by NITA when generating configuration files.
  - **Purpose**: The values entered in this column have to match with the data types used by the Jinja2 templates.
  - **Example**: `admin`, `securePassword123`, `192.168.1.1`

**Critical Attribute/Value Pairs within the `BASE` worksheet**

Among the various attribute/value pairs you can define, the following are particularly important:

- **`netconf_user`**: This value specifies the username that NITA will utilize to authenticate with network devices.

- **`netconf_passwd`**: This represents the password associated with the user defined in `netconf_user`. It is critical for secure communication.

- **`netconf_port`**: This defines the port through which the NETCONF protocol will connect to the network devices. Commonly, this is port `830`.

It is essential to ensure that these parameters are accurate and match the network environment, as they provide the ability for NITA to communicate with the devices.

![Netconf User Spreadsheet Image](docs/images/netconf_user.PNG)


## Management Interface Data Worksheet

The next sheet in the workbook is dedicated to configuring the management interfaces of devices. This worksheet comprises four columns, where the first column represents the YAML file (aka: `host` column) that will be generated. The next columns define the attribute/value pairs necessary for configuring interfaces, specifying IP addresses, and setting subnet masks for each device you intend to configure.

The layout of the "`Management Interface`" data worksheet should appear similar to the example shown below:

![Management Interface Spreadsheet Image](docs/images/management_interface.PNG)

**Column Definitions**

- **Host**: 
  - **Description**: The first column, designated as "`host`", specify's the name of the YAML file that will be generated. This file will contain configurations related to the management interface.
  - **Purpose**: Properly naming this file allows for easy identification and management of configurations associated with individual devices.
  - **Example**: `router1_management.yml`, `switch2_management.yml`

- **Interface (int)**: 
  - **Description**: The "`int`" column is used to specify the management interface on each device. This could be an interface name like `GigabitEthernet0/0`, `em0` etc.
  - **Purpose**: Correctly identifying the management interface is critical for the deployment of configuration changes.
  - **Example**: `GigabitEthernet0/0`, `fxp0`

- **IP Address (ip)**: 
  - **Description**: The "`ip`" column is where you provide the IP address assigned to the management interface. This is the address that will be utilized for device management going forward.
  - **Purpose**: The IP address must be unique within the network and appropriately configured.
  - **Example**: `192.168.1.10`, `10.0.0.1`

- **Subnet Mask (mask)**: 
  - **Description**: The "`mask`" column defines the subnet mask corresponding to the IP address specified. The subnet mask determines the network portion of the IP address and defines the range of addresses within the subnet.
  - **Purpose**: Proper configuration of the subnet mask is crucial for establishing correct routing and communication paths within the network. A misconfigured subnet mask can prevent the device from communicating with other devices on the same network.
  - **Example**: `255.255.255.0`, `255.255.255.128`


## Integration with NITA: 
Once the spreadsheet is populated, and uploaded to NITA, NITA will utilize the data you’ve entered in the workbook to generate the relevant YAML files. This eliminates the need for manual file creation and ensures consistency for device configurations. It is crucial to remember where you store the file so you can locate and upload this to NITA later!.  

---- 

# NITA Files and Their Purpose

In every NITA project, there are several standard files that we recommend including to ensure a smooth and efficient setup. Instead of reinventing the wheel, we suggest copying these files from the existing `examples` available for [eBGP WAN](https://github.com/Juniper/nita/tree/main/examples/ebgp_wan) and [EVPN VXLAN](https://github.com/Juniper/nita/tree/main/examples/evpn_vxlan_erb_dc). These examples offer a well-structured starting point for your projects.

Below is a breakdown of the files that you should be aware of and work with:

| File | Purpose |
|---|---|
| `project.yaml` | This file serves as a cornerstone of the NITA framework, utilized as part of Jenkins jobs to orchestrate the network configuration process. It acts as the main playbook for building or testing the network infrastructure and includes configuration parameters for the entire project. |
| `build/sites.yaml` | This file is crucial for directing the required `roles` in your project. It maps out which roles should be applied to specific hosts and which can be customized according to the needs of the project, allowing for flexibility of configurations. |
| `ansible.cfg` | This configuration file contains default values for Ansible parameters, defining settings that affect the behavior of how Ansible runs. Although this file typically remains unchanged, it can be modified if there are project-specific requirements that necessitate adjustments to Ansible's parameters. ie to modify SSH-related parameters, like control path or timeout.|
| `build.sh` | This shell script executes the playbooks specified in the `build/sites.yaml` file against the specified groups, devices, or roles listed in your [hosts inventory](#host-inventory). It acts as the execution engine for the deployment process. |
| `make_clean.yaml` | This playbook is designed to create or reset the build directories for each host. As implied by its name, it cleans up the file structure to allow for a fresh build process, ensuring that residual files do not interfere with new deployments. |
| `make_etc_hosts.yaml` | This playbook runs the `make_hosts_entry.sh` script, which reads from the [hosts inventory](#host-inventory) and uses the data in the `management_interface` worksheet of the Excel workbook to construct and populate the `/etc/hosts` file. This ensures that each device can resolve the necessary hostnames correctly within the network. |
| `make_hosts_entry.sh` | This script updates the host table in `/etc/hosts` for the devices specified in your [hosts inventory](#host-inventory). By modifying the hosts file, it facilitates easier communication between devices in your network by ensuring that hostnames are resolved to IP addresses appropriately. |

### Editing Files

Generally, there is little need to modify most of the files mentioned above, with the exception of `project.yaml` and `build/sites.yaml`. 

- **`project.yaml`**: This file can be customized for specific job categories within NITA. It's where you define parameters such as the Docker or Kubernetes image to be loaded when the Jenkins job is executed, allowing you to tailor the project environment to your needs.

- **`build/sites.yaml`**: This file should be updated to reflect the new project requirements and the various roles you have generated. Configuring this file correctly ensures that the deployment process aligns with the Ansible roles associated with your devices.

By utilizing these standard files and understanding their purposes, you can effectively manage your NITA projects and streamline the process of network automation.

----

### Customizing `project.yaml`
This file will include essential fields like `name:` and `description:`, along with several `action` statements. Each action defines jobs and should include these basic statements:

| Statement         | Description                                                                 |
|-------------------|-----------------------------------------------------------------------------|
| `name:`           | A descriptive name for this action (this will appear in the "Trigger" menu on the NITA Webapp UI). |
| `jenkins_url:`    | A unique name for the directory where Jenkins will store its project files. |
| `category:`       | Can be either `NOOB`, `BUILD`, or `TEST`, which is needed by the Webapp UI.|
| `configuration:`  | Include at least one `shell_command:` statement for the job to be run.     |

#### The `shell_command:` Statement

If the magic happens in `project.yaml`, then it's the `shell_command:` statement that acts as the spell. Here’s what each category does:

| Category | The `shell_command` Will                                                                                   |
|----------|------------------------------------------------------------------------------------------------------------|
| NOOB     | Writes the YAML files and runs a bash script (e.g., `noob.sh`) in an Ansible Docker container that executes an Ansible playbook. |
| BUILD    | Writes the YAML files and runs a bash script (e.g., [`build.sh`](https://github.com/Juniper/nita/blob/main/examples/evpn_vxlan_erb_dc/build.sh)) in an Ansible Docker container to execute an Ansible playbook. |
| TEST     | Writes the YAML files and runs a bash script (e.g., [`test.sh`](https://github.com/Juniper/nita/blob/main/examples/evpn_vxlan_erb_dc/test.sh)) in a Robot Docker container to execute specified tests. |

In order to customize the `project.yaml` file for your NITA project, follow the instructions below, which relate to adapting the shell command in the relevant `build` section. This section is responsible for pulling the Docker or Kubernetes image from the public repository, downloading it to your local machine, and then executing it.

#### Step-by-Step Instructions for Customizing `project.yaml`

   **Locate the Example File**: Start by opening the example project (current use case) located at `examples/ebgp_wan/project.yaml`.

   **Edit the Build Section**:
   - Find the `configuration` section of the YAML file. The key part of this section includes a shell command that specifies which Docker image to use. 
   - The current command in the example is:
     ```shell
      shell_command: 'write_yaml_files.py; python3 create_ansible_job_k8s.py build juniper/nita-ansible:22.8-2; kubectl apply -f build.yaml; kubectl wait --for=jsonpath="{.status.ready}"=1 job/build; kubectl logs job/build -f'
     ```
   - This command pulls the public Juniper image `juniper/nita-ansible` with the associated tag `22.8-2` from Docker Hub using the command `docker pull juniper/nita-ansible`.

   **Modify the Image**:
   - If you need to modify the existing image, you can log into the container locally, make adjustments, and save the new version of the image locally. After making your changes, you must push the modified image to Docker Hub to make it publicly available.
   - Ensure you tag your image appropriately. For example, if your organization is `acme`, and the new image is `ansible-dynamite` with a tag of `3.2.1.000`, you would need to tag and push it as follows:
     ```shell
     docker tag `local-image-name` acme/ansible-dynamite:3.2.1.000
     docker push acme/ansible-dynamite:3.2.1.000
     ```

   **Updating the Shell Command**:
   - After successfully uploading your modified image to Docker Hub, return to the `project.yaml` file to update the build command with your new image details.
   - Here’s how to modify the line in the `project.yaml`:
     ```yaml
     # Original line
     # shell_command: 'write_yaml_files.py; python3 create_ansible_job_k8s.py build juniper/nita-ansible:22.8-2; ...

     # Updated line
     shell_command: 'write_yaml_files.py; python3 create_ansible_job_k8s.py build acme/ansible-dynamite:3.2.1.000; ...
     ```

#### Complete Example of the Updated `project.yaml`

Here’s how the relevant section would look after modifications:

```yaml
description: Simple IPCLOS based WAN setup to link two data centers
name: ebgp_wan_0.3
action:
  - name: Build
    jenkins_url: build_vmx_wan
    # Must be one of NOOB, BUILD or TEST
    category: BUILD
    configuration:
      shell_command: 'write_yaml_files.py; python3 create_ansible_job_k8s.py build acme/ansible-dynamite:3.2.1.000; kubectl apply -f build.yaml; kubectl wait --for=jsonpath="{.status.ready}"=1 job/build; kubectl logs job/build -f'
```

----

### Playbook Updates - `sites.yaml`

To ensure successful configuration management in the NITA project, it’s important to update the `sites.yaml` file located in the "build" folder. This file serves a critical role in directing Ansible roles as well as specifying the devices against which the playbooks will be executed. Depending on your specific environmental needs, you may have to modify these `roles` and `hosts` accordingly.

#### Steps to Update `sites.yaml`

  **Locate the File**: 
   - Navigate to the example project structure and open the `build` folder.
   - Find and open `sites.yaml` in an editor of your choice.

  **Understand the Structure**: 
   - The `sites.yaml` file outlines which roles are assigned to each group of devices. It is essential to understand the existing configuration to make effective changes.
   - Below is an existing example taken from the `ebgp_dc` project:

   ```yaml
   ---
   # ****************************************************************************************************** #
   #                                                                                                        #
   # Project: nita                                                                                          #
   #                                                                                                        #
   # Copyright (c) Juniper Networks, Inc., 2021. All rights reserved.                                       #
   #                                                                                                        #
   # Notice and Disclaimer: This code is licensed to you under the Apache 2.0 License (the "License").      #
   # You may not use this code except in compliance with the License. This code is not an official          #
   # Juniper product. You can obtain a copy of the License at                                               #
   # https://www.apache.org/licenses/LICENSE-2.0.html                                                       #
   #                                                                                                        #
   # SPDX-License-Identifier: Apache-2.0                                                                    #
   #                                                                                                        #
   # Third-Party Code: This code may depend on other components under separate copyright notice and license #
   # terms. Your use of the source code for those components is subject to the terms and conditions of the  #
   # respective license as noted in the Third-Party source code file.                                       #
   #                                                                                                        #
   # ****************************************************************************************************** #
   - import_playbook: ../make_clean.yaml
   - import_playbook: ../make_etc_hosts.yaml

   - hosts: switches
     pre_tasks:
       connection: local
     roles:
       - { role: junos_common }
       - { role: junos_qfx_common }

   - hosts: firewalls
     pre_tasks:
       connection: local
     roles:
       - { role: junos_common }
       - { role: srx_common }

   - hosts: switches:firewalls
     connection: local
     gather_facts: no
     roles:
       - { role: junos_commit_config }
       - { role: build_config }
   ```

  **Modify Roles and Hosts**:
   - The role `junos_common` is currently utilized for all hosts under the `switches` category. If the new environment does not use firewalls, you can comment out or remove these related sections.
   - **To comment out sections**: Add `#` at the beginning of the line. This tells Ansible to ignore those lines during execution. 
   - **To remove a role**: Simply delete the corresponding entry from the `roles:` list.

   #### Example of Updated `sites.yaml`
   After modifications for a static route setup, the `sites.yaml` might look like this (_note the new section at the bottom of the file_):

   ```yaml
   ---
   # ********************************************************
   # Project: nita
   # ...
   # ********************************************************
   - import_playbook: ../make_clean.yaml
   - import_playbook: ../make_etc_hosts.yaml

   - hosts: switches
     pre_tasks:
       connection: local
     roles:
       - { role: junos_common }
       - { role: junos_qfx_common }

   # Removed firewall roles since they are not used in the test environment
   # - hosts: firewalls
   #   pre_tasks:
   #     connection: local
   #   roles:
   #     - { role: junos_common }
   #     - { role: srx_common }

   - hosts: routers
     connection: local
     gather_facts: no
     roles:
       - { role: routing_options }
   ```

- **Ensure Roles Are Correctly Defined**: Before executing the updated playbook, double-check that the roles are properly configured in the inventory. Any missing roles will cause failures in playbook execution.

### Manually Testing Static Route Implementation

For the static route implementation, we want to verify that the static route has been correctly configured on a Junos device. The specific command we intend to set via the automation process is:

```plaintext
set routing-options static route 0.0.0.0/0 next-hop 123.123.123.123
```

It’s important to manually validate this configuration is working as expected on the device. Here is how you can do that:

#### Steps for Manual Testing

1. **Log into the Junos Device**: Start by accessing your Junos device. For example, if you are testing on a device named `dc1-spine2`, you can log in using SSH.

   ```shell
   ssh user@dc1-spine2 {or dc1-spine2 ip}
   ```

2. **Verify the Configuration**: verify that the static route has been applied correctly by checking the routing table:

   ```shell
   user@dc1-spine2# show route 
    
    inet.0: 7 destinations, 7 routes (7 active, 0 holddown, 0 hidden)
    + = Active Route, - = Last Active, * = Both
    
    0.0.0.0/0          *[Static/5] 3w2d 06:03:36, metric 0
                        >  to 192.168.1.1 via ge-0/0/0.0
   ```

   Look for the entry corresponding 0.0.0.0/0 static route, and ensure that it is present and marked appropriately.

----

# Building a Project Zip File

Now that you understand the items required to build a new project, we need to upload this project data to NITA as a zip file, this project zip file contains all of the playbooks and templates that you need. In this section, we will look at how to create a zip file for the static route implementation example.

### Project Specific Files

The contents of the zip file will obviously vary depending upon what you want the NITA project to achieve; however, there are specific items that you should always include. Generally speaking, all of the magic happens in the `project.yaml` file, so pay particular attention to this file and to any Jinja2 templates you have created. 


### Creating the Zip File

To create a zip file for your project, follow these steps:

1. **Organize Your Files**: Make sure your project directory includes all required files:
   - `project.yaml`
   - Any other playbooks, scripts, and templates referenced in the  `project.yaml`.

2. **Navigate to Your Project Directory**:
   Open a terminal session and use the `cd` command to change to your project directory:

  ```bash
   cd /path/to/your/project-directory
  ```

Create the Zip File: Use the zip command to create a zip file. (_replace project_name.zip with your desired zip file name_):

```bash
zip -r project_name.zip *
```

The -r option tells zip to include all files and subdirectories in the current directory.

Upload the Zip File: Now that you have created your zip file, you can upload it to NITA. Follow the upload procedure specified in the [NITA - Examples](https://github.com/Juniper/nita/blob/main/docs/nita-examples.md).

----

# Reference Example
For a detailed reference on how all of this comes together, take a good look at the EVPN VXLAN or eBGP WAN example that we provide in the [GitHub repository](https://github.com/Juniper/nita). These examples showcases well-structured projects, and reviewing them can provide insights on best practices for your own project setup.

---- 

## Automating Tests

We've looked at how NITA can be used to automate the deployment of device configurations, next lets look at how we can add automated tests to verify that the deployments worked. NITA uses Robot Framework to execute tests which can use libraries such as `pybot` or our modified version `pybot_jrouter` and this gives a huge amount of flexibility for creating automated tests.

---- 

# TOOLS

The following third party sites provide some useful tools which you may find helpful...

| URL | Description |
|---|---|
| https://www.convertjson.com/yaml-to-json.htm | Convert YAML to JSON |
| https://j2live.ttl255.com | Online Jinja2 Parser and Renderer |
| https://codebeautify.org/yaml-parser-online | YAML parser |
| https://jsonformatter.org/ | JSON formatter |

---- 

# VERSION

This document is relevant for NITA version 22.8.

---- 

# SEE ALSO
[nita-cmd](https://github.com/Juniper/nita/blob/main/docs/nita-cmd.md)

