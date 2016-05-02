Chef Cookbook Debugging with Test Kitchen and Pry
-------------------------------------------------
This test assumes you know how to converge using Test Kitchen, the information will work when any of the Test Kitchen drivers (Vagrant, EC2 etc).

The basic technique is to let Test Kitchen perform the heavy lifting of

* Creating a VM
* Installing a provisioner on the VM
* Resolving cookbooks using Berkshelf or Librarian
* Sending across a tarball containing the cookbooks
* Setting up the VM to run the provisioner

The basic process is below

* On your laptop create a cookbook, add test kitchen (with kitchen-init), add a test suite and leave the run list empty
* Optionally if you already have a cookbook using test kitchen then add a .kitchen.local.yml file to add a new test suite with an empty run list
* Perform a converge with test kitchen specifying your test suite with the empty run list
* Log in to the VM

Because you did not specify a run list the Chef-Client has nothing to do, you must login to the VM and perform the converge using chef-client manually.

First, login to the VM, then

For Windows 
* Open a command window and change folder to C:\tmp\kitchen\cookbooks, edit the recipe to add your pry commands
* Later versions of test kitchen (1.4 and above) the cookbooks may be in C:\Users\vagrant\AppData\Local\Temp\kitchen
* Change folder to C:\tmp\kitchen

For Linux 
* Change folder to /tmp/kitchen/cookbooks and edit the recipe to add your pry commands
* Change folder to /tmp/kitchen

In the command window/prompt type chef-client -z -j dna.json -c client.rb -r "run_list"

Chef-Client will upload cookbooks, nodes, environments to an in-memory Chef Server, load your cookbooks into the Chef-Client and start to execute your cookbook code; when the pry prompt is invoked you can do everything you can when running with full Chef-Client and Chef-Server!

Let's just expand that
----------------------

We are using test kitchen to create the VM, install Chef client and send across cookbooks.

When the cookbooks have been copied across test kitchen then invokes the Chef-Client using the following commands

On Windows
* $env:systemdrive\opscode\chef/bin\chef-client.bat --local-mode --config /tmp/kitchen/client.rb --log_level auto --force-formatter --no-color  --chef-zero-port 8889 --json-attributes /tmp/kitchen/dna.json 

On Linux: 
* sudo -E /opt/chef/bin/chef-client --local-mode --config /tmp/kitchen/client.rb --log_level debug --force-formatter --no-color  --chef-zero-port 8889 --json-attributes /tmp/kitchen/dna.json

Chef-client --local-mode is equivalent to Chef-Client -z, this is a really cool new feature of Chef-Client that will create an in memory Chef server so that your cookbooks with Chef searches work (previously using Chef-Solo this was not possible).

Update: Here is a cool article using Chef Zero to get fast feedback on a cookbook  https://www.chef.io/blog/2013/10/31/chef-client-z-from-zero-to-chef-in-8-5-seconds/

Pry
---
Pry is a great Ruby debugging tool, to use pry for debugging cookbooks you have to load it into memory first by adding

````
require 'pry'
````

to your cookbook. After you have loaded pry you can add breakpoints by typing 
````
binding.pry
````

Example is below

````
require 'pry' # Load pry into memory

x = 2
y = 4
z = x + y 
binding.pry # Breakpoint here, we want to interrogate z before doing any other actions, to do that simply type z in the pry prompt!

````

The best thing to do is add pry to your cookbooks when they are deployed onto your local VM, if the pry statements are added to your cookbooks on your local SCM repo you might accidentall check the code in, this would not be good as it will cause your cookbooks to "hang" waiting on the breakpoint.

Pry can be used to add breakpoints in the compile or converge phase of a Chef run, you can put the pry breakpoint into any part of the Chef code.

````
# Pry in the compile phase
...
log 'Chef End' do
    message "Before message create"
    level :info
end
 
 
require 'pry'
binding.pry
````

The above will invoke a pry prompt during the Compile phase (basically when Chef loads a file and Ruby interprets and executes it), alternatively you can add a pry binding inside a ruby_block resource, this time pry will be invoked in the Converge phase (when resources are popped from the resource queue and executed)

````
# Pry in the converge phase
...
ruby_block 'Load Pry Session' do
    block do
        require 'pry'
        binding.pry
    end
end
````

The above will invoke a pry prompt during the converge phase (when the ruby_block resource is actioned).

When the Pry debugger breakpoint is reached you can query ANYTHING about your Chef cookbook environment!

Pry has several plugins that extend the base behavior two I recommend are
* Ruby 1.9.3 and below, pry-debugger
* Ruby 2.0.0 and above, pry-byebug

Helpful Debugging Commands
--------------------------

Inspecting Ruby objects
-----------------------
**P**

A favourite of mine is to output the underlying object values using inspect, typically this is <object>.inspect.to_s
However, there is a shortcut to inspect and object and output to stdout; the single letter command is p, typically used this way

p <object>

**JSON.pretty_generate**

Chef uses a lot of complex structures and hashes, the output of inspecting an object can be quite hard to read, sometimes it is easier to read the structure as JSON.
First include the JSON library (this is using the require method), then output your structure using  JSON.pretty_generate(hash); alternatively you can also use to_json.

**pp**

Pretty print is similar to JSON.pretty_generate and will format complex structures, pp is available in Chef-Shell, in IRB or Pry first require 'pp', e.g.
require 'pp'
pp <object to view>

methods, instance_methods, respond_to?
--------------------------------------
Sometimes it�s difficult to know exactly what methods a class contains. To test to see if a single method exists and can be called you can use  the respond_to? method (in Ruby a method name ending with a question mark should return true or false, a method name ending with an exclamation mark means warning or use with care as it normally changes the value or structure of the data in the object).

````
class TestClass
  def method1
  end
  def method2
  end
  def method3
  end
end
 
puts "*****----- TestClass.methods #{TestClass.methods}"
puts "\n*****----- TestClass.intance_methods #{TestClass.instance_methods}"
````

**Self, Self.Class**

The Self and Self.Class methods might not seem that handy but Chef has a complex object hierarchy and it's sometimes easier to know what class you are dealing with.

**local_variables, global_variables**

The local_variables command will display all variables within the current scope.
The global_variables command will display all variables within a global scope (you'll notice that global variables start with a dollar symbol, you can also declare a global variable by pre-fixing your variable names with a dollar symbol, i.e. $MYGLOBAL = "Hello World".
Ruby does place a few handy variables in global scope such as $LOAD_PATH.

**Symbol**

To view all Ruby symbols type Symbol.all_symbols or Symbol.all_symbols.inspect

Chef Environment
----------------
A lot of the environment information is gathered for Chef by a utility called Ohai, you can call Ohai directly on the command line; you can extract individual attributes with a command line parameter or optionally look for values with grep or find /i.

<images: ohai-example>

**Debugging Chef attributes**
Chef attributes are a complex topic in their own right; there are many different orders of precedent rules to apply a value to an attribute; in order to determine how the attribute has obtained its value you can use the debug_value method.

````
puts node['a']['b']['c'].debug_value
````

````
# Type self.class to see if the context of object you are in is a node
# If you are not in a node object then try self.respond_to?(:node)
# Objects such as Chef::Recipe contains node and run_context; if you are really stuck download the Chef source code!
  
# Use node.recipes directory or assign it to a new variable, don't forget to call the Ruby methods method or use respond_to? to get a list of supported methods
node_recipes = node.recipes
nodes_recipes.include?(recipe_name)
  
# Other useful methods from node
node.role?(role_name)
node.recipe_list
node.run_state
node.run_context
node.chef_environment
node.attributes
node.recipe?(recipe_name)
node.run_list ...
  
# run_context is available from the node object and contains information about the runtime environment
run_context.cookbook_collection
run_context.definitions
run_context.resource_collection
run_context.immediate_notification_collection
run_context.delayed_notification_collection
run_context.events
run_context.loaded_recipe?
run_context.has_template_in_cookbook?
run_context.has_cookbook_file_in_cookbook ...
````