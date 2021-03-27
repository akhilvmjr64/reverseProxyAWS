# Reverse Proxy in AWS

Now here in this article I will be showing how I configured the reverse proxy over webserver on AWS by following the steps below using ansible playbook

#### [Firstly before reading the article have a look at the ansible dynamic inventory documentation for the AWS ec2 instance](https://docs.ansible.com/ansible/latest/collections/amazon/aws/aws_ec2_inventory.html)

Now we need to configure the reverse proxy, we know that ansible uses ssh to manage the managed nodes from the master node. Now the problem here is AWS provides ec2-user but to install anything we need root privileges hence we need to do previlege escalation from ec2-user to root as AWS provides ec2-user as the default user in AWS. In AWS we create a keypair at the time of creation of the instance we use this to login to the instance from remote location. With this private key we can login to the instance as ec2-user and then esaclate the privileges to root user. To login to the ec2-instance with private key via ansible we need to specify the path of the key either at the time when we are running the `ansible-playbook` command or in the `ansible.cfg` file in the present working directory or in the `.ansible.cfg` file in the home directory of the user. Then to do privilege escalation we need to use `become` keyword either in the ansible config file or while running the command `ansible-playbook` we need to also mention to which user we want to escalate to. Following is the `ansible.cfg` file that I have created in the working directory to do all the things that I mentioned above

```conf
[defaults]
inventory=./inventory
remote_user=ec2-user
host_key_checking=false
private_key_file=./ansible_aws.pem
become=yes
become_user=root
```

Here in the above configuration file as I have used the path to the inventory as `./inventory` in this directory we need place the inventory files. Before configuring the AWS instances we need to create the inventory file first, for this I have use the dynamic inventory which will automatically get the ip address or DNS hostname based on our declaration in the inventory file. I have grouped the instances based on the value of name of the instances. Following is the inventory file that I have used(**Note that this dynamic inventory filename should end with either .aws_ec2.yml or .aws_ec2.yaml**).

```yaml
aws_access_key: XXXXXXXX
aws_secret_key: XXXXXXXXXXXX
plugin: amazon.aws.aws_ec2
keyed_groups:
        - key: tags['Name']
regions:
        - us-east-1
```

The inventory file above will use the value of name attribute of the instance and adds hosts into that host group. I have name the instances as `servers` and `proxy` which I want to configure as webservers and proxy servers respectively (**Note if you are using the same configuration files from my github repo then use the same names for the instances**). 

Following is to show that I have create the instances with the names mentioned above
![aws_instances.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1616868226022/IF3bS_EOb.png)

Now after creating the dynamic inventory we can configure the ec2 instances. As we used the names for the EC2 instances as servers and proxy we will have two groups `_servers` and `_proxy`. I have used these group names to configure the instances in those groups as `servers` or `proxy` server

### Configuring the instances as web servers:
To cofigure the instances as web servers we need to install httpd and php and then configure httpd for that I followed the below steps:

- We need to configure yum as I am using RedHat as the OS in AWS instance
- We need to install the http and php softwares
- Copy the webpages
- Configure the httpd
- Start the httpd service

I have done these using the part of play book below:
```yaml
- hosts : _servers
  vars_files :
          - ./vars.yml
  tasks :
          - name : "installing httpd package for server"
            package :
                  name : "httpd"
                  state : present
          - name : "installing php package for server"
            package :
                  name : "php"
                  state : present
          - name : "removing the index.html file on webserver"
            file :
                  path : "/var/www/html/index.html"
                  state : absent
          - name : "copying the php file to the webserver"
            copy :
                  src : "./index.php"
                  dest : "/var/www/html/index.php"
          - name : "copying the httpd configuration file for the server"
            template :
                  src : "./httpd.conf"
                  dest : "/etc/httpd/conf/httpd.conf"
            notify : "restarting httpd service"
          - name : "setting the selinux to permissive mode"
            shell : "setenforce 0"
          - name : "killing the service running the {{server_port}} port"
            shell : "kill `netstat -tnlp | grep :{{server_port}} | awk '{print $7}' | awk -F/ '{print $1}'`"
            ignore_errors : yes
          - name : "starting the httpd service"
            service :
                  name : "httpd"
                  state : started
  handlers :
          - name : "restarting httpd service"
            service :
                  name : "httpd"
                  state : restarted
```
In the above part of playbook I have used the configuration files `httpd.conf` which can be seen  [here](https://github.com/akhilvmjr64/reverseProxyAWS/blob/main/httpd.conf) , I have used the variable file which contain the definitions for the port numbers of the proxy service and the httpd service which can be seen  [here](https://github.com/akhilvmjr64/reverseProxyAWS/blob/main/vars.yml) , I have also used the index.php file which looks as follows
```php
<pre>
<?php
print `/usr/bin/ifconfig`
?>
</pre>
```
All these files will be copied from the master node to the managed nodes while running the command

### Configuring the instances as proxy servers:
To cofigure the instances as proxy servers we need to install haproxy and then configure haproxy for that I followed the below steps:

- We need to configure yum as I am using RedHat as the OS in AWS instance
- We need to install haproxy software
- Configure the haproxy by copying the `haproxy.cfg` file to the proxy server
- Start the haproxy service

I have done the above using the part of play book below:
```yaml
- hosts : _proxy
  vars_files :
          - ./vars.yml
  tasks :
          - name : "installing haproxy on proxy server"
            package :
                  name : "haproxy"
                  state : "present"
          - name : "copying the haproxy configuration file to proxy server"
            template :
                  src : "./haproxy.cfg"
                  dest : "/etc/haproxy/haproxy.cfg"
            notify : "restarting haproxy service"
          - name : "setting selinux to permissive mode"
            shell : "setenforce 0"
          - name : "killing the service running on {{proxy_port}} port"
            shell : "kill `netstat -tnlp | grep :{{proxy_port}} | awk '{print $7}' | awk -F/ '{print $1}'`"
            ignore_errors : yes
          - name : "starting haproxy service"
            service :
                  name : "haproxy"
                  state : started
          - debug :
                  msg : "At {{groups['_proxy']}}:{{proxy_port}} proxy server is running"
  handlers :
          - name : "restarting haproxy service"
            service :
                  name : "haproxy"
                  state : restarted
```
In the above part of playbook I have used the configuration files `haproxy.cfg` in this file I have use the jinja templating by which the webserver IP address will be known to the proxy server following is the jinja templating I used which will locate the webservers
```conf
{% for i in groups['_servers'] %}
    server  app{{loop.index}} {{i}}:{{server_port}} check
{% endfor %}
```
The complete configuration file can be seen  [here](https://github.com/akhilvmjr64/reverseProxyAWS/blob/main/haproxy.cfg) , I have used the variable file which contain the definitions for the port numbers of the proxy service and the httpd service which can be seen  [here](https://github.com/akhilvmjr64/reverseProxyAWS/blob/main/vars.yml) . All these files will be copied from the master node to the managed nodes while running the `ansible-playbook` command

Finally we need to run the playbook using the command `ansible-plabook load_balancer.yml -b` here -b option is used as we are doing privilege escalation

#### Note : For this practical as we need are allowing certain port either on the proxy server or in the webserver you need to have security group which allows all traffic also remember the names of the instances must be servers(for the webserver instances) and proxy(for the proxy server instances) changing these you need to make changes at many places in the playbooks and configuration files

## Webpage Final output :
![Web Page Output 1](https://cdn.hashnode.com/res/hashnode/image/upload/v1616874760099/SE2AwHC61.png)


![Web Page Output 2](https://cdn.hashnode.com/res/hashnode/image/upload/v1616874839148/hJNe0CLn0.png)

### If you want to know how to configure reverse proxy locally you can have a look at it [here](https://hashnode.com/post/configuring-reverse-proxy-using-ansible-playbooks-ckmryb49u07zkl0s1clewegym)

If everything works fine the output will be as follows
```out

PLAY [all] **************************************************************************************************************************************************************

TASK [Gathering Facts] **************************************************************************************************************************************************
ok: [ec2-3-84-238-79.compute-1.amazonaws.com]
ok: [ec2-34-224-62-167.compute-1.amazonaws.com]
ok: [ec2-3-90-70-104.compute-1.amazonaws.com]

TASK [configuring yum for all the hosts to install httpd, php and haproxy] **********************************************************************************************
changed: [ec2-34-224-62-167.compute-1.amazonaws.com]
changed: [ec2-3-90-70-104.compute-1.amazonaws.com]
changed: [ec2-3-84-238-79.compute-1.amazonaws.com]

PLAY [_servers] *********************************************************************************************************************************************************

TASK [Gathering Facts] **************************************************************************************************************************************************
ok: [ec2-3-90-70-104.compute-1.amazonaws.com]
ok: [ec2-3-84-238-79.compute-1.amazonaws.com]

TASK [installing httpd package for server] ******************************************************************************************************************************
changed: [ec2-3-90-70-104.compute-1.amazonaws.com]
changed: [ec2-3-84-238-79.compute-1.amazonaws.com]

TASK [installing php package for server] ********************************************************************************************************************************
changed: [ec2-3-90-70-104.compute-1.amazonaws.com]
changed: [ec2-3-84-238-79.compute-1.amazonaws.com]

TASK [removing the index.html file on webserver] ************************************************************************************************************************
ok: [ec2-3-90-70-104.compute-1.amazonaws.com]
ok: [ec2-3-84-238-79.compute-1.amazonaws.com]

TASK [copying the php file to the webserver] ****************************************************************************************************************************
changed: [ec2-3-90-70-104.compute-1.amazonaws.com]
changed: [ec2-3-84-238-79.compute-1.amazonaws.com]

TASK [copying the httpd configuration file for the server] **************************************************************************************************************
changed: [ec2-3-90-70-104.compute-1.amazonaws.com]
changed: [ec2-3-84-238-79.compute-1.amazonaws.com]

TASK [setting the selinux to permissive mode] ***************************************************************************************************************************
changed: [ec2-3-90-70-104.compute-1.amazonaws.com]
changed: [ec2-3-84-238-79.compute-1.amazonaws.com]

TASK [killing the service running the 8084 port] ************************************************************************************************************************
fatal: [ec2-3-90-70-104.compute-1.amazonaws.com]: FAILED! => {"changed": true, "cmd": "kill `netstat -tnlp | grep :8084 | awk '{print $7}' | awk -F/ '{print $1}'`", "delta": "0:00:00.045791", "end": "2021-03-27 19:14:33.136567", "msg": "non-zero return code", "rc": 2, "start": "2021-03-27 19:14:33.090776", "stderr": "kill: usage: kill [-s sigspec | -n signum | -sigspec] pid | jobspec ... or kill -l [sigspec]", "stderr_lines": ["kill: usage: kill [-s sigspec | -n signum | -sigspec] pid | jobspec ... or kill -l [sigspec]"], "stdout": "", "stdout_lines": []}
...ignoring
fatal: [ec2-3-84-238-79.compute-1.amazonaws.com]: FAILED! => {"changed": true, "cmd": "kill `netstat -tnlp | grep :8084 | awk '{print $7}' | awk -F/ '{print $1}'`", "delta": "0:00:00.047703", "end": "2021-03-27 19:14:33.194425", "msg": "non-zero return code", "rc": 2, "start": "2021-03-27 19:14:33.146722", "stderr": "kill: usage: kill [-s sigspec | -n signum | -sigspec] pid | jobspec ... or kill -l [sigspec]", "stderr_lines": ["kill: usage: kill [-s sigspec | -n signum | -sigspec] pid | jobspec ... or kill -l [sigspec]"], "stdout": "", "stdout_lines": []}
...ignoring

TASK [starting the httpd service] ***************************************************************************************************************************************
changed: [ec2-3-90-70-104.compute-1.amazonaws.com]
changed: [ec2-3-84-238-79.compute-1.amazonaws.com]

RUNNING HANDLER [restarting httpd service] ******************************************************************************************************************************
changed: [ec2-3-90-70-104.compute-1.amazonaws.com]
changed: [ec2-3-84-238-79.compute-1.amazonaws.com]

PLAY [_proxy] ***********************************************************************************************************************************************************

TASK [Gathering Facts] **************************************************************************************************************************************************
ok: [ec2-34-224-62-167.compute-1.amazonaws.com]

TASK [installing haproxy on proxy server] *******************************************************************************************************************************
changed: [ec2-34-224-62-167.compute-1.amazonaws.com]

TASK [copying the haproxy configuration file to proxy server] ***********************************************************************************************************
changed: [ec2-34-224-62-167.compute-1.amazonaws.com]

TASK [setting selinux to permissive mode] *******************************************************************************************************************************
changed: [ec2-34-224-62-167.compute-1.amazonaws.com]

TASK [killing the service running on 80 port] ***************************************************************************************************************************
fatal: [ec2-34-224-62-167.compute-1.amazonaws.com]: FAILED! => {"changed": true, "cmd": "kill `netstat -tnlp | grep :80 | awk '{print $7}' | awk -F/ '{print $1}'`", "delta": "0:00:00.047987", "end": "2021-03-27 19:19:57.476808", "msg": "non-zero return code", "rc": 2, "start": "2021-03-27 19:19:57.428821", "stderr": "kill: usage: kill [-s sigspec | -n signum | -sigspec] pid | jobspec ... or kill -l [sigspec]", "stderr_lines": ["kill: usage: kill [-s sigspec | -n signum | -sigspec] pid | jobspec ... or kill -l [sigspec]"], "stdout": "", "stdout_lines": []}
...ignoring

TASK [starting haproxy service] *****************************************************************************************************************************************
changed: [ec2-34-224-62-167.compute-1.amazonaws.com]

TASK [debug] ************************************************************************************************************************************************************
ok: [ec2-34-224-62-167.compute-1.amazonaws.com] => {
    "msg": "At http://ec2-34-224-62-167.compute-1.amazonaws.com:80 proxy server is running"
}
```
