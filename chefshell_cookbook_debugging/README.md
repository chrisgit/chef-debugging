Chef Cookbook Debugging with Chef-Shell
----------------------------------------
Chef-Shell is part of the Chef eco system and can be used to debug cookbooks (link: https://docs.chef.io/chef_shell.html, https://docs.chef.io/ctl_chef_shell.html and https://docs.chef.io/debug.html).

Chef-Shell can be used in conjunction with test kitchen when Chef-Shell is invoked in it's 'Zero' mode.

As with pry debugging it is easier to let Test Kitchen perform the heavy lifting task of installing Chef and copying to cookbooks to the Virtual Machine.
* Create a cookbook and add test kitchen (kitchen.yml etc), add a test suite and leave the run list empty
* If you already have a cookbook with a test suite then add a new suite in .kitchen.local.yml and leave the run list empty or clear out the run list of an existing suite
* Perform a converge with test kitchen specifying your test suite with the empty run list
* Log in to the VM

For Windows 
* Open a command window and type chef-zero; the chef-zero server will run
* Open another command window and navigate to C:\tmp\kitchen\cookbook
* Later versions of Chef the cookbooks will be stored in %USERPROFILE%\AppData\Local\temp\kitchen

For Linux
* Type chef-zero -d (located in /opt/chef/bin so full command maybe /opt/chef/bin/chef-zero -d, you can start this in background with am ampersand &)
* NB: You may need to install chef-zero, you can do that with gem install chef-zero
* Navigate to /tmp/kitchen/cookbooks

At this point there is a Chef-Zero server running on your VM ready for you to upload your nodes, cookbooks and environments but before we can upload anything to the Chef-Zero server we need to put in a breakpoint to our recipe.

The easiest way to add a breakpoint to your recipe using the breakpoint resource. The breakpoint resource is ignored on normal Chef-Client runs and only used when the Cookbook is run with Chef-Shell, therefore you can safely add a breakpoint to your Cookbook source. example of the breakpoint resource is below

````
...
directory 'C:\\Configuration' do
    action :create
end
 
# Will be used by Chef-Shell only
breakpoint "Before Create Config" do
  action :break
end
 
# Fixed resource ... we missed off the Path attribute value, thanks Pry
template 'Create Config' do
    source "config_template.erb"
    path "C:\\Configuration\\Test.cfg"
    variables ({
        :min_memory => node['Config']['minimum_memory'],
        :max_memory => node['Config']['maximum_memory'],
        :secure => node['Config']['use_ssl']
    })
    action :create
end
````

After saving our recipe with the breakpoint we can upload the files to the Chef-Zero server. 

Change folder to the location of test kitchen files
* Windows: %USERPROFILE%\AppData\Local\Temp\kitchen
* Linux: /tmp/kitchen

* Upload ALL files in your tmp/kitchen folder using knife upload . -c client.rb (knife upload was previously knife-essentials), NB: This will only upload JSON format files

Alternatively you can load individual folders, the commands below will work with JSON files or the older Ruby style files
* Cookbooks: knife cookbook upload -a -c client.rb
* Databags: knife data bag from file data_bags -a -c client.rb
* Environment: knife environment from file environments -a -c client.rb
* Roles: knife role from file roles/*.rb -c client.rb (or might be *.json)
* Nodes: knife node from file nodes/*.json -c client.rb (you have to do these individually)

When the files have uploaded to your Chef-Zero server start your chef-shell session with chef-shell -z -c client.rb -o "run_list" (for Linux you may have to sudo -su first)

NB: You can edit/modify your cookbooks and re-run knife upload . -c client.rb and the updated cookbook will upload to the Chef zero server, this means the cookbooks aren't frozen and means there is no need to stop the Chef zero server, start a new instance and then upload the cookbooks/nodes/environments.

Chef-Shell will load all of your cookbooks specified by the run-list, when the Chef-Shell prompt appears the Compile phase has already happened which means that you can checked the values of attributes and ensure resources have been loaded, alternatively peform a Chef-Run and start debugging!

To start a Chef run in Chef-Shell simply type “run_chef”, the converge will run until the first breakpoint is hit.

As with pry (or any other Ruby debugger) you now have access to all of the Chef run time classes.

In production an attributes' value might come from a Chef Environment, Chef Role or set by a search result in a recipe; with the node.debug_value we can tell exactly how the value has been set, it's a nice debugging command.

After your breakpoint has been hit, you've investigated and rectified the problem you can complete your converge within Chef-Shell by typing “chef_run.resume”.

Shef?
-----
Originally Chef-Shell was called Shef, there was a really good article by Steve Danna about how to use Shef (link: http://stevendanna.github.io/blog/2012/01/28/shef-debugging-tips-1/).

Steve also wrote a set of Ruby helpers that can be used in conjunction with Shef (now Chef-Shell), the most valuable and still valid today is his Shef/Extras.rb file found here: https://github.com/stevendanna/knife-hacks/tree/master/shef). 

To use Shef/Extras.rb copy the file to your VM, when in Chef-Shell type require 'path to extras.rb file' followed by ShefExtras.load, when you've done that you can use Steve's ordered_resources methods and insert_break; insert_break gives you the capability of setting a breakpoint WITHOUT having to modify your Chef cookbook to add a breakpoint resource.

