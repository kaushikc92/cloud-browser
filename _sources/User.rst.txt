.. _user:

***********
User Manual
***********

Overview
========

Cloud browser is a tool for easy visualization of large csv files. This tool allows you to upload csv files and view
them as a map i.e. perform pan and zoom operations through UI. Cloud browser processes csv files on the fly and has a
low latency for initial loading and browsing as compared to traditional tools such as Apple Numbers and Microsoft Excel.
It is available as a micro-cloud service, hence can be hosted in one place and then the users do not need to worry about
any requirements or dependencies other than a web browser.

Requirements
============

Cloud browser is configured to use a maximum of 30 GB of disk space. This tool has been tested for the linux
environment. In order to set up cloud browser on a host, a python 3 environment will be required which could be native,
virtual or a container. Python pip is required for installing dependencies. Wkhtmltopdf package needs to be installed on
the machine. Write permissions are required on the 'media' folder where temporary data will be stored.

Installation
============

* First get account credentials for an ec2 Linux machine and set a security policy that will allow access to port 80 of
the machine. 

* After ssh into the amazon box, update packages and install docker ::

    sudo yum update -y
    sudo yum install -y docker

* Start Docker service ::

    sudo service docker start

* Run the docker container ::
    
    sudo docker run -p 8000:8000 kaushikc92/cloud_browser:v1

* Install and start nginx ::

    sudo yum install nginx
    sudo service nginx start

* Edit the nginx configs ::
    
    sudo vi /etc/nginx/conf.d/virtual.conf

* Add the following line to the above file inside the http object ::
    
    client_max_body_size 200M;

* Restart nginx to reflect new changes ::

    sudo service nginx restart

* You could also build your own docker image from the source code and run it in the docker container.

* Another option is to clone the source code and host it elsewhere without using docker.



Usage
=====

This tool is pretty straight forward to use. The index page opens up a file manager portal where you can upload a csv
file, choose to browse one of the csv files already uploaded, delete a file or download a file. Clicking on the csv file
will open the file in a map window where you can use pan and zoom operations to browse it. A slider is available to the
right of the map window which can be used to quickly jump to any portion of a large csv file.
