# Build your own component!

A pretty common usecase for ResearchCloud components is to install and run a webapplication, and serve it to the outside world. Doing so requires at least the following steps to performed in your playbook:

1. Installing dependencies.
1. Installing the application.
1. Creating a system service definition for the application, so it reloads when the workspace is restarted.
1. Running the application (via the system service), so that it listens on `localhost`
1. Using a reverse proxy to pass on incoming requests to the workspace's FQDN to the application running on `localhost`.
  * This requires a webserver to be installed. We'll be using Nginx on ResearchCloud, which is available in its own component.

Follow the [Preparation](#preparation) and [Development](#development) instructions below to get started!

Here are two example web applications that you could try to get running in your component:

* ASReview
* Ollama API

### ASReview

ASReview is an application that leverages machine learning to make the performance of systematic reviews more efficient. The exercise:

* Install and run the ASReview web application
* Configure an Nginx reverse proxy for the application, enabling SRAM authentication on the workspace.

### Ollama API

Using [Ollama](https://github.com/ollama/ollama) you can download and run various LLMs and interact with them in various ways, including using a [REST API](https://github.com/ollama/ollama?#rest-api). The exercise:

* Install and run Ollama
  * allow the user to specify different models that should be loaded using a ResearchCloud parameter.
* Configure Nginx to reverse proxy the REST API. Protect the route with HTTP basic authentication.

## Preparations

### Fork the template repository

https://github.com/UtrechtUniversity/src-component-galaxy/

### Create a Component in the portal

1. Login to the ResearchCloud portal
1. Go to Catalog > Components and create a new one using the '+' button.
    * Choose script type 'Ansible Playbook'
    * Fill in the required details
    * Add parameters to your component

**Note**: you can of course come back to edit your component and its parameters. When you do so, remember that you'll need to [promote your changes to the *Live* version of the component](https://servicedesk.surf.nl/wiki/pages/viewpage.action?pageId=102826582)!

### Create a Catalog Item

For the purposes of this tutorial, you can simply find the *UU Demo Test* Catalog Item and Clone it, then add your custom component to it.

**Note**: as you will see, the *UU Test Demo* Catalog Item already contains the three standard SURF components, as well as Nginx. Nginx will be configured to allow authorization/authentication using SRAM/Single-sign on.

### Pull and start the the test container

Since we'll be testing a webapplication that will be served with nginx, you can use a special flavour of the [test container](https://github.com/UtrechtUniversity/SRC-test-workspace/) that already has the *SRC-Nginx* component installed on it! Just pull the following image:

`podman pull ghcr.io/utrechtuniversity/src-test-workspace:ubuntu_jammy-nginx` (or use `docker` instead)

**Note**: the container comes with nginx installed just as it would be on a workspace. However, for testing purposes, it has SSL disabled and is not actually configured to perform authentication using an external auth server.

For testing purporses, it will be useful to publish the port on the container on which Nginx is listening (`80`) to a port on your host machine. Try running this command:

`podman run -p 8080:8080 -d --name src_component_test -v $(pwd):/etc/rsc/my_component ghcr.io/utrechtuniversity/src-test-workspace:ubuntu_jammy /sbin/init`

If all goes well, you will be able to open a browser and load http://localhost:8080 to connect to Nginx on the container. Of course, nothing will actually be served yet.

### Edit your component's code and test

Make your changes, then apply them to the test container using:

`podman exec src_component_test run_component.sh /etc/rsc/my_component/playbook.yml` (from the same directory as your playbook)

## Development

### Installing dependencies

Look up your application's dependencies:

1. Any required system packages should be installed with the [package module](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/package_module.html)
1. Any required Python packages should probably be installed in a virtualenv. Use the [pip module](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/pip_module.html)!
1. Similarly for other kinds of depencies (Node, etc.)

### Installing the application

Depending on the preferred installation method of your application: [clone](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/git_module.html), [download](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/get_url_module.html), or use a [package manager](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/package_module.html) to install it.

### Create a system service definition for the application

At this point, you *could* just run your application (for instance, by issuing ). However, what happens if the workspace is restarted? Your application won't simply come up again!

To ensure the application is restarted when needed:

1. create a service unit file.
1. copy it to correct location
1. use the `systemd_service` module to enable and start the service

### Let Nginx serve your application using a reverse proxy

This could be complicated, but fortunately, you can use the `uusrc.general.nginx_reverse_proxy` role for this! See [here](https://utrechtuniversity.github.io/researchcloud-items/playbooks/reverse_proxy.html) for documentation.

1. Apply the role in your playbook and pass in the variables necessary to set up a reverse proxy to port 5000 (assuming that is the port your webapp is using) on the container.
1. Run your playbook on the container, and try connecting to http://localhost:8080

If this works, try enabling various forms of authentication:

1. Use the `auth: sram` attribute to enable SRAM authorization and Single-Sign on on the workspace.
    * Note: this won't actually work on the test container.)
1. Use the `auth: basic` attribute to enable HTTP basic username/password authentication.

# Deploy on ResearchCloud

When you're ready to deploy on ResearchCloud, don't forget to `git push` to your component's repository first!
