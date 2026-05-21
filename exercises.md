# Build your own component!

Exercises:

- [GPU fact](#gpu-fact).
- [Webapplication](#Webapplication).

## GPU fact

This exercise is designed to practice with Ansible's use of the Jinja templating system.

*You don't need to create an actual component for this*. You can practice with a playbook that you run locally, or an a ResearchCloud VM. Or you can use an online Jinja playground like [this one](https://j2live.ttl255.com/) -- be sure to select the 'Ansible' radio button to use the Ansible set of filters.

Imagine that you have a playbook in which you want to do something conditionally on whether the user has selected a workspace with a GPU. You'd need a variable that determines whether a GPU is present or not. Generally, Ansible provided you with special variables called [facts](https://www.redhat.com/en/blog/playing-ansible-facts) that give you information about the system, such as the number of CPU cores, disks, network...

Sadly, there is (at time of writing) no Ansible fact that gives you information about the system's GPU's! This is likely because there is no simple, unified way to query the system and ask whether any GPU's are available.

But of course we can develop a (good enough) solution ourselves! On Linux, the `lshw` (list hardware) utility provides information that we can use.

### Step 1

Write a playbook that:

1. Uses the `lshw` command to output `json` information on all graphics-related hardware (*hint*: use the `-class display` option to limit output the relevant subset).
1. Use Jinja filters to parse the outputted `json` into an Ansible variable, and select all entries from the list that are GPUs (as an approximation, let's say that an entry is a GPU if it's `description` field contains the word '3d' or 'gpu').
1. Store this in a variable `fact_gpus`. Also create a variable `has_gpu` which is only true if there is at least one GPU.

Example output of `lshw -class display -json`:

```
[                           
  {
    "id" : "display:0",
    "class" : "display",
    "handle" : "PCI:0000:00:02.0",
    "description" : "VGA compatible controller",
    "product" : "GD 5446",
    "vendor" : "Cirrus Logic",
    "physid" : "2",
    "businfo" : "pci@0000:00:02.0",
    "version" : "00",
    "width" : 32,
    "clock" : 33000000,
    "configuration" : {
      "latency" : "0"
    },
    "capabilities" : {
      "vga_controller" : true
    }
  },
  {
    "id" : "display:1",
    "class" : "display",
    "claimed" : true,
    "handle" : "PCI:0000:00:06.0",
    "description" : "VGA compatible controller",
    "product" : "TU102 [GeForce RTX 2080 Ti]",
    "vendor" : "NVIDIA Corporation",
    "physid" : "6",
    "businfo" : "pci@0000:00:06.0",
    "version" : "a1",
    "width" : 64,
    "clock" : 33000000,
    "configuration" : {
      "driver" : "nvidia",
      "latency" : "0"
    },
    "capabilities" : {
      "vga_controller" : true,
      "bus_master" : "bus mastering",
      "cap_list" : "PCI capabilities listing",
      "rom" : "extension ROM"
    }
  }
]
```

### Step 2

Suppose we know there is a GPU and CUDA is installed. We'd like to know the relevant CUDA version. Use the `nvidia-smi` command and Jinja filters to create an Ansible variable that contains the CUDA version.

Example output of `nvidia-smi --version`:

```
NVIDIA-SMI version  : 595.71.05
NVML version        : 595.71
DRIVER version      : 595.71.05
CUDA Version        : 13.2
```

## Webapplication

A pretty common usecase for ResearchCloud components is to install and run a web application, and serve it to the outside world, using SRAM to provide Single Sign-on for users. Doing so requires the following steps to performed in your playbook:

1. Installing dependencies.
1. Installing the application.
   * Configuring the application, if needed.
1. Creating a system service definition for the application, so it reloads when the workspace is restarted.
1. Running the application (via the system service), so that it listens on `localhost`
1. Using a reverse proxy to pass on incoming requests to the workspace's FQDN to the application running on `localhost`.
  * This requires a webserver to be installed. We'll be using Nginx on ResearchCloud, which is available in its own component.

Follow the [Preparation](#preparations) and [Development](#development) instructions below to get started!

For this exercise you could use any web application that you like. If you have no application that you'd like to try yourself, you can simply use Python's `http` module's in-built fileserver as an app. Try it out locally first:

```
$ python3 -m http.server
Serving HTTP on :: port 8000 (http://[::]:8000/) ...
# navigate to http://localhost:8000 and see a file listing of the directory that you ran the command in!
```

## Preparations

### Create a repository from this the template repository and clone it locally

[https://github.com/UtrechtUniversity/src-component-template](https://github.com/UtrechtUniversity/src-component-template)

### Create a Component in the portal

1. Login to the ResearchCloud portal
1. Go to Catalog > Components and create a new one using the '+' button.
    * Choose script type 'Ansible Playbook'
    * Fill in the required details
    * Add parameters to your component

**Note**: you can of course come back to edit your component and its parameters later. When you do so, remember that you'll need to [promote your changes to the *Live* version of the component](https://servicedesk.surf.nl/wiki/pages/viewpage.action?pageId=102826582)!

### Create a Catalog Item

For the purposes of this tutorial, you can simply find the *UU Demo Test* Catalog Item and Clone it, then add your custom component to it.

**Note**: as you will see, the *UU Test Demo* Catalog Item already contains the three standard SURF components, as well as Nginx. Nginx will be configured to allow authorization/authentication using SRAM/Single-sign on.

### Pull and start the the test container

Since we'll be testing a webapplication that will be served with nginx, you can use a special flavour of the [test container](https://github.com/UtrechtUniversity/SRC-test-workspace/) that already has the *SRC-Nginx* component installed on it! Just pull the following image:

`podman pull ghcr.io/utrechtuniversity/src-test-workspace:ubuntu_jammy-nginx` (or use `docker` instead)

**Note**: the container comes with nginx installed just as it would be on a workspace. However, for testing purposes, it has SSL disabled and is not actually configured to perform authentication using an external auth server.

For testing purporses, it will be useful to publish the port on the container on which Nginx is listening (`80`) to a port on your host machine. Try running this command:

`podman run -p 8080:80 -d --name src_component_test -v $(pwd):/etc/rsc/my_component ghcr.io/utrechtuniversity/src-test-workspace:ubuntu_jammy /sbin/init`

If all goes well, you will be able to open a browser and load http://localhost:8080 to connect to Nginx on the container. Of course, nothing will actually be served yet.

### Edit and test

Make changes to your ansible playbook, then apply them to the test container using:

`podman exec src_component_test run_component.sh /etc/rsc/my_component/playbook.yml` (from the same directory as your playbook)

The `run_component.sh` script will automatically look for a file named `component_vars.yml` in the same directory as your playbook. You can use this file to mock ResearchCloud parameters.

*Hint*: for best testing, make all of your variables in `component_vars.yml` strings!

```yml
foo: 'true' # ResearchCloud turns all variables into strings, so to mock correctly, use quotes to set too the string 'true', instead of a YAML Boolean.
```

## Development

Below are steps that your playbook will probably (or definitely) need to execute in order to get your web application up and running. The steps are abstractly described, on purpose: try to figure out how you can use Ansible to do these things yourself!

### Installing dependencies

Look up your application's dependencies, if there are any. Required system packages should be installed with the [package module](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/package_module.html).

Depending on how your application is shipped, there may also be other kinds of dependencies (Node etc.).

### Installing the application

Depending on the preferred installation method of your application: [clone](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/git_module.html), [download](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/get_url_module.html), or use a [package manager](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/package_module.html) to install it.

If your application is shipped as a Python package, it should probably be installed in a virtualenv. Use the [pip module](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/pip_module.html)!

### Create a system service definition for the application

At this point, you *could* just run your application (for instance, by issuing a command like `python3 /path/to/my/app`). However, what happens if the workspace is restarted? Your application won't simply start up again!

To ensure the application is restarted when needed:

1. create a systemd unit file for your application.
1. copy it to correct location using the [copy module](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/copy_module.html).
   * A good location for a systemd file is probably `/lib/systemd/system/yourapp.service`.
1. use the [systemd module](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/systemd_module.html) to enable and start the service.

Here's a template for a systemd unit file:

```
[Unit]
Description=My application
After=network.target

[Service]
User=root
WorkingDirectory=/path/to/your/app/dir
ExecStart=<app start command>
Restart=always

[Install]
WantedBy=multi-user.target
```

### Let Nginx serve your application using a reverse proxy

This could be complicated, but fortunately, you can use the `uusrc.general.nginx_reverse_proxy` role for this! See [here](https://utrechtuniversity.github.io/researchcloud-items/playbooks/reverse_proxy.html) for documentation.

1. Apply the role in your playbook and pass in the variables necessary to set up a reverse proxy to port 5000 (assuming that is the port your webapp is using) on the container.
1. Run your playbook on the container, and try connecting to http://localhost:8080

If this works, try enabling various forms of authentication:

1. Use the `auth: sram` attribute to enable SRAM authorization and Single-Sign on on the workspace.
    * Note: this won't actually work on the test container.)
1. Use the `auth: basic` attribute to enable HTTP basic username/password authentication.

### ...and more

Of course, the above steps are not exhaustive. For example, maybe it would be nice if your application didn't run as root (you could create a dedicated user using the [user](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/user_module.html) module). But maybe give your new component a try on ResearchCloud first!

# Deploy on ResearchCloud

When you're ready to deploy on ResearchCloud, don't forget to `git push` to your component's repository first!
