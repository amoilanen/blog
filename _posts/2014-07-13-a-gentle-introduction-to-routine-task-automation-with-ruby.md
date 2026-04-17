---
layout: default
title: "A Gentle Introduction to Routine Task Automation with Ruby"
date: 2014-07-13
---

# Contents

[Learn You a Bit of Ruby](#ruby)
[Simple Routine](#routine)
[Automation](#automation)
[Alternatives to Ruby](#alternatives)
[More Examples of Automation](#examples)
[Studying Ruby Further](#studying)

<a name="ruby"></a>

## Learn You a Bit of Ruby

Even if you are not a software developer you absolutely have to sometime give [Ruby](http://www.ruby-lang.org/en/ "Ruby") a try. It is a nice language geared towards increasing programmer's productivity and is a lot like a Swiss Army Knife of the world of programming: in many situations you will find that Ruby is the right tool for achieving what you need.

I mainly use Ruby for rapid prototyping of new features, automation of routine administrative tasks and for test automation but Ruby is not limited to this. For example, a very popular area for Ruby is Web development, where the [Ruby-on-Rails](http://rubyonrails.org/ "Ruby-on-Rails") framework is widely used.

There are, of course, as it is the case with any language, areas for which Ruby and associated with it libraries and frameworks are not the best choice. Examples of this are performance-critical low level applications such as drivers or highly scalable powerful server-side architectures. We just have to keep in mind that not just Ruby but any tool is best used in a particular context and has its natural limitations and imperfections as well as advantages.

Learning Ruby is quite pleasant and surprisingly easy and you will quickly be able to achieve in 10 lines of code what seasoned Java developers cann't do in 100 lines. Chances are that Ruby will also give you a better understanding of programming languages and algorithms in general.

<a name="routine"></a>

## Simple Routine

My first experience with Ruby was with automating tasks that I had to routinely perform on my machine. Let's now consider one such simple task:

On my computer I have quite a few directories with photos but they are a bit unorganized. For each event (like "New Year Celebration") I have a directory with photos which in turn can contain subdirectories with photos that can contain other subdirectories with photos, etc. Mainly this is because I make pictures from different devices but also sometimes I select a few photos and put them in a separate subdirectory so that I can send them to a friend.

Now I would like to organize these photos in such a way that the directory structure is flattened: only the top level directories corresponding to events are left, all the photos from the nested subdirectories are copied to these top level directories and then the nested subdirectories themselves are removed.

So, how do we go about such a routine task? Of course, we can do all the actions manually and perform roughly the same routine procedure involving moving files and removing directories for 30 or so top level event directories. However, this will be extremely boring, tiresome, unproductive, error-prone and wrong. Such routines should not be left to humans, they should be automated and done by a machine. Let's use the Swiss Army Knife - Ruby to automate this.

<a name="automation"></a>

## Automation

Some familiarity with Ruby would not be excessive here, but it's OK if you don't understand everything in the code listings as I will try to explain the code as we go. However, if you are new to programming in general or to Ruby, feel free to follow the following nice 20 minutes [Ruby tutorial](http://www.ruby-lang.org/en/documentation/quickstart/ "Ruby tutorial"). At this point you can also try to install Ruby and familiarize yourself with how to create and run simple scripts if you want to follow along and play with the provided code a bit.

All begins fairly straightforward. We import the libraries that we will need for our simple script.

```ruby
require 'find'
require 'fileutils'
require 'pathname'
```

Then we write a procedure that will perform some action passed as a block for each first level subdirectory in a given directory:

```ruby
def for_each_child_directory(root_dir)
  return if !block_given?
  old_dir = Dir.pwd
  Dir.chdir root_dir
  Dir.glob("*") do |file_path|
    if File.directory?(file_path)
      yield file_path
    end
  end
  Dir.chdir old_dir
end
```

This procedure can now be called as follows:

```ruby
  for_each_child_directory("/home/anton") do |dir|
    puts "Found first level directory #{dir}"
  end
```

where

```ruby
  do |dir|
    puts "Found first level directory #{dir}"
  end
```

is a block of code that is passed to the method **for_each_child_directory** in addition to the explicit parameter **root_dir**. You can think of this block as a special kind of hidden parameter which contains something that can be executed and to which we can pass parameters of its own with the following construct:

```ruby
  yield file_path
```

Now, let's implement other procedures.

```ruby
def move_all_new_files_from_subdirectories_to(dir)
  Find.find(dir) do |file_path|
    already_exists = File.exist?(File.join(dir, Pathname.new(file_path).basename))
    if !File.directory?(file_path) && !already_exists
      FileUtils.mv(file_path, dir)
    end
  end
end
```

This procedure moves all the files from the subdirectories to the root directory which is specified as the parameter **dir**. For each file in a subdirectory we move it to the root directory only if it is not a directory and it does not already exist there.

Once the files have been moved to the root we can remove the empty subdirectories of the root directory **dir**:

```ruby
def remove_all_subdirectories_in(dir)
  for_each_child_directory(dir) do |dir|
    FileUtils.rm_rf dir
  end
end
```

And now we bind the procedures that we defined so far together and make sure that we can call the script from the console:

```ruby
def flatten_directory(dir)
  move_all_new_files_from_subdirectories_to(dir)
  remove_all_subdirectories_in(dir)
end

def flatten_child_directories(root_dir)
  for_each_child_directory(root_dir) do |dir|
    flatten_directory dir
  end
end

flatten_child_directories(ARGV[0] || ".")
```

**ARGV[0]** is the first argument that we pass to the script from the console when we call it like this:

**ruby flatten_directories.rb "/home/anton/photos"**

The whole script can be found [here](https://github.com/antivanov/misc/blob/master/Ruby/flatten_directories.rb)

<a name="alternatives"></a>

## Alternatives to Ruby

There are, of course, a number of alternatives for automating the task above or similar tasks. For example, we could have used Bash for Linux or PowerShell for Windows. The advantage of such tools is that they are native to the platform for which you implement automation. However, their huge disadvantage is poor scalability (it is hard to write large scripts) and maintainability. Bash has a very terse and cryptic syntax that is hard to learn and use unless you write Bash scripts every day. Another thing is that what you learn about Bash is specific for Bash and you cannot really reuse it anywhere else outside of Bash scripting. So you invest quite a lot of time into learning a programming language that has a very limited usage: administrative scripting on a specific platform. Often the Bash scripts written some time ago are hard to read and understand, especially this is true for large scripts (>= 100 lines of code). My advice would be to use Bash only for really simple scripts and do not allow your scripts grow to more than 10-15 lines. Otherwise Ruby (often with some Bash or PowerShell commands mixed into the script) will do a much better job.

Let's list the main advantages of using Ruby to automate routine tasks:

- Availability of many good libraries for easy networking and file system access

  The used 'fileutils' library provides a good example of this.

- Minimum of dependency on the underlying platform

  The written script is cross-platform, it can be executed both on Windows and Linux. In most scripts, however, there may be calls specific for the underlying operating system, for example:

  ```ruby
  def download(file_URL, file_name)
    system("wget -c \"#{file_URL}\" -O \"#{file_name}\"")
  end
  ```

  Here we are using the [wget](http://www.gnu.org/software/wget/ "wget") utility to download a file. Unless there is a **wget** available the script will not run. So the cross-platformness has its limits.

- Seamless integration with the native commands

  It is possible to resort to Bash in a Ruby script by using the methods **system** or **IO.popen**. However, it is much nicer to glue the low level commands with Ruby rather than with pure Bash.

- Ruby allows to write procedural style simple scripts, but you can also introduce objects and additional structure to your scripts as you go.

  The script above is written in the procedural style in which Ruby allows you to program, but should you find yourself in a need to write a larger script you can easily use objects and modules as in any Ruby program and this will give you more structure and make it easier to maintain and understand the script.

<a name="examples"></a>

## More Examples of Automation

Here are a few other examples of automation:

- [CD grabber](https://github.com/antivanov/misc/blob/master/Ruby/cd_grabber.rb "CD grabber")

  Grabs CD tracks into MP3 with an option to fetch information about tracks from [Amazon.com](http://www.amazon.com "Amazon.com") or from the local file system.

- [Apartment watcher](https://github.com/antivanov/misc/blob/master/Ruby/apartment_watcher.rb "Apartment watcher")

  Checks the availability of apartments and notifies via e-mail when a new apartment with the specified search options becomes available.

- [Podcast client](https://github.com/antivanov/rpodder "Podcast client")

  Downloads podcasts from a given URL to the local machine.

<a name="studying"></a>

## Studying Ruby Further

You will not need to know advanced Ruby to write simple scripts like the one in the present article and [Everyday Scripting with Ruby: for Teams, Testers, and You](http://pragprog.com/book/bmsft/everyday-scripting-with-ruby "Everyday Scripting with Ruby: for Teams, Testers, and You") will provide a nice start if you are not a programmer.

But if you are a novice Ruby programmer and would like to study Ruby more like a programming language from a perspective of a developer you can read [Beginning Ruby](http://beginningruby.org/ "Beginning Ruby").

Ruby is a generic programming language well-suited for a number of tasks. One of them is automation. So, next time when you are faced with some routine administrative task on your machine why don't you learn a bit of Ruby and give it a try to help you automate it?

## Links

[Ruby language](http://www.ruby-lang.org/en/ "Ruby language")
[Ruby in 20 minutes (tutorial)](http://www.ruby-lang.org/en/documentation/quickstart/ "Ruby in 20 minutes")
[Everyday Scripting with Ruby: for Teams, Testers, and You (book)](http://pragprog.com/book/bmsft/everyday-scripting-with-ruby "Everyday Scripting with Ruby: for Teams, Testers, and You")
[Beginning Ruby (book)](http://beginningruby.org/ "Beginning Ruby")
[Pro Bash Programming: Scripting the Linux Shell (book)](http://www.barnesandnoble.com/w/pro-bash-programming-chris-johnson/1102592577 "Pro Bash Programming: Scripting the Linux Shell")
[Windows PowerShell](http://powershell.org/ "Windows PowerShell")
[Ruby-on-Rails](http://rubyonrails.org/ "Ruby-on-Rails")
