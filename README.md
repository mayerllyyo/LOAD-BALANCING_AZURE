## Escuela Colombiana de Ingeniería

### Software Architectures - ARSW

## Scaling in Azure with Virtual Machines, Scale Sets, and Service Plans

## **Authors**

- **Santiago Hurtado Martínez** [SantiagoHM20](https://github.com/SantiagoHM20)

- **Mayerlly Suárez Correa** [mayerllyyo](https://github.com/mayerllyyo)

### Dependencies

* Create a free account in Azure. You can follow this [documentation](https://azure.microsoft.com/es-es/free/students/). Doing so will give you **$100 USD in credits for 12 months**.

### Part 0 - Understanding the Quality Scenario

Attached to this lab, you will find a fully developed application whose goal is to calculate the n-th value of the Fibonacci sequence.

**Scalability**
When a group of users concurrently requests an n-th Fibonacci number (greater than 1,000,000) under normal operating conditions, **all requests must be answered**, and the system’s **CPU usage must not exceed 70%**.

---

### Part 1 - Vertical Scalability

1. Go to the [Azure Portal](https://portal.azure.com/) and create a **Virtual Machine** with the basic characteristics shown in *Image 1*, corresponding to the following:

   * Resource Group = `SCALABILITY_LAB`
   * Virtual machine name = `VERTICAL-SCALABILITY`
   * Image = Ubuntu Server
   * Size = Standard B1ls
   * Username = `scalability_lab`
   * SSH public key = your public SSH key

![Image 1](images/part1/part1-vm-basic-config.png)

2. To connect to the VM, use the following command, replacing the `x` values with your VM’s IP address (check the *Connect* section of your created VM for more details):

   ```
   ssh scalability_lab@xxx.xxx.xxx.xxx
   ```

3. Install Node.js. Follow the section *Installing Node.js and npm using NVM* found in this [link](https://linuxize.com/post/how-to-install-node-js-on-ubuntu-18.04/).

4. To install the lab’s attached application, upload the `FibonacciApp` folder to a repository you can access and execute these commands inside the VM:

   ```
   git clone <your_repo>
   cd <your_repo>/FibonacciApp
   npm install
   ```

5. To run the app, you can use `node FibonacciApp.js`, but once you close the SSH connection, the app will stop running. To avoid that behavior, we’ll use **forever**. Run the following commands in the VM:

   ```
   node FibonacciApp.js
   ```

6. Before testing the endpoint, go to the *Networking* section in Azure and create an **Inbound port rule** as shown below.
   To verify that the app works, open a browser and access:
   `http://xxx.xxx.xxx.xxx:3000/fibonacci/6`
   The response should be:
   `The answer is 8`

![](images/part1/part1-vm-3000InboudRule.png)

7. The function that calculates the n-th Fibonacci number is inefficient and consumes a lot of CPU. Using the browser console, record the response times for these values:

   * 1000000
   * 1010000
   * 1020000
   * 1030000
   * 1040000
   * 1050000
   * 1060000
   * 1070000
   * 1080000
   * 1090000

8. Go to Azure and check the **CPU usage** for the VM. (The results may take 5 minutes to appear.)

![Image 2](images/part1/part1-vm-cpu.png)

9. Now, we’ll use **Postman** to simulate concurrent load on the system. Follow these steps:

   * Install **newman** with `npm install newman -g`. For more info, see [this link](https://learning.getpostman.com/docs/postman/collection-runs/command-line-integration-with-newman/).
   * Go to `FibonacciApp/postman` on a machine other than the VM.
   * In the `[ARSW_LOAD-BALANCING_AZURE].postman_environment.json` file, change the value of `VM1` to your VM’s IP.
   * Run the following commands:

     ```
     newman run ARSW_LOAD-BALANCING_AZURE.postman_collection.json -e [ARSW_LOAD-BALANCING_AZURE].postman_environment.json -n 10 &
     newman run ARSW_LOAD-BALANCING_AZURE.postman_collection.json -e [ARSW_LOAD-BALANCING_AZURE].postman_environment.json -n 10
     ```

10. CPU usage will be very high, and a large number of concurrent requests may cause the service to fail.
    To fix this, we’ll use **Vertical Scaling**.
    In Azure, go to the *Size* section and select size `B2ms`.

![Image 3](images/part1/part1-vm-resize.png)

11. Once the change takes effect, repeat steps **7, 8, and 9**.
12. Evaluate the quality scenario related to the non-functional scalability requirement and conclude whether this model of scalability meets it.
13. Return the VM to its original size to avoid additional charges.

---

**Questions**

1. How many and which resources does Azure create along with the VM?
2. Briefly describe the purpose of each resource.
3. When closing the SSH connection to the VM, why does the application stop running when we use `node FibonacciApp.js`? Why must we create an *Inbound port rule* before accessing the service?
4. Attach a table with response times and explain why the function takes so long.
5. Attach an image of CPU usage and explain why the function consumes that much CPU.
6. Attach the Postman execution summary image and interpret:

   * Execution times for each request.
   * Any failures—document and explain them.
7. What’s the difference between sizes `B2ms` and `B1ls` (not just hardware specs)?
8. Is increasing the VM size a good solution in this scenario? What happens to the FibonacciApp when changing the VM size?
9. What happens to the infrastructure when the VM size changes? What negative effects does it have?
10. Was there any improvement in CPU usage or response time? Why or why not?
11. Increase the number of parallel Postman executions to `4`. Is the system’s performance proportionally better?

---

### Part 2 - Horizontal Scalability

#### Creating the Load Balancer

Before continuing, delete the previous resource group to avoid additional costs and start fresh.

1. The Load Balancer is essential for enabling horizontal scalability. Create a Load Balancer in Azure as shown in the image:

![](images/part2/part2-lb-create.png)

2. Next, create a **Backend Pool**, following this example:

![](images/part2/part2-lb-bp-create.png)

3. Then, create a **Health Probe**, following this image:

![](images/part2/part2-lb-hp-create.png)

4. Next, create a **Load Balancing Rule**, as shown below:

![](images/part2/part2-lb-lbr-create.png)

5. Create a **Virtual Network** in the resource group, as shown:

![](images/part2/part2-vn-create.png)

---

#### Creating the Virtual Machines (Nodes)

Now, we’ll create **3 VMs** (VM1, VM2, and VM3) with standard public IPs in **3 different Availability Zones**. Then, we’ll add them to the Load Balancer.

1. In the basic configuration, follow the image below.
   Note the *Availability Zone*:

   * VM1 → Zone 1
   * VM2 → Zone 2
   * VM3 → Zone 3

![](images/part2/part2-vm-create1.png)

2. In the *Networking* configuration, make sure the previously created **Virtual Network** and **Subnet** are selected.
   Assign a public IP and enable **zone redundancy**.

![](images/part2/part2-vm-create2.png)

3. For the **Network Security Group**, select *Advanced* and configure it as shown.
   Don’t forget to create an *Inbound Rule* for port **3000**.
   When creating VM2 and VM3, you can reuse the existing Network Security Group.

![](images/part2/part2-vm-create3.png)

4. Assign this VM to the Load Balancer, as shown below:

![](images/part2/part2-vm-create4.png)

5. Finally, install the Fibonacci application on each VM by running:

```
git clone https://github.com/daprieto1/ARSW_LOAD-BALANCING_AZURE.git

curl -o- https://raw.githubusercontent.com/creationix/nvm/v0.34.0/install.sh | bash
source /home/vm1/.bashrc
nvm install node

cd ARSW_LOAD-BALANCING_AZURE/FibonacciApp
npm install

npm install forever -g
forever start FibonacciApp.js
```

Repeat this process for all 3 VMs.
For now, we’ll do it manually, but note that automation tools like **Azure Resource Manager**, **OSDisk Images**, **Terraform**, **Packer**, **Puppet**, or **Ansible** can automate this process.

---

#### Testing the Final Infrastructure

1. The access endpoint for the system will be the **Load Balancer’s public IP**.
   First, verify that basic services are working:

   ```
   http://52.155.223.248/
   http://52.155.223.248/fibonacci/1
   ```

2. Perform the same **load tests using newman** as in Part 1.
   Create a comparative report contrasting:

   * Response times
   * Successful requests
   * Costs of both infrastructures (horizontal load balancing vs. vertically scaled VM)

3. Add a **4th VM** and run the newman tests again, but this time with **4 parallel executions**.
   Create a report showing CPU usage for all 4 VMs and explain why the success rate increased with this scalability model.

```
newman run ARSW_LOAD-BALANCING_AZURE.postman_collection.json -e [ARSW_LOAD-BALANCING_AZURE].postman_environment.json -n 10 &
newman run ARSW_LOAD-BALANCING_AZURE.postman_collection.json -e [ARSW_LOAD-BALANCING_AZURE].postman_environment.json -n 10 &
newman run ARSW_LOAD-BALANCING_AZURE.postman_collection.json -e [ARSW_LOAD-BALANCING_AZURE].postman_environment.json -n 10 &
newman run ARSW_LOAD-BALANCING_AZURE.postman_collection.json -e [ARSW_LOAD-BALANCING_AZURE].postman_environment.json -n 10
```

---

**Questions**

* What are the types of Load Balancers in Azure, and how do they differ?
  What is an SKU, what types exist, and how do they differ?
  Why does the Load Balancer need a public IP?
* What is the purpose of the **Backend Pool**?
* What is the purpose of the **Health Probe**?
* What is the purpose of the **Load Balancing Rule**?
  What types of session persistence exist, why is it important, and how can it affect system scalability?
* What is a **Virtual Network**? What is a **Subnet**? What are *address space* and *address range* used for?
* What are **Availability Zones**, and why did we choose three different ones? What does it mean for an IP to be *zone-redundant*?
* What is the purpose of the **Network Security Group**?
* Attach **Newman report 1 (from step 2)**.
* Present the **Deployment Diagram** of the solution.
