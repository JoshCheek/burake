Give me a Burake
================

_Making it less obnoxious to use Bundler's rake and other binstubs_

UPDATE:
-------

If you're using RVM, then `rvm get head`, and this script will be unnecessary. It does the same thing 
(hijacks rake and then redirects it to ./bin/rake) but works for several common commands, not just rake. 
And it does it by looking for a Gemfile, so no need to maintain a list of projects. 
[Check it out](https://rvm.beginrescueend.com/integration/bundler/).



What does it do?
----------------

So [Bundler](http://gembundler.com/) is awesome, but to do it right you have to constantly type
`bundle exec rake whatever`, which blows. You can make it a bit better by `bundle install --binstubs`
which will give you a bin directory with your executables in it. So now you can run `bin/rake whatever`
which is better. But, lets be honest, who forgets and keeps typing `rake whatever`? And who feels just a
little bit taxed each time they have to run a rake command, knowing that if it's not the last command or
two they ran, then they're going to have to type out the full `bin/rake`? Yeah, me too. I know it sounds
silly, but that's how it is.

I thought about a lot of possible solutions, and finally decided this had the best balance between security,
easiness, and just doing what you want without having to think about it.


How does it work?
-----------------

You put your own rake in the path so it gets discovered before all the other rakes. This rake checks a
manifest of Bundler projects, and if you're currently in one, it runs _that_ rake, otherwise, it runs
system rake (or rvm's rake, if you're using rvm).


What about non-rake binaries?
-----------------------------

Make them into rake tasks that invoke the correct binary. (note that rake resets your path to the dir
of the Rakefile, so regardless of where you invoke rake from, you can invoke the binaries relatively
from the dir of the Rakefile)

A couple examples:

    desc 'compare app to specs'
    task :spec do
      sh  'bin/rspec '              +
          '--color '                +
          '--format=documentation ' +
          'spec/**_spec.rb'
    end

    desc 'run the server on port 9394'
    task :server do
      sh 'bin/shotgun config.ru -p 9394'
    end



Okay, I want it, what do I do?
------------------------------

###Add a dir for binaries

Make a dir at ~/bin (or whatever you want). Then stick `export PATH="~/bin:$PATH"` at the end of your
~/.profile This will cause your shell to look in ~/bin for binaries, allowing you to write your own,
or to hijack them like we're doing here. (NOTE: make sure you stick it after any rvm stuff, or rvm
will re-hijack rake from you)


###Create the new rake

Take the code below and put it into ~/bin/rake which will get loaded before other rakes. Then
`chmod +x ~/bin/rake` to make sure it is executable. 


    #!/usr/bin/env sh

    manifest="$HOME/.bundler_projects"
    current_dir=`pwd`


    # exit if missing the manifest
    if [ ! -r $manifest ]; then
      echo "You need to make a '$manifest' text file which lists the root directories of projects you want to use." 1>&2
      exit 1
    fi


    # get project dir from manifest that correlates to CWD
    # I tried to do this in bash, but I'm just not good enough to figure out how
    project_root="$(ruby -e "
      possibilities  =  File.readlines '$manifest'
      possibilities  =  possibilities.map     { |dir| File.expand_path dir.chomp }
      matches        =  possibilities.select  { |dir| '$current_dir'.start_with? dir }
      best_match     =  matches.sort_by       { |dir| dir.length }.last
      print best_match
    ")"
    

    # if no project_root, or no binary, use real rake
    if [ -z "$project_root" ] || [ ! -x "$project_root/bin/rake" ]; then
      rake_binary=`which rake`
    else
      rake_binary="$project_root/bin/rake"
    fi
  

    # run the appropriate rake, forward the args
    $rake_binary "$@"



###Test that it works

Clone this repo and run `./burake_spec` to make sure it works. (if you don't have rspec: `gem install rspec`)

If your tests passed, then it should work for you. If not, fork this, fix it, send me a pull request.


###Create your manifest

Now you just need to add your projects to the manifest, which is the file ~/.bundler_projects I do this by `cd`ing
into the project root and typing `pwd >> ~/.bundler_projects` basically, each line is just a path to a project with
that has an executable bin/rake. This is basically for security, we could just look for a bin/rake in any ancestor
dirs, but there is risk associated with that. This should prevent you from running a rake that you didn't explicitly
decide to.


###Make sure you're using binstubs

This whole thing assumes you have your rake in your project's bin dir. Bundler will create and populate this for
you when you run `bundle install --binstubs`. You should always be using these binaries for this project, or you
run the risk of sidestepping the Bundler sandbox and messing up your dependencies.


Gratz, you're good to go.
