# Build Your Own Component

This document contains two exercises for creating ResearchCloud components. Follow the step-by-step instructions to build and practice with components.

*The exercises below are designed such that you can test them with Ansible locally (in case of the second exercise, by using a special test container). If you want to test using Ansible on a ResearchCloud VM, beware that Ansible is installed in a virtual environment. To activate it: `source /etc/src/venv/src-venv/bin/activate`.*

## Exercises:
1. [Check for GPUs using Ansible facts](#exercise-1-check-for-gpus-using-ansible-facts)
2. [Set up a Web Application](#exercise-2-set-up-a-web-application)

---

## Exercise 1: Check for GPUs Using Ansible Facts

In this exercise, you'll learn how to work with Ansible and Jinja filters to determine if a GPU is available and retrieve related information.

### Objective

Develop skills in:
- Generating and parsing system information using Ansible
- Using Jinja filters for templating
- Creating custom variables to determine GPU availability

---

#### Practice Environment
While creating this playbook, you can:
- Run it locally.
- Test it on a ResearchCloud VM.
- Use an online Jinja templating tool, such as [this one](https://j2live.ttl255.com/) (select the “Ansible” option).

---

### Steps

#### Step 1: Identify GPU Information
1. Write a playbook to execute the `lshw` command and return GPU-related information as JSON.  
   * **Hint**: Use the `-class display` option in `lshw` to narrow output to GPU/display hardware.
   * **Hint**: If you cannot use `lshw` on a machine with GPU, see below for example output.
   
   Example command:  
   ```bash
   lshw -class display -json
   ```

2. Use Ansible's JSON parsing feature (or Jinja filters) to process the JSON output.

3. From the parsed data:
   - Identify entries where `description` contains "3d" or "gpu" (these are likely GPUs).
   - Store the GPU entries in a variable named `fact_gpus`.

4. Create a flag variable called `has_gpu`, which should be `true` if at least one GPU is detected.

<details>
<summary><b>Example Playbook Logic</b></summary>

Use the following Ansible modules:
- Use the `command` module to run `lshw`.
- Use Jinja to filter and create the `fact_gpus` and `has_gpu` variables.
- See solution Ansible tasks [here](https://github.com/UtrechtUniversity/researchcloud-items/blob/main/playbooks/roles/fact_workspace_info/tasks/main.yml#L51-L97).

</details>


<details>
<summary><b>Example lshw output on a machine with GPU</b></summary>
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
</details>

---

#### Step 2: Retrieve CUDA Version (if applicable)
If a GPU is found and CUDA is installed, retrieve the installed CUDA version.

1. Use the `nvidia-smi` command to obtain the CUDA version.  
   Example command:  
   ```bash
   nvidia-smi --version
   ```

2. Parse the command output to extract the `CUDA Version` string and store it in an Ansible variable.

*Example Output of `nvidia-smi --version`:*
```bash
NVIDIA-SMI version  : 595.71.05
NVML version        : 595.71
DRIVER version      : 595.71.05
CUDA Version        : 13.2
```

<details>
<summary><b>Example Playbook Logic</b></summary>

Use the following Ansible modules:
- Use the `command` module to run `nvida-smi`.
- Use Jinja to filter out the version on the line containing 'CUDA Version'.
- See solution Ansible tasks [here](https://github.com/UtrechtUniversity/researchcloud-items/blob/main/playbooks/roles/fact_workspace_info/tasks/main.yml#L51-L97).

</details>

## Exercise 2: Set Up a Web Application

In this exercise, you'll deploy and serve a web application using Ansible. You'll learn how to configure the application to run in a workspace and use Nginx as a reverse proxy to expose it to users.

---

### Objective

You'll learn how to:
1. Set up and use a Python virtual environment (venv) to isolate your web application.
2. Deploy Python’s `http` module in-built webserver as an example app.
3. Define a `systemd` service to manage the application.
4. Configure Nginx as a reverse proxy to expose the application.

---

### Recommended Example

**Example Web Application:** Use Python's built-in `http` module's webserver. This webserver is simple, effective, and requires minimal setup. If you'd like to use a real-world application, you can substitute your app while following the same steps.

**Outcome for Python Example:**  
The Python server will allow users to view and browse the files in a directory. After completing the setup:
- Visiting `http://localhost:8080` through Nginx will show the directory's file listing served by Python.

To try the server locally, run:

```bash
python3 -m http.server
```

You will see:

```bash
Serving HTTP on :: port 8000 (http://[::]:8000/) ...
```

---

### Steps

#### Step 1: Prepare the Environment

1. **Create a repository:**  
   Clone the [SRC Component Template Repository](https://github.com/UtrechtUniversity/src-component-template) to get started.

2. **Add the `uusrc.general` collection requirement:**  
   Update the `requirements.yml` file in your repository with the following:

   ```yaml
   collections:
     - name: uusrc.general
   ```

3. **Create a ResearchCloud component:**  
   In the ResearchCloud portal:
   - Navigate to **Catalog > Components**.
   - Create a new Ansible Playbook component and define any parameters necessary.

4. **Set up a Catalog Item for testing:**  
   Use the `Ubuntu Nginx Development Base` Catalog Item as your development workspace and add your custom component to it.

5. **Start a test container with Nginx pre-installed:**  

   Pull the container:
   ```bash
   podman pull ghcr.io/utrechtuniversity/src-test-workspace:ubuntu_noble-nginx
   ```

   Run the container:
   ```bash
   podman run -p 8080:80 -d --name src_component_test -v $(pwd):/etc/rsc/my_component ghcr.io/utrechtuniversity/src-test-workspace:ubuntu_noble-nginx /sbin/init
   ```

---

#### Step 2: Develop Your Playbook

##### 1. Create a Virtual Environment
Use Ansible to create a Python virtual environment that will isolate your application.

<details>
<summary><b>Ansible Task</b></summary>

```yaml
- name: Install Python venv package
  ansible.builtin.package:
    name: python3-venv
    state: present

- name: Create a virtual environment for the web application
  ansible.builtin.command:
    cmd: python3 -m venv /opt/python_webapp_venv
    creates: /opt/python_webapp_venv
```

</details>

---

##### 2. Define a System Service for the Webserver

To ensure the Python webserver starts automatically on boot and is managed as a service, create a `systemd` unit file.

**Example `systemd` service template:**

```ini
[Unit]
Description=Python HTTP Server in Virtual Environment
After=network.target

[Service]
User=root
WorkingDirectory=/path/to/directory/to/serve
ExecStart=/opt/python_webapp_venv/bin/python -m http.server 5000
Restart=always

[Install]
WantedBy=multi-user.target
```

Modify `/path/to/directory/to/serve` to the directory you want served.

Then in your playbook, ensure this file is copied to the right location using the [copy module](https://docs.ansible.com/projects/ansible/latest/collections/ansible/builtin/copy_module.html):

<details>
<summary><b>Ansible Task for Copying the systemd Service</b></summary>

```yaml
- name: Deploy systemd service for Python web server
  ansible.builtin.copy:
    dest: /lib/systemd/system/python_webapp.service
    content: |
      [Unit]
      Description=Python HTTP Server in Virtual Environment
      After=network.target

      [Service]
      User=root
      WorkingDirectory=/path/to/directory/to/serve
      ExecStart=/opt/python_webapp_venv/bin/python -m http.server 5000
      Restart=always

      [Install]
      WantedBy=multi-user.target
```

</details>

Then use Ansible's [`systemd` module](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/systemd_module.html) to enable and start the service.

<details>
<summary><b>Ansible Task to Start the Service</b></summary>

```yaml
- name: Enable and start Python webserver
  ansible.builtin.systemd:
    name: python_webapp
    enabled: true
    state: started
```

</details>

---

##### 3. Configure Nginx as a Reverse Proxy

The `uusrc.general.nginx_reverse_proxy` role allows you to configure Nginx to forward incoming requests to your Python webserver.  

Include the `uusrc.general.nginx_reverse_proxy` role in the `roles:` section of your playbook like this:

```yaml
roles:
  - role: uusrc.general.nginx_reverse_proxy
    vars: # your variables below here -- see https://utrechtuniversity.github.io/researchcloud-items/roles/nginx_reverse_proxy.html
    # ensure nginx reverse proxies to the right port for your webapplication
```

---

#### Step 3: Test the Setup

1. Run your playbook in the test container:

   ```bash
   podman exec src_component_test run_component.sh /etc/rsc/my_component/playbook.yml
   ```

2. Open [http://localhost:8080](http://localhost:8080) in your browser. You should see Python's web server serving the directory.

If step 1 fails or step 2 does not produce the correct result, modify your playbook and run it again!

---

#### Step 4: Enable authentication

Configure the `nginx_reverse_proxy` role to enable authentication.

1. **Enable authentication:**
   - Use `auth: sram` for Single Sign-On via SRAM authentication (note: this won't work on the test container but will function on ResearchCloud).
   - Use `auth: basic` for HTTP basic authentication on the test container.

Repeat Step 3 to test!

---

#### Step 5: Deploy on ResearchCloud

When you're ready:
1. Push all changes to the component's repository:
   ```bash
   git push
   ```
2. Start a new ResearchCloud workspace with your Catalog Item.
3. When the workspace is ready, click the yellow 'Access' button. The browser should be redirected to your webapplication running in the cloud!
