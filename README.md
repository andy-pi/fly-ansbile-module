# Automating Deployment using Fly's API and Ansible

## Introduction  

In this article I'm going to show you how to create an ansible playbook that deploys your S3 static sites and configures Fly's global delivery network automatically. You'll be able to spin up fast, HTTPS protected sites in no time.

When it comes to deployment, Ansible is fast becoming the tool of choice for many DevOps engineers.  When I disovered the benefits of using Fly especially for HTTPS, I thought it would be a really useful to automate configuration of Fly so I've written an Ansible module to interact with their API which I explain how to use in the walkthrough below.  This combination allowed me to automate my complete deployment process, upload, DNS, Fly, everything.  

It could be that you're responsible for the deployment of static sites on a regular basis and you're looking to use Fly to provision an automatically renewing Let's Encrypt SSL certificate. Or perhaps your app is composed of many microservices and you want to construct your API using Fly rules.  We can now automate the deployment of these scenarios with Ansible!

Let's use an example to demonstrate why this is useful. Imagine you're a designer-developer at a boutique design studio called FancyPages. At FancyPages you create and host static sites for many clients, and they want their content to be served over HTTPS as fast as possible all over the globe. Furthermore you want to make this deployment as simple and repeatable as possible, to free up more time for design, and to eliminate errors and missed-steps when configuring manually.  In order to acheive this, FancyPages can create an ansible role to upload the static site to AWS S3 bucket, setup Fly to provision an SSL certificate, and create new a CNAME record to point to Fly at their DNS provider.  This can then be resused for all of FancyPages other clients - just put in a new bucket name and the correct domain name into the ansible variables file and run the playbook.

Let's get started on our walkthrough of deploying FancyPages static sites with Fly using Anisble.


## Local machine setup  

Firstly grab the Fly Ansible Module and example playbook from github. If you haven't used Ansible previously, you'll first need to install Python 3.5+ and virtualenv, then setup a python3 virtual environement and work from there to install ansible. The S3 anisble modules also require some additional packages:

`git clone https://github.com/andy-pi/fly-ansbile-module.git`  
`cd fly-ansible-module`  
`python3 -m venv env`  
`source env/bin/activate`  
`pip3 install ansible boto boto3 botocore`

## Variables

Next up create a file for the variables.  

`mkdir group_vars`  
`touch group_vars/all`  

and populate it with the following info from your Fly and AWS accounts. Note the AWS credentials must have permissions set to allow usage of S3:  

    fly_auth_key: "YOUR_FLY_API_TOKEN"
    site: "example-com"  
    hostname: "example.com"
    aws_access_key: "YOUR_AWS_ACCESS_KEY"
    aws_secret_key: "YOUR_AWS_SECRET_KEY"
    bucket_name: "example_static_site"
    local_path_html: "html/"
    

To get your API token, login to Fly.io, click Your Account, then Personal Access Tokens on the sidebar. Whilst you are there, you'll need to create your first site, hostname and choose the middleware for HTTPS, and for edge caching from the web interface. Fly's API is currently under heavy development and doesn't yet have a create site or add middleware feature - so for your the first time you'll need to do this using the web interface. After that you can use the same site and any middleware you've configured for multiple additional hostnames.  

You can find the site slug by opening the site in the fly dashboard - you'll see it in your browser address bar.  


## Example Playbook Workthrough

Let's setup an Ansible playbook `play.yml` using localhost for now, and we're going to perform a list of tasks:

    - hosts: localhost
      tasks:

### AWS S3

Firstly we'll create an S3 bucket:

      - name: create s3 bucket
        s3_bucket: 
          name: "{{ bucket_name }}"
          region: "{{ s3_region }}"
          aws_access_key: "{{ aws_access_key }}"
          aws_secret_key: "{{ aws_secret_key }}"
          state: present
        
Now let's sync our files to S3:
    
      - name: sync files to s3
        s3_sync:
          bucket: "{{ bucket_name }}"
          region: "{{ s3_region }}"
          aws_access_key: "{{ aws_access_key }}"
          aws_secret_key: "{{ aws_secret_key }}"
          file_root: "{{ local_path_html }}"
          file_change_strategy: force
          permission: public-read
        
The last part of our S3 setup is to configure the bucket:
    
      - name: add website configuration to bucket
        s3_website:
          name: "{{ bucket_name }}"
          region: "{{ s3_region }}"
          aws_access_key: "{{ aws_access_key }}"
          aws_secret_key: "{{ aws_secret_key }}"
          suffix: index.html
          error_key: errors.html
          state: present


### Fly.io

Now we've got our static site files on S3 we can move onto configuring Fly. Firstly we need to create new hostname. We can use the variables we've already setup.  The key output at this stage is `result.attributes.preview_hostname` thats the `xyz.shw.io` style preview hostname provided by Fly. Here we've just paused the ansible role and printed it for the user to update manually - but ideally you're DNS provider can also be accessed through an API (or even Ansible module), and you can automate this step too.

      - name: Create a fly hostname
        fly:
          command: create_hostname
          fly_auth_key: "{{ fly_auth_key }}"
          site: "{{ site }}"
          hostname: "{{ hostname }}"
        register: result
    
      - pause:
          prompt: "Set your DNS CNAME record for {{ hostname }} to point to {{ result.response.preview_hostname }}"
        
The variable `{{ result.response.attributes.preview_hostname }}` is what I use as the input for my DNS provider.
          
Next up we add a backend to Fly, in this case an S3 bucket (which we're assuming is already setup as a website), and we need to provide the name of the bucket and the region it is located in.  Other backends will have different settings, and all the details can be found in the Fly API documentation (https://fly.io/docs/api/#backends).  We also register the result of this operation as we'll need to use the backend id that fly provides to link it to the hostname we just created.  


      - name: Add a fly backend
        fly:
          command: add_backend
          fly_auth_key: "{{ fly_auth_key }}"
          site: "{{ site }}"
          backend_name: "my_s3_static_site"
          backend_type: "aws_s3"
          backend_settings: 
            bucket: "{{ bucket_name }}"
            region: "{{ s3_region }}"
        register: result
        
        
The final step is to add a rule. This links the hostname to the backend, and Fly allows us to specify both the incoming path (example-hostname.com/blog) and the path replacement, which would be equivalent to a folder inside an S3 bucket. In the example we're simply using the root of the hostname and the S3 bucket's index page.

      - name: Add a fly rule
        fly:
          command: add_rule
          fly_auth_key: "{{ fly_auth_key }}"
          site: "{{ site }}"
          hostname: "{{ hostname }}"
          backend_id: '{{ result.response.id }}'
          action_type: "rewrite"
          path: "/"
          priority: "10"
          path_replacement: "/index.html"
          
Now we've completed our ansible playbook its time to run it:

`ansible-playbook play.yml`

Ansible will execute the commands in our playbook deploying our site entirely automatically. The next time you need to deploy a different site, simply edit `group_vars/all`, pop in your next site's bucket, hostname and path to the html on the local machine and you are good to go!  

Once your DNS routes have gone live, you can point your browser at your hostname, and Fly will serve up your static site from the S3 backend with a Let's Encrypt SSL certificate. 

## Why Fly?

There are many advantages for FancyPages in this scenario:

Fly automatically provisions Let's Encrypt SSL certificates for your website, and setup is way easier than only using the AWS ecosystem, in which you'd have to go through a number of steps to create a certificate in AWS Certificate Manager, click approve on the email they send you; then to use the certificate you must configure Cloudfront. With Fly, this is all done for you automatically saving you precious time and money.

The next piece of good news is that Fly can serve any S3 bucket on your domain - the bucket name does not need to be exactly the same as the domain, which is a requirement for using custom domains with S3. For the benefit of FancyPages' clients, they need their sites to load at blistering speed in any location. This means that their content needs to be served as close as possible to their users.

Although FancyPages uploads all of their web site code to one server location, using Fly's edge-caching middleware, these static resources can be cached in multiple locations around the world, reducing the time it takes to serve the data to the browser.

In addition to all that, Fly can rewrite HTML, erm... on-the-fly! This is especially useful in helping ensure a fully secured website. It may be the case that the HTML has many links that were originally hard coded - for example CSS or JavaScript resources. If FancyPages does not carefully check all of these their customers will see warnings in their browser about some parts of the site being insecure - not good!

With Fly's HTML rewrite option you don't even need to go through all your existing HTML to change the source. Simply add a rewrite rule to change all http to https, problem solved for all pages on all your sites. The rewrite rule is only currently available from the web interface, but it is coming soon to the API (and thus the Ansible module!).
